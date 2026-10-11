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

{% tabs %}
{% tab title="AdvantageScope" %}
AdvantageScope only reads NetworkTables until you turn on **tuning mode**. Click the tuning mode button, the sliders icon beside the search box at the top of the sidebar. It is only shown while AdvantageScope is connected to a live robot or simulation.

<figure><img src="../../.gitbook/assets/live-tuning-ascope.png" alt="Schematic of the AdvantageScope sidebar: step 1, click the tuning mode button, a sliders icon beside the search box; step 2, type a new value into a field under NT Tuning"><figcaption><p>Turn tuning mode on, then edit the fields under <code>NT:/Tuning</code>.</p></figcaption></figure>

With tuning mode on, the fields under `NT:/Tuning` can be edited in the sidebar. If typing a value does nothing, check this button first.
{% endtab %}

{% tab title="Elastic" %}
Drag a field from `NT:/Tuning/` onto the dashboard as a text or number widget and type the new value. Elastic can always write to NetworkTables, so there is nothing to turn on.
{% endtab %}
{% endtabs %}
{% endstep %}

{% step %}
#### Watch what happens

Graph the matching read-only fields under `NT:/Mechanisms/`. [NetworkTables keys](#networktables-keys) shows where they are, and [What to tune and what to watch](#what-to-tune-and-what-to-watch) lists which key to watch for each value.
{% endstep %}
{% endstepper %}

{% hint style="warning" %}
The `Live Tuning` command requires the subsystem or mechanism, so the controller, a default command, or any other trigger for that subsystem or mechanism can interrupt it. Dashboard edits stop being applied when it is interrupted.
{% endhint %}

<figure><img src="../../.gitbook/assets/new_liveutinging.gif" alt="Starting the Live Tuning command and editing a tunable value on the dashboard"><figcaption><p>Starting the command and changing a value.</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/AScopeLiveTuningExample.gif" alt="Tuning a mechanism in AdvantageScope while the robot is enabled"><figcaption><p>Tuning while the robot is enabled.</p></figcaption></figure>

## NetworkTables keys

Everything on this page is named by its NetworkTables key, which is what you see in AdvantageScope and Elastic.

| What | Where it is published |
| --- | --- |
| Values you type | `NT:/Tuning/<Mechanism>/<Motor>/<key>` |
| Values you watch | `NT:/Mechanisms/<Mechanism>/<Motor>/<key>` |
| The `Live Tuning` command | `NT:/Tuning/<Subsystem>/Live Tuning` |

`<Mechanism>` and `<Motor>` are the names given to `withTelemetry(...)` on the mechanism config and on the `SmartMotorControllerConfig`. A motor controller used without a mechanism class has no `<Mechanism>` level.

For example, an arm named `Arm` with a motor named `ArmMotor`:

* You type kP into `NT:/Tuning/Arm/ArmMotor/closedloop/feedback/kP`.
* You watch the arm at `NT:/Mechanisms/Arm/ArmMotor/mechanism/position`.

{% hint style="info" %}
Some keys exist in both tables. `closedloop/setpoint/position` under `NT:/Tuning` is the setpoint you type. Under `NT:/Mechanisms` it is the setpoint the controller is using right now.
{% endhint %}

### The position and velocity to watch

Which keys hold the position and velocity depends on the mechanism.

| Mechanism | Position | Velocity | Units |
| --- | --- | --- | --- |
| Rotational: arms, pivots, turrets, flywheels | `mechanism/position` | `mechanism/velocity` | Rotations, rot/s |
| Linear: elevators, and anything with a mechanism circumference | `measurement/position` | `measurement/velocity` | Meters, m/s |

The tables below write `mechanism/position` and `mechanism/velocity`. On a linear mechanism, read them as `measurement/position` and `measurement/velocity`.

## Units: what you type and what you read

Tunable fields and read-only fields do not use the same units.

<figure><img src="../../.gitbook/assets/live-tuning-units.png" alt="For a flywheel you type 3000 RPM into closedloop/setpoint/velocity and read 50 rotations per second at mechanism/velocity. For an arm you type 30 degrees into mechanism/limit/upper and read 0.0833 rotations at mechanism/position."><figcaption><p>The value you type is in the tunable unit, not the unit of the read-only field next to it.</p></figcaption></figure>

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

1. Drag the telemetry field, for example `NT:/Mechanisms/Arm/ArmMotor/mechanism/position`, into a Line Graph.
2. Right-click the field in the graph legend and choose **Convert Units…**
3. Pick the target unit: Rotations to Degrees, rot/s to RPM, Meters to Inches, and so on.

Now the graph shows degrees and RPM, the same units the tunable fields expect. When the graph shows 180°, you type `180`.

{% hint style="success" %}
**Recommended setup for a tuning session**

* Convert `mechanism/position` and `closedloop/setpoint/position` to degrees. On a linear mechanism, convert `measurement/position` to inches if you prefer.
* Convert `mechanism/velocity` to RPM (or deg/s).
* Graph `closedloop/setpoint/position` and `mechanism/position` together. YAMS has no closed loop error field: the error is the gap between the two lines.
* Keep the graphs visible next to the tunable fields.
{% endhint %}

## What to tune and what to watch

Every tunable field has a read-only field that tells you whether your change worked.

<figure><img src="../../.gitbook/assets/live-tuning-response.png" alt="Three step responses: kP too low rises slowly and stops short, about right settles quickly on the setpoint, kP too high overshoots and rings"><figcaption><p>What the position graph looks like after a setpoint change.</p></figcaption></figure>

{% tabs %}
{% tab title="PID" %}
| You type (`NT:/Tuning`) | Watch (`NT:/Mechanisms`) | Why |
| --- | --- | --- |
| `closedloop/feedback/kP` | `mechanism/position` vs `closedloop/setpoint/position` | Error should shrink faster as kP rises. Oscillation means kP is too high. |
| `closedloop/feedback/kI` | `mechanism/position` vs `closedloop/setpoint/position` over time | Removes a steady error that never closes. Watch for windup causing overshoot. |
| `closedloop/feedback/kD` | `mechanism/velocity`, `motor/outputVoltage` | Dampens oscillation. If voltage spikes on direction changes, kD is too high. |

For a velocity mechanism such as a flywheel, compare `mechanism/velocity` with `closedloop/setpoint/velocity` instead.

Gains are in volts per rotation of error (per meter for linear mechanisms). See [Gain units](#gain-units) below.
{% endtab %}

{% tab title="Feedforward" %}
| You type (`NT:/Tuning`) | Watch (`NT:/Mechanisms`) | Why |
| --- | --- | --- |
| `closedloop/feedforward/kS` | `motor/outputVoltage` at rest | Voltage to overcome static friction. Raise until the mechanism just starts to move. |
| `closedloop/feedforward/kV` | `mechanism/velocity` vs `closedloop/setpoint/velocity`, `motor/outputVoltage` | With kP = 0, adjust until measured velocity matches the setpoint at steady state. |
| `closedloop/feedforward/kA` | `mechanism/velocity` during the acceleration ramp | Improves tracking while a motion profile accelerates. |
| `closedloop/feedforward/kG` (arms, elevators) | `mechanism/position`, `motor/outputVoltage` | Arms: hold level and raise until the arm stays put. Elevators: a constant that fights gravity. |

{% hint style="info" %}
For arms, `kG * cos(angle)` is applied: full kG when level (0°), none when vertical (±90°). YAMS handles the cosine for you.
{% endhint %}
{% endtab %}

{% tab title="Motion profile" %}
All of these keys start with `closedloop/motionprofile/`.

| You type (`NT:/Tuning`) | Units | Watch (`NT:/Mechanisms`) | Why |
| --- | --- | --- | --- |
| `maxVelocity` | RPM (or m/s) | `mechanism/velocity` while cruising | The plateau speed. Raise to move faster; lower if the motor stalls while cruising. |
| `maxAcceleration` | RPM/s (or m/s²) | `current/stator`, `mechanism/velocity` ramp | How quickly speed builds. High acceleration means a current spike at the start. |
| `maxJerk` | RPM/s² | Shape of `mechanism/velocity` | Smooths the ramp when sudden acceleration causes mechanical shock. |
| `kV`, `kA` | Same units as the feedforward kV, kA | `mechanism/velocity` | The system constants the exponential profile plans with. |
| `maxInput` | Volts | `motor/outputVoltage` | The most voltage the exponential profile will ask for. |
{% endtab %}

{% tab title="Setpoints" %}
| You type (`NT:/Tuning`) | Units | Watch (`NT:/Mechanisms`) | Why |
| --- | --- | --- | --- |
| `closedloop/setpoint/position` | Degrees (or m) | `mechanism/position` vs `closedloop/setpoint/position` | Send the mechanism to a position without a button binding. |
| `closedloop/setpoint/velocity` | RPM (or m/s) | `mechanism/velocity`, `motor/outputVoltage` | Command a speed, for example to find kV without a full profile. |
| `closedloop/slot` | Slot number | `closedloop/slot` | Switch which gain slot the controller uses. |
{% endtab %}

{% tab title="Limits" %}
| You type (`NT:/Tuning`) | Units | Watch (`NT:/Mechanisms`) | Why |
| --- | --- | --- | --- |
| `mechanism/limit/lower` | Degrees | `mechanism/position`, `limits/mechanism/lower` (boolean) | The boolean turns true at the lower soft limit. Move the limit to change where it stops. |
| `mechanism/limit/upper` | Degrees | `mechanism/position`, `limits/mechanism/upper` (boolean) | The same for the upper soft limit. |
| `measurement/limit/lower`, `measurement/limit/upper` | Meters | `measurement/position` | Soft limits for linear mechanisms. |
| `current/limit/stator` | Amps | `current/stator`, `mechanism/velocity` | Caps torque. Watch for speed sagging when the limit is too low. |
| `current/limit/supply` | Amps | `current/supply`, `motor/outputVoltage` | Prevents brownouts. Lower it if supply current spikes dip the voltage. |
{% endtab %}

{% tab title="Ramp rates and inversion" %}
| You type (`NT:/Tuning`) | Units | Watch (`NT:/Mechanisms`) | Why |
| --- | --- | --- | --- |
| `ramprate/dutycycle/openloop` | Seconds, 0 to 100% | `motor/outputVoltage` during open loop commands | Slows voltage changes in open loop. Reduces shock on `set(dutyCycle)`. |
| `ramprate/dutycycle/closedloop` | Seconds, 0 to 100% | `motor/outputVoltage` during closed loop moves | The same for closed loop, when a setpoint step causes a big voltage jump. |
| `motor/inverted` | Boolean | `mechanism/velocity` | Flips the motor direction. Positive voltage should give a positive velocity. |
| `encoder/inverted` | Boolean | `mechanism/position` | Flips the encoder direction. |
{% endtab %}
{% endtabs %}

{% hint style="info" %}
**Worth watching the whole time**

* `motor/temperature`: a hot motor backing off can look like a gain problem.
* `motor/outputVoltage`: the raw command the motor is getting. Confirms your feedforward math.
* `rotor/position`, `rotor/velocity`: values before gearing. Useful for checking the gearing is right.
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

{% hint style="success" %}
**New to tuning?** The WPILib docs explain what each gain does and have interactive simulations to practice on. Read the one for your mechanism before tuning the real thing:

* [Introduction to PID](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/introduction-to-pid.html) and [introduction to feedforward](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/introduction-to-feedforward.html)
* [Tuning a flywheel velocity controller](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/tuning-flywheel.html)
* [Tuning a turret position controller](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/tuning-turret.html)
* [Tuning a vertical arm position controller](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/tuning-vertical-arm.html)
* [Tuning a vertical elevator](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/tuning-elevator.html)
{% endhint %}

<figure><img src="../../.gitbook/assets/live-tuning-order.png" alt="Flywheel order: kS, kV with kP at zero, kP, kD if it oscillates, then optionally kA. Arm order: kG, kV until the velocity tracks, kP, then kD."><figcaption><p>The order to tune in for a flywheel and for an arm.</p></figcaption></figure>

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
2. **kV**: raise until `mechanism/velocity` tracks `closedloop/setpoint/velocity` during a move. Leave kA at 0 unless the arm lags while accelerating.
3. **kP**: raise until tracking is quick but not oscillating.
4. **kD**: damp any oscillation that is left.

{% hint style="info" %}
`kG * cos(angle)` is applied, so there is no gravity compensation when the arm is vertical (±90° from horizontal) and full kG when it is level.
{% endhint %}
{% endtab %}
{% endtabs %}

## Swerve drive

`SwerveDrive` has its own tuning command and its own keys. It can tune three things, one at a time:

| Mode | Turn it on with | What it does |
| --- | --- | --- |
| Auto-align | `autoalign/enabled` | Drives the robot to a pose you type, using the translation and rotation PID gains. |
| Drive motors | `modules/drive/enabled` | Runs every module's drive motor at a velocity you type. |
| Azimuth motors | `modules/azimuth/enabled` | Turns every module to an angle you type. |

| What | Where it is published |
| --- | --- |
| Values you type | `NT:/Tuning/<name>/<key>` |
| The tuning command | `NT:/Mechanisms/<name>/tuning` |
| The robot pose to watch | `NT:/Mechanisms/<name>/pose` |

`<name>` is the name given to `SwerveDriveConfig.withTelemetry(name, ...)`. The keys are published at `TelemetryVerbosity.HIGH`.

{% hint style="warning" %}
The tuning command is only published when both a translation and a rotation controller are set with `withTranslationController(...)` and `withRotationController(...)`. Nothing you type is applied unless the command is running.
{% endhint %}

### Start the tuning command

{% tabs %}
{% tab title="Commands v2" %}
Drag `NT:/Mechanisms/<name>/tuning` onto your Elastic dashboard and press its button, or bind it to a controller button.
{% endtab %}

{% tab title="Commands v3" %}
The command is published through `CommandTunable`. Set `NT:/Mechanisms/<name>/tuning/running` to `true` to start it and `false` to stop it.
{% endtab %}
{% endtabs %}

Starting the command resets the translation and rotation PID controllers.

### Pick a mode

{% tabs %}
{% tab title="Auto-align" %}
| You type (`NT:/Tuning/<name>/`) | Units | What it is |
| --- | --- | --- |
| `autoalign/enabled` | Boolean | Drive to the pose while `true`. |
| `autoalign/setpoint/x`, `autoalign/setpoint/y` | Meters | Field relative target position. |
| `autoalign/setpoint/rot` | Degrees | Field relative target heading. |
| `autoalign/translation/p`, `i`, `d` | PID on meters | Translation gains. |
| `autoalign/rotation/p`, `i`, `d` | PID on radians | Rotation gains. |

The gains start at the values given to `withTranslationController(...)` and `withRotationController(...)`, not `0`.

1. Start the tuning command.
2. Type the target into `autoalign/setpoint/x`, `autoalign/setpoint/y` and `autoalign/setpoint/rot`.
3. Set `autoalign/enabled` to `true`. The robot drives to the pose with the current gains.
4. Change a gain and watch `NT:/Mechanisms/<name>/pose` on a 2D field. Changing a gain only resets that controller when the value actually changed.
5. Copy the gains you like into `withTranslationController(...)` and `withRotationController(...)`. Values typed on the dashboard are lost when the robot code restarts.
6. Set `autoalign/enabled` back to `false` before handing control back to the driver.

<figure><img src="../../.gitbook/assets/autoalign-tuning-setup.png" alt="AdvantageScope 2D field next to the Tuning/swerve/autoalign tree with the enabled, setpoint and gain entries"><figcaption><p>The <code>Tuning/swerve/autoalign</code> tree next to a 2D field.</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/autoalign-tuning.gif" alt="The simulated robot drives to a target of x 12, y 4, rotation 90 while autoalign/enabled is true"><figcaption><p>Driving to <code>x=12</code>, <code>y=4</code>, <code>rot=90</code> in simulation.</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/autoalign-tuning-converged.png" alt="The robot settled at X 11.975 m, Y 3.977 m, 90 degrees against a target of 12, 4, 90"><figcaption><p>Settled at the target pose.</p></figcaption></figure>
{% endtab %}

{% tab title="Drive motors" %}
| You type (`NT:/Tuning/<name>/`) | Units | What it is |
| --- | --- | --- |
| `modules/drive/enabled` | Boolean | Run the drive motors while `true`. |
| `modules/drive/velocity` | Meters per second | Wheel speed for every module. |
| `modules/drive/inplace` | Boolean | `false`: every module points at 0° and the robot drives straight. `true`: the modules point tangent to the robot center and the robot spins in place. |
| `modules/drive/feedback/p`, `i`, `d` | PID gains | Drive motor feedback gains. |
| `modules/drive/feedforward/s`, `v`, `a` | Feedforward gains | Drive motor kS, kV, kA. |

The gains start at the first module's configured values.

1. Start the tuning command.
2. Type a speed into `modules/drive/velocity`.
3. Set `modules/drive/enabled` to `true`. Every module drives at that speed.
4. Change the gains. Each one is sent to the motors only when its value changes.
5. To test without the robot travelling across the field, set `modules/drive/inplace` to `true`. A positive velocity spins the robot counter-clockwise.
6. Copy the gains into the drive motor's `withClosedLoopController(...)` and `withFeedforward(...)`, then set `modules/drive/enabled` to `false`. The drive motors are commanded to 0 m/s once it is off.

<figure><img src="../../.gitbook/assets/module-drive-tuning.png" alt="AdvantageScope with modules/drive/enabled on and every module driving at minus 3 meters per second"><figcaption><p>Every module driving at the typed velocity.</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/module-drive-inplace.gif" alt="With modules/drive/inplace true, raising the velocity spins the chassis faster in place"><figcaption><p>Spinning in place with <code>modules/drive/inplace</code>.</p></figcaption></figure>
{% endtab %}

{% tab title="Azimuth motors" %}
| You type (`NT:/Tuning/<name>/`) | Units | What it is |
| --- | --- | --- |
| `modules/azimuth/enabled` | Boolean | Hold the angle while `true`. |
| `modules/azimuth/angle` | Degrees | Target angle for every module. |
| `modules/azimuth/feedback/p`, `i`, `d` | PID gains | Azimuth motor feedback gains. |
| `modules/azimuth/feedforward/s`, `v`, `a` | Feedforward gains | Azimuth motor kS, kV, kA. |

1. Start the tuning command.
2. Type an angle into `modules/azimuth/angle`.
3. Set `modules/azimuth/enabled` to `true`. Every module turns to that angle. The drive motors are held at 0, so the wheels only turn in place.
4. Change the gains and type new angles to see the response.
5. Copy the gains into the azimuth motor's `withClosedLoopController(...)` and `withFeedforward(...)`, then set `modules/azimuth/enabled` to `false`. The modules stay at the last angle; they do not return to 0°.

<figure><img src="../../.gitbook/assets/module-azimuth-tuning.gif" alt="Editing modules/azimuth/angle and every module turning to the new heading"><figcaption><p>Every module following <code>modules/azimuth/angle</code>.</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/module-azimuth-tuning.png" alt="The Tuning/swerve/modules tree in AdvantageScope with the azimuth and drive entries"><figcaption><p>The <code>Tuning/swerve/modules</code> tree.</p></figcaption></figure>
{% endtab %}
{% endtabs %}

### Good to know

* **One mode at a time.** If more than one `enabled` key is `true`, YAMS keeps one and sets the others back to `false`. Auto-align wins over drive tuning, and drive tuning wins over azimuth tuning.
* **Every module shares the values.** The `modules/drive/*` and `modules/azimuth/*` keys apply to all modules at once. There are no per-module keys.
* **Module optimization still applies.** Both module modes go through `SwerveDrive.setSwerveModuleStates(...)`, so `withOptimization(true)` behaves the same as it does while driving.
* **No gyro needed in simulation.** The simulated drive tracks its own heading, so you can tune the drive and azimuth motors before writing any gyro code. On a real robot, `SwerveDrive` throws an `IllegalStateException` if `withGyro(...)` was never called.
