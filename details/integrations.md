---
icon: plug-circle-plus
---

# Live tuning

## What is live tuning?

Live tuning lets you tune your closed-loop controller and feedforward through NetworkTables with AdvantageScope or Elastic.

Live Tuning is set up automatically for any `SmartMotorController` whose config uses `TelemetryVerbosity.HIGH` (or a `SmartMotorControllerTelemetryConfig` passed to `withTelemetry(name, telemetryConfig)`, which also uses `HIGH`). When the motor controller sets up its telemetry, YAMS registers one shared `Live Tuning` command per subsystem (Commands v2) or mechanism (Commands v3). Every motor controller in that subsystem or mechanism adds itself to the same command.

### Setting up live tuning explicitly

If you set up telemetry yourself, for example with `SmartMotorController.setupTelemetry(NetworkTable, NetworkTable)`, the `Live Tuning` command is not registered for you. Call `setupLiveTuning()` on the config to opt in:

```java
SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
    .withTelemetry("ArmMotor", TelemetryVerbosity.HIGH);
SmartMotorController motor = new TalonFXWrapper(armMotor, DCMotor.getKrakenX60(1), motorConfig);

motor.setupTelemetry(telemetryTable, tuningTable);
motorConfig.setupLiveTuning();
```

{% hint style="info" %}
`setupLiveTuning()` does nothing until the `SmartMotorController` has set up its telemetry (that is what attaches the controller to the config), or when the config has no subsystem (Commands v2) or mechanism (Commands v3). Registering two different subsystem or mechanism instances with the same name throws an `IllegalStateException`, so give each one a unique name (for example `LeftTurret` and `RightTurret`).
{% endhint %}

In C++ there is no explicit call: the `Live Tuning` command is registered when `SetupTelemetry()` runs with `TelemetryVerbosity::HIGH` and the config has a subsystem.

## TunerX in simulation (CTRE hardware)

For TalonFX and TalonFXS motors, YAMS feeds its physics simulation results back through CTRE's vendorsim layer via Phoenix's `getSimState()` API. This means Phoenix Tuner X can connect to your simulation and display every motor signal, position, velocity, closed-loop error, applied voltage, current draw, exactly as it would on real hardware.

**To use TunerX in sim:**

1. In a terminal, run your robot code in simulation (for example, `./gradlew simulateJava`).
2. Open Phoenix Tuner X.
3. Connect to the simulation instance. It appears as a local device.
4. In the motor view, open the **Signal** tab. The physics simulation updates every signal in real time.

This is the same workflow as diagnosing real hardware. You can verify gain behavior, confirm soft/hard limit triggering, and plot signal graphs without ever touching a physical robot.

{% hint style="info" %}
TunerX vendorsim integration is specific to **CTRE TalonFX and TalonFXS** motors. REV Spark motors use WPILib's simulation framework and are visible through AdvantageScope or Elastic via NetworkTables instead.
{% endhint %}

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

## How do I use live tuning?

{% hint style="danger" %}
Live tuning can be **dangerous**. Test it in simulation before using it on the robot.
{% endhint %}

The `Live Tuning` command is published to `NT:/Tuning/SUBSYSTEM_NAME/Live Tuning`, where `SUBSYSTEM_NAME` is the name of the subsystem (Commands v2) or mechanism (Commands v3) given to the `SmartMotorControllerConfig`. The tunable values themselves are published under `NT:/Tuning/`.

{% tabs %}
{% tab title="Commands v2" %}
In Elastic, drag the `Live Tuning` command from `NT:/Tuning/SUBSYSTEM_NAME/Live Tuning`. It is a regular WPILib Commands v2 `Command` that requires the subsystem.

After you enable the robot, press the button to enable Live tuning.
{% endtab %}

{% tab title="Commands v3" %}
Commands v3 commands are not dashboard widgets themselves, so YAMS publishes the `Live Tuning` command through `yams.commands3.telemetry.CommandTunable`. It shows up as a `Command` with a single `running` boolean at `NT:/Tuning/MECHANISM_NAME/Live Tuning/running`:

