---
icon: plug-circle-plus
---

# Live tuning

Live tuning lets you change gains, motion profiles and limits from a dashboard while the robot is enabled, and see the result straight away. It works over NetworkTables with AdvantageScope or Elastic.

<figure><img src="../../.gitbook/assets/live-tuning-loop.png" alt="A loop: the dashboard sends tunable values under NT:/Tuning to the Live Tuning command, which applies them to the SmartMotorController every loop, and telemetry under NT:/Mechanisms flows back to the dashboard"><figcaption><p>Change a number, watch the graph, change it again.</p></figcaption></figure>

## Turn it on

Give the motor's telemetry `TelemetryVerbosity.HIGH`. That is all most robots need.

<pre class="language-java"><code class="lang-java">SmartMotorControllerConfig lowerFlyWheelConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withZeroPower(MotorMode.COAST)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
      .withMomentOfInertia(Inches.of(4), Pounds.of(2))
      .withClosedLoopController(1,
                                0,
                                0) // You generally do not want a profile because its not a position controlled loop.
      .withFeedforward(new SimpleMotorFeedforward(0, 0, 0)) // Helps track changing RPM goals
      .withMotorInverted(false)
<strong>      .withTelemetry("LowerFlyWheel", TelemetryVerbosity.HIGH) // yams.core.telemetry.enums.TelemetryVerbosity
</strong></code></pre>

| How the motor's telemetry is set up | Live Tuning command |
| --- | --- |
| `withTelemetry(name, TelemetryVerbosity.HIGH)` | Registered for you |
| `withTelemetry(name, telemetryConfig)` with a `SmartMotorControllerTelemetryConfig` | Registered for you |
| You call `setupTelemetry(NetworkTable, NetworkTable)` yourself (Java) | Call `setupLiveTuning()` afterwards, see below |
| C++ with `TelemetryVerbosity::HIGH` and a subsystem on the config | Registered for you when `SetupTelemetry()` runs |

YAMS registers **one** `Live Tuning` command per subsystem (Commands v2) or mechanism (Commands v3). Every motor controller in that subsystem or mechanism joins the same command.

<details>

<summary>Setting up live tuning explicitly</summary>

If you set up telemetry yourself, the `Live Tuning` command is not registered for you. Call `setupLiveTuning()` on the config after the telemetry is set up:

```java
SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
    .withTelemetry("ArmMotor", TelemetryVerbosity.HIGH);
SmartMotorController motor = new TalonFXWrapper(armMotor, DCMotor.getKrakenX60(1), motorConfig);

motor.setupTelemetry(telemetryTable, tuningTable);
motorConfig.setupLiveTuning();
```

`setupLiveTuning()` does nothing when:

* the `SmartMotorController` has not set up its telemetry yet (that is what attaches the controller to the config), or
* the config has no subsystem (Commands v2) or mechanism (Commands v3).

Registering two different subsystem or mechanism instances with the same name throws an `IllegalStateException`. Give each one a unique name, for example `LeftTurret` and `RightTurret`.

In C++ there is no explicit call.

</details>

{% hint style="info" %}
Using CTRE TalonFX or TalonFXS motors? Phoenix Tuner X can connect to your simulation too. See [TunerX in simulation](tunerx-in-simulation.md).
{% endhint %}

## Use it

{% hint style="danger" %}
Live tuning can be **dangerous**. Test it in simulation before using it on the robot, and understand how it behaves first.
{% endhint %}

{% stepper %}
{% step %}
#### Try it in simulation

Run the robot code in simulation and go through the steps below there first.
{% endstep %}

{% step %}
#### Enable the robot

Dashboard edits are only applied while the robot is enabled and the `Live Tuning` command is running.
{% endstep %}

{% step %}
#### Start the `Live Tuning` command

It is published at `NT:/Tuning/SUBSYSTEM_NAME/Live Tuning`, where `SUBSYSTEM_NAME` is the name of the subsystem (Commands v2) or mechanism (Commands v3) given to the `SmartMotorControllerConfig`.

{% tabs %}
{% tab title="Commands v2" %}
In Elastic, drag the `Live Tuning` command from `NT:/Tuning/SUBSYSTEM_NAME/Live Tuning` onto your dashboard and press its button.