* Set `running` to `true` to schedule the command on the default `Scheduler`.
* Set `running` to `false` to cancel it.
* `running` reads back whether the command is currently running.

The command requires the mechanism and runs a `while (true)` loop that applies the dashboard values and calls `coroutine.yield()` each loop.
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
The `Live Tuning` command requires the subsystem or mechanism, so it can be interrupted by the controller, a default command, or any other trigger for that subsystem or mechanism in your program! Dashboard edits are only applied while it runs.
{% endhint %}

<figure><img src="../.gitbook/assets/new_liveutinging.gif" alt=""><figcaption></figcaption></figure>

You can tune your mechanism while the robot is enabled.

<figure><img src="../.gitbook/assets/AScopeLiveTuningExample.gif" alt=""><figcaption></figcaption></figure>

## When should I not use live tuning?

Do not use Live tuning before testing it in simulation. Understand its behavior first.

## Tunable Parameters and Their Units

When live tuning, it's important to understand what units each parameter expects. Below is a comprehensive reference for all tunable values.

### Units and AdvantageScope

YAMS publishes read-only fields and tunable fields in different units:

| Quantity | Read-only fields | Tunable fields |
| --- | --- | --- |
| Angular position | **Rotations** | **Degrees** |
| Angular velocity | **Rotations per Second** | **Rotations per Minute (RPM)** |
| Angular acceleration | **Rotations per Second²** | **RPM per Second** |
| Linear position, velocity, acceleration (linear closed loop control) | **Meters**, **Meters per Second**, **Meters per Second²** | Same as read-only |
| Voltage, current | **Volts**, **Amps** | **Volts**, **Amps** |
| Temperature | **Fahrenheit** | Not tunable |

This means the value you type into a tunable field is in the tunable unit, not the unit of the read-only field next to it. For example, a flywheel target of 3000 RPM is entered as `3000` in `TunableSetpointVelocity` while `MechanismVelocity` reads `50` rot/s (3000 ÷ 60), and an arm soft limit of 30° is entered as `30` in `MechanismUpperLimit` while `MechanismPosition` reads `0.0833` rotations (30 ÷ 360).

**AdvantageScope solves this.** Its built-in unit conversion lets you display any read-only field in whatever units feel natural for your mechanism, degrees for arms, RPM for flywheels, inches for elevators, without changing the underlying data or your robot code. To set it up:

1. Drag the telemetry field (e.g. `NT:/Mechanisms/Arm/ArmMotor/MechanismPosition`) into a Line Graph
2. Right-click the field in the graph legend → **Convert Units…**
3. Pick your target unit (Rotations → Degrees, rot/s → RPM, Meters → Inches, etc.)

Once conversion is applied, the graph shows the mechanism position in degrees and velocity in RPM, the same units the tunable fields expect, so you can type `180` into a tunable position field when the graph shows 180°.

{% hint style="success" %}
**Recommended setup for a tuning session:** Open AdvantageScope and configure unit conversions for `MechanismPosition` (→ degrees or inches), `MechanismVelocity` (→ deg/s, RPM, or in/s), and `ClosedLoopError` (same as position). Keep these graphs visible alongside the tunable fields in your NT dashboard. The converted read-only fields give you a real-unit view of what the mechanism is actually doing while you enter raw-unit values into the tunable fields.
{% endhint %}

### PID Controller Parameters

| Parameter         | Symbol | Unit                                  | Description                                                                                      |
| ----------------- | ------ | ------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Proportional Gain | kP     | Volts per Rotation (V/rot)            | Output voltage per unit of position error. For linear mechanisms, this is V/m.                   |
| Integral Gain     | kI     | Volts per Rotation-Second (V/(rot·s)) | Output voltage per accumulated position error over time. For linear mechanisms, this is V/(m·s). |
| Derivative Gain   | kD     | Volts per Rotation/Second (V/(rot/s)) | Output voltage per unit of velocity error. For linear mechanisms, this is V/(m/s).               |

{% hint style="info" %}
**Position Control**: Error is measured in **Rotations** (mechanism output shaft) or **Meters** (for linear mechanisms with linear closed loop controllers set).

**Velocity Control**: Error is measured in **Rotations per Second** or **Meters per Second**.
{% endhint %}

### Feedforward Parameters

#### SimpleMotorFeedforward (Flywheels, Simple Velocity Control)

| Parameter         | Symbol | Unit                                    | Description                                                              |
| ----------------- | ------ | --------------------------------------- | ------------------------------------------------------------------------ |
| Static Gain       | kS     | Volts (V)                               | Voltage to overcome static friction. Applied in the direction of motion. |
| Velocity Gain     | kV     | Volts per Rotation/Second (V/(rot/s))   | Voltage per unit of target velocity.                                     |
| Acceleration Gain | kA     | Volts per Rotation/Second² (V/(rot/s²)) | Voltage per unit of target acceleration.                                 |

#### ArmFeedforward (Arms, Pivots)

| Parameter         | Symbol | Unit                                    | Description                                                                     |
| ----------------- | ------ | --------------------------------------- | ------------------------------------------------------------------------------- |
| Static Gain       | kS     | Volts (V)                               | Voltage to overcome static friction.                                            |
| Gravity Gain      | kG     | Volts (V)                               | Voltage to hold the arm horizontal (at 0°). Actual output is `kG * cos(angle)`. |
| Velocity Gain     | kV     | Volts per Rotation/Second (V/(rot/s))   | Voltage per unit of target angular velocity.                                    |
| Acceleration Gain | kA     | Volts per Rotation/Second² (V/(rot/s²)) | Voltage per unit of target angular acceleration.                                |

{% hint style="info" %}
In YAMS, `ArmFeedforward` uses **Rotations** for kV and kA units, consistent with other feedforward types. The cosine calculation for gravity compensation is handled internally.
{% endhint %}

#### ElevatorFeedforward (Elevators, Linear Lifts)

| Parameter         | Symbol | Unit                               | Description                                                                                      |
| ----------------- | ------ | ---------------------------------- | ------------------------------------------------------------------------------------------------ |
| Static Gain       | kS     | Volts (V)                          | Voltage to overcome static friction.                                                             |
| Gravity Gain      | kG     | Volts (V)                          | Constant voltage to counteract gravity. Applied continuously when moving up or holding position. |
| Velocity Gain     | kV     | Volts per Meter/Second (V/(m/s))   | Voltage per unit of target velocity.                                                             |
| Acceleration Gain | kA     | Volts per Meter/Second² (V/(m/s²)) | Voltage per unit of target acceleration.                                                         |

### Motion Profile Parameters

#### Trapezoidal Profile

| Parameter        | Symbol | Unit                           | Description                                                                    |
| ---------------- | ------ | ------------------------------ | ------------------------------------------------------------------------------ |
| Max Velocity     | maxV   | Rotations per Second (rot/s)   | Maximum velocity during the cruise phase. For linear: Meters per Second (m/s). |
| Max Acceleration | maxA   | Rotations per Second² (rot/s²) | Acceleration/deceleration rate. For linear: Meters per Second² (m/s²).         |

#### Exponential Profile

| Parameter         | Symbol | Unit                                    | Description                                               |
| ----------------- | ------ | --------------------------------------- | --------------------------------------------------------- |
| Max Input Voltage | V      | Volts (V)                               | Maximum voltage the profile will command.                 |
| kV                | kV     | Volts per Rotation/Second (V/(rot/s))   | System velocity constant from motor characterization.     |
| kA                | kA     | Volts per Rotation/Second² (V/(rot/s²)) | System acceleration constant from motor characterization. |

### Soft Limits

| Parameter             | Unit                          | Description                                       |
| --------------------- | ----------------------------- | ------------------------------------------------- |
| Lower Limit           | Rotations (rot) or Meters (m) | Minimum allowed mechanism position.               |
| Upper Limit           | Rotations (rot) or Meters (m) | Maximum allowed mechanism position.               |
| Closed Loop Tolerance | Rotations (rot) or Meters (m) | Position error threshold for "at goal" detection. |

### Current Limits