It is a regular WPILib Commands v2 `Command` that requires the subsystem.
{% endtab %}

{% tab title="Commands v3" %}
Commands v3 commands are not dashboard widgets themselves, so YAMS publishes the command through `yams.commands3.telemetry.CommandTunable`. It shows up with a single `running` boolean at `NT:/Tuning/MECHANISM_NAME/Live Tuning/running`:

* Set `running` to `true` to schedule the command on the default `Scheduler`.
* Set `running` to `false` to cancel it.
* `running` reads back whether the command is currently running.

The command requires the mechanism and runs a `while (true)` loop that applies the dashboard values and calls `coroutine.yield()` each loop.
{% endtab %}
{% endtabs %}
{% endstep %}

{% step %}
#### Change a value

The tunable values are published under `NT:/Tuning/`. Edit one, and it is applied on the next loop.
{% endstep %}

{% step %}
#### Watch what happens

Graph the matching read-only fields under `NT:/Mechanisms/`. [What to tune and what to watch](#what-to-tune-and-what-to-watch) lists which field to watch for each value.
{% endstep %}
{% endstepper %}

{% hint style="warning" %}
The `Live Tuning` command requires the subsystem or mechanism, so the controller, a default command, or any other trigger for that subsystem or mechanism can interrupt it. Dashboard edits stop being applied when it is interrupted.
{% endhint %}

<figure><img src="../../.gitbook/assets/new_liveutinging.gif" alt="Starting the Live Tuning command and editing a tunable value on the dashboard"><figcaption><p>Starting the command and changing a value.</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/AScopeLiveTuningExample.gif" alt="Tuning a mechanism in AdvantageScope while the robot is enabled"><figcaption><p>Tuning while the robot is enabled.</p></figcaption></figure>

## Units: what you type and what you read

Tunable fields and read-only fields do not use the same units.

<figure><img src="../../.gitbook/assets/live-tuning-units.png" alt="For a flywheel you type 3000 RPM and read 50 rotations per second. For an arm you type 30 degrees and read 0.0833 rotations."><figcaption><p>The value you type is in the tunable unit, not the unit of the read-only field next to it.</p></figcaption></figure>

| Quantity | You type (tunable fields) | You read (read-only fields) |
| --- | --- | --- |
| Angular position | **Degrees** | **Rotations** |
| Angular velocity | **RPM** | **Rotations per second** |
| Angular acceleration | **RPM per second** | **Rotations per second²** |
| Linear position, velocity, acceleration (linear closed loop control) | **Meters**, **m/s**, **m/s²** | **Meters**, **m/s**, **m/s²** |
| Voltage, current | **Volts**, **Amps** | **Volts**, **Amps** |
| Temperature | Not tunable | **Fahrenheit** |

### See both in the same units with AdvantageScope

AdvantageScope can convert any read-only field for display, without changing the data or your robot code.

1. Drag the telemetry field, for example `NT:/Mechanisms/Arm/ArmMotor/MechanismPosition`, into a Line Graph.
2. Right-click the field in the graph legend and choose **Convert Units…**
3. Pick the target unit: Rotations to Degrees, rot/s to RPM, Meters to Inches, and so on.

Now the graph shows degrees and RPM, the same units the tunable fields expect. When the graph shows 180°, you type `180`.

{% hint style="success" %}
**Recommended setup for a tuning session**

* Convert `MechanismPosition` and `SetpointPosition` to degrees (or inches).
* Convert `MechanismVelocity` to RPM (or deg/s, in/s).
* Graph `SetpointPosition` and `MechanismPosition` together. YAMS has no closed loop error field: the error is the gap between the two lines.
* Keep the graphs visible next to the tunable fields.
{% endhint %}

## What to tune and what to watch

Every tunable field has a read-only field that tells you whether your change worked.

<figure><img src="../../.gitbook/assets/live-tuning-response.png" alt="Three step responses: kP too low rises slowly and stops short, about right settles quickly on the setpoint, kP too high overshoots and rings"><figcaption><p>What the position graph looks like after a setpoint change.</p></figcaption></figure>

{% tabs %}
{% tab title="PID" %}
| Tunable field | Watch | Why |
| --- | --- | --- |
| `kP` | `MechanismPosition` vs `SetpointPosition` | Error should shrink faster as kP rises. Oscillation means kP is too high. |
| `kI` | `MechanismPosition` vs `SetpointPosition` over time | Removes a steady error that never closes. Watch for windup causing overshoot. |
| `kD` | `MechanismVelocity`, `OutputVoltage` | Dampens oscillation. If voltage spikes on direction changes, kD is too high. |

Gains are in volts per rotation of error (per meter for linear mechanisms). See [Gain units](#gain-units) below.
{% endtab %}

{% tab title="Feedforward" %}
| Tunable field | Watch | Why |
| --- | --- | --- |
| `kS` | `OutputVoltage` at rest | Voltage to overcome static friction. Raise until the mechanism just starts to move. |
| `kV` | `MeasurementVelocity` vs `SetpointVelocity`, `OutputVoltage` | With kP = 0, adjust until measured velocity matches the setpoint at steady state. |
| `kA` | `MeasurementVelocity` during the acceleration ramp | Improves tracking while a motion profile accelerates. |
| `kG` (arms, elevators) | `MechanismPosition`, `OutputVoltage` | Arms: hold level and raise until the arm stays put. Elevators: a constant that fights gravity. |

{% hint style="info" %}
For arms, `kG * cos(angle)` is applied: full kG when level (0°), none when vertical (±90°). YAMS handles the cosine for you.
{% endhint %}
{% endtab %}

{% tab title="Motion profile" %}
| Tunable field | You type | Watch | Why |
| --- | --- | --- | --- |
| `TrapezoidalProfileMaxVelocity` | RPM (or m/s) | `MechanismVelocity` while cruising | The plateau speed. Raise to move faster; lower if the motor stalls while cruising. |
| `TrapezoidalProfileMaxAcceleration` | RPM/s (or m/s²) | `StatorCurrent`, `MechanismVelocity` ramp | How quickly speed builds. High acceleration means a current spike at the start. |
| `TrapezoidalProfileMaxJerk` | RPM/s² | Shape of `MechanismVelocity` | Smooths the ramp when sudden acceleration causes mechanical shock. |
| `ExponentialProfileKV`, `ExponentialProfileKA` | Same units as kV, kA | `MechanismVelocity` | The system constants the exponential profile plans with. |
| `ExponentialProfileMaxInput` | Volts | `OutputVoltage` | The most voltage the exponential profile will ask for. |
{% endtab %}

{% tab title="Setpoints" %}
| Tunable field | You type | Watch | Why |
| --- | --- | --- | --- |
| `TunableSetpointPosition` | Degrees (or m) | `MechanismPosition` vs `SetpointPosition` | Send the mechanism to a position without a button binding. |
| `TunableSetpointVelocity` | RPM (or m/s) | `MeasurementVelocity`, `OutputVoltage` | Command a speed, for example to find kV without a full profile. |
| `TunableClosedLoopControllerSlot` | Slot number | `ActiveClosedLoopControllerSlot` | Switch which gain slot the controller uses. |
{% endtab %}

{% tab title="Limits" %}
| Tunable field | You type | Watch | Why |
| --- | --- | --- | --- |
| `MechanismLowerLimit` | Degrees | `MechanismPosition`, `MechanismLowerLimit` (boolean) | The boolean turns true at the lower soft limit. Move the limit to change where it stops. |
| `MechanismUpperLimit` | Degrees | `MechanismPosition`, `MechanismUpperLimit` (boolean) | The same for the upper soft limit. |
| `MeasurementLowerLimit`, `MeasurementUpperLimit` | Meters | `MeasurementPosition` | Soft limits for linear mechanisms. |
| `StatorCurrentLimit` | Amps | `StatorCurrent`, `MechanismVelocity` | Caps torque. Watch for speed sagging when the limit is too low. |
| `SupplyCurrentLimit` | Amps | `SupplyCurrent`, `OutputVoltage` | Prevents brownouts. Lower it if supply current spikes dip the voltage. |
{% endtab %}

{% tab title="Ramp rates" %}
| Tunable field | You type | Watch | Why |
| --- | --- | --- | --- |
| `OpenloopRampRate` | Seconds, 0 to 100% | `OutputVoltage` during open loop commands | Slows voltage changes in open loop. Reduces shock on `set(dutyCycle)`. |
| `ClosedloopRampRate` | Seconds, 0 to 100% | `OutputVoltage` during closed loop moves | The same for closed loop, when a setpoint step causes a big voltage jump. |
{% endtab %}
{% endtabs %}

{% hint style="info" %}
**Worth watching the whole time**

* `MotorTemperature`: a hot motor backing off can look like a gain problem.
* `OutputVoltage`: the raw command the motor is getting. Confirms your feedforward math.
* `RotorPosition`, `RotorVelocity`: values before gearing. Useful for checking the gearing is right.
{% endhint %}

Some settings are **not** tunable live and have to be changed in the config: the closed loop tolerance, voltage compensation, the closed loop maximum voltage and the temperature cutoff.

### Gain units

<details>

<summary>PID gains</summary>

| Gain | Unit | Meaning |
| --- | --- | --- |
| `kP` | V/rot | Volts per rotation of position error. Linear mechanisms: V/m. |
| `kI` | V/(rot·s) | Volts per accumulated position error over time. Linear: V/(m·s). |
| `kD` | V/(rot/s) | Volts per unit of velocity error. Linear: V/(m/s). |

For position control the error is in rotations of the mechanism (or meters with linear closed loop control). For velocity control it is in rotations per second (or meters per second).

</details>

<details>

<summary>Feedforward gains</summary>

| Gain | SimpleMotorFeedforward (flywheels) | ArmFeedforward (arms, pivots) | ElevatorFeedforward (elevators) |
| --- | --- | --- | --- |
| `kS` | V, static friction | V, static friction | V, static friction |
| `kG` | Not used | V, holds the arm level; applied as `kG * cos(angle)` | V, constant against gravity |
| `kV` | V/(rot/s) | V/(rot/s) | V/(m/s) |
| `kA` | V/(rot/s²) | V/(rot/s²) | V/(m/s²) |

In YAMS, `ArmFeedforward` uses rotations for kV and kA, like the other feedforward types.

</details>

<details>

<summary>Motion profile settings in code</summary>

| Profile | Setting | Unit in code |
| --- | --- | --- |
| Trapezoidal | Max velocity | Any WPILib angular velocity (or linear velocity) unit |
| Trapezoidal | Max acceleration | Any WPILib angular acceleration (or linear acceleration) unit |
| Exponential | Max input | Volts |
| Exponential | kV, kA | V/(rot/s), V/(rot/s²) |

On the dashboard, the trapezoidal values are typed in RPM and RPM per second, as in the Motion profile tab above.

</details>

## Tuning order

Tune the feedforward first, then the feedback, one value at a time.

<figure><img src="../../.gitbook/assets/live-tuning-order.png" alt="Flywheel order: kS, kV with kP at zero, kP, kD if it oscillates, then optionally kA. Arm order: kG, kV and kA from SysId, kP, then kD."><figcaption><p>The order to tune in for a flywheel and for an arm.</p></figcaption></figure>

{% tabs %}
{% tab title="Flywheel" %}
With `SimpleMotorFeedforward`:

1. **kS**: raise slowly until the wheel just starts to move.
2. **kV**: set a target speed with kP = 0, then adjust kV until the steady speed matches.
3. **kP**: raise to remove the leftover error and speed up the response.
4. **kD**: only if it oscillates, add a little to damp it.
5. **kA** (optional): for faster spin up.

{% hint style="success" %}
**Units example:** a kV of 0.12 V/(rot/s) means 0.12 volts for every rotation per second. At 100 rot/s (6000 RPM) the feedforward contributes 12 V.
{% endhint %}
{% endtab %}

{% tab title="Arm" %}
With `ArmFeedforward`:

1. **kG**: hold the arm level (0° from horizontal) and raise kG until it stays there.
2. **kV and kA**: start from SysId characterization values.
3. **kP**: raise until tracking is quick but not oscillating.
4. **kD**: damp any oscillation that is left.

{% hint style="info" %}
`kG * cos(angle)` is applied, so there is no gravity compensation when the arm is vertical (±90° from horizontal) and full kG when it is level.
{% endhint %}
{% endtab %}
{% endtabs %}