| Parameter            | Unit     | Description                                                         |
| -------------------- | -------- | ------------------------------------------------------------------- |
| Stator Current Limit | Amps (A) | Maximum current through the motor windings. Controls torque output. |
| Supply Current Limit | Amps (A) | Maximum current drawn from the battery. Prevents brownouts.         |

### Ramp Rates

| Parameter             | Unit        | Description                                              |
| --------------------- | ----------- | -------------------------------------------------------- |
| Open Loop Ramp Rate   | Seconds (s) | Time to ramp from 0% to 100% output in open loop mode.   |
| Closed Loop Ramp Rate | Seconds (s) | Time to ramp from 0% to 100% output in closed loop mode. |

### Temperature

| Parameter          | Unit         | Description                                                   |
| ------------------ | ------------ | ------------------------------------------------------------- |
| Temperature Cutoff | Celsius (°C) | Motor controller will stop if temperature exceeds this value. |

### Voltage

| Parameter               | Unit      | Description                                                                           |
| ----------------------- | --------- | ------------------------------------------------------------------------------------- |
| Voltage Compensation    | Volts (V) | Nominal voltage for consistent behavior regardless of battery voltage. Typically 12V. |
| Closed Loop Max Voltage | Volts (V) | Maximum voltage output from the closed loop controller.                               |

## Tunable Fields and What to Watch

Every tunable field has a corresponding read-only field you should monitor while adjusting it. The table below maps each tunable parameter to the telemetry signals that tell you whether your change is working.

### PID Gains

| Tunable Field | Unit                      | What to Watch (Read-Only)                                    | Why                                                                                                      |
| ------------- | ------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| `kP`          | V/rot (or V/m for linear) | `MechanismPosition` vs `SetpointPosition`, `ClosedLoopError` | Error should shrink faster as kP rises. Oscillation means kP is too high.                                |
| `kI`          | V/(rot·s)                 | `ClosedLoopError` over time                                  | Persistent steady-state error that won't converge, kI eliminates it. Watch for windup causing overshoot. |
| `kD`          | V/(rot/s)                 | `MechanismVelocity`, `OutputVoltage`                         | Dampens oscillation. If voltage spikes on direction changes, kD is too high.                             |

### Feedforward Gains

| Tunable Field         | Unit       | What to Watch (Read-Only)                                    | Why                                                                                                                      |
| --------------------- | ---------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| `kS`                  | V          | `OutputVoltage` at rest                                      | Voltage to overcome static friction. Increase until the mechanism just barely starts moving from rest.                   |
| `kV`                  | V/(rot/s)  | `MeasurementVelocity` vs `SetpointVelocity`, `OutputVoltage` | At steady state (kP = 0), adjust kV until measured velocity matches setpoint velocity.                                   |
| `kA`                  | V/(rot/s²) | `MeasurementVelocity` during acceleration ramp               | Improves tracking during the acceleration phase of a motion profile.                                                     |
| `kG` (Arms/Elevators) | V          | `MechanismPosition`, `OutputVoltage` at horizontal           | For arms: hold horizontal and increase until the arm holds position. For elevators: constant offset that fights gravity. |

### Motion Profile Constraints

| Tunable Field                       | Unit             | What to Watch (Read-Only)                 | Why                                                                                                 |
| ----------------------------------- | ---------------- | ----------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `TrapezoidalProfileMaxVelocity`     | RPM (or m/s)     | `MechanismVelocity` during cruise phase   | The plateau velocity. Increase to move faster; decrease if the motor is stalling during the cruise. |
| `TrapezoidalProfileMaxAcceleration` | RPM/s (or m/s²)  | `StatorCurrent`, `MechanismVelocity` ramp | Controls how quickly velocity builds. High acceleration = high current spike at start of motion.    |
| `TrapezoidalProfileMaxJerk`         | RPM/s²           | `MechanismVelocity` shape                 | Smooths the velocity ramp. Useful when acceleration discontinuities cause mechanical shock.         |

### Setpoints (Tunable Test Inputs)

| Tunable Field             | Unit           | What to Watch (Read-Only)              | Why                                                                                                                   |
| ------------------------- | -------------- | -------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `TunableSetpointPosition` | deg (or m)     | `MechanismPosition`, `ClosedLoopError` | Command the mechanism to a specific position without button bindings. Use to test individual setpoints during tuning. |
| `TunableSetpointVelocity` | RPM (or m/s)   | `MeasurementVelocity`, `OutputVoltage` | Command a specific velocity. Use to characterize kV without running a full motion profile.                            |

### Soft Limits

| Tunable Field         | Unit       | What to Watch (Read-Only)                            | Why                                                                                                                     |
| --------------------- | ---------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `MechanismLowerLimit` | deg        | `MechanismPosition`, `MechanismLowerLimit` (boolean) | Boolean flips true when the mechanism hits the lower soft limit. Adjust the limit to change where the controller clips. |
| `MechanismUpperLimit` | deg        | `MechanismPosition`, `MechanismUpperLimit` (boolean) | Same for upper limit.                                                                                                   |

Linear mechanisms also publish `MeasurementLowerLimit` and `MeasurementUpperLimit`, tunable in meters.

### Current Limits

| Tunable Field        | Unit | What to Watch (Read-Only)            | Why                                                                                          |
| -------------------- | ---- | ------------------------------------ | -------------------------------------------------------------------------------------------- |
| `StatorCurrentLimit` | A    | `StatorCurrent`, `MechanismVelocity` | Lower stator limit = lower torque cap. Watch velocity sag when the limit is too restrictive. |
| `SupplyCurrentLimit` | A    | `SupplyCurrent`, `OutputVoltage`     | Prevents battery brownouts. Reduce if SupplyCurrent spikes cause voltage dips.               |

### Ramp Rates

| Tunable Field        | Unit       | What to Watch (Read-Only)                 | Why                                                                                          |
| -------------------- | ---------- | ----------------------------------------- | -------------------------------------------------------------------------------------------- |
| `OpenloopRampRate`   | s (0→100%) | `OutputVoltage` during open-loop commands | Slows voltage changes in open loop. Reduces mechanical shock on `set(dutyCycle)` commands.   |
| `ClosedloopRampRate` | s (0→100%) | `OutputVoltage` during closed-loop ramp   | Similar but for closed loop. Useful when step-changes in setpoint cause large voltage jumps. |

{% hint style="info" %}
**Useful read-only fields at all times during tuning:**

* `Temperature`, watch for thermal throttling masking gain problems
* `OutputVoltage`, the raw command the motor is receiving; confirms your feedforward math
* `RotorPosition` / `RotorVelocity`, pre-gearing values; useful for verifying gearing configuration is correct
{% endhint %}

## Example: Tuning a Flywheel

When tuning a flywheel with `SimpleMotorFeedforward`:

1. **Start with kS**: Slowly increase until the wheel just barely starts moving
2. **Tune kV**: Set a target velocity, then adjust kV until the steady-state velocity matches (with kP = 0)
3. **Add kP**: Increase proportional gain to reduce steady-state error and improve response time
4. **Fine-tune kD**: If there's oscillation, add derivative gain to dampen it
5. **Add kA** (optional): If you need faster acceleration response, tune the acceleration feedforward

{% hint style="success" %}
**Units Example**: If your flywheel has a kV of 0.12 V/(rot/s), that means 0.12 volts are needed for every rotation per second of velocity. At 100 rot/s (6000 RPM), the feedforward would contribute 12V.
{% endhint %}

## Example: Tuning an Arm

When tuning an arm with `ArmFeedforward`:

1. **Find kG**: Hold the arm horizontal (90° from vertical or 0° from horizontal depending on your coordinate system) and increase kG until it holds position
2. **Set kV and kA**: These come from SysId characterization
3. **Tune kP**: Increase until position tracking is responsive but not oscillating
4. **Add kD**: Dampen any oscillations

{% hint style="info" %}
Remember that `kG * cos(angle)` is applied, so at vertical (±90° from horizontal), no gravity compensation is applied, and at horizontal (0°), full kG is applied.
{% endhint %}
