---
icon: hand-pointer
---

# Telemetry

## The basics of telemetry

{% hint style="info" %}
Using DataLog will increase your storage on the RIO and could have negative consequences. Please review [WPILib ](https://docs.wpilib.org/en/stable/docs/software/telemetry/datalog.html#custom-data-logging-using-datalog)for more information.
{% endhint %}

By default telemetry is sent to NetworkTables via NT4 entries. This is not the only way telemetry can be logged! If you set a DataLogName inside of `SmartMotorControllerTelemetryConfig` with `SmartMotorControllerTelemetryConfig.withDataLogName(entry)` it will log everything that is sent to NT4 in that log. However this can pollute your telemetry quite alot! We recommend pruning your telemetry so that only things you absolutely want in the file will appear.

You can disable NetworkTables logging with `SmartMotorControllerTelemetryConfig.withoutNetworkTables()` !

{% hint style="success" %}
You should set a DataLogName for competition matches to ensure you have enough data to see what happened on the field and make real time adjustments. This will dramatically improve your season and is very easy to do!
{% endhint %}

{% hint style="info" %}
Looking for a deeper dive on enabling/disabling NetworkTables, choosing what to log, and how this all works (or doesn't) for `SwerveDrive`? See [DataLog Best Practices](../details/datalog-best-practices.md). For a step-by-step walkthrough of configuring a `SwerveDrive` to log and then opening the resulting file in AdvantageScope, see [How do I view my DataLog in AdvantageScope?](../how-to/how-do-i-view-my-datalog-in-advantagescope.md).
{% endhint %}

`MechanismTelemetry` is the class that actually coordinates all of this under the hood, every Mechanism (and every `SwerveModule`) owns one, and it is what wires a Mechanism's NT4 publishers to a WPILib `DataLog` entry whenever a DataLogName is configured, so both destinations always see the same values.

## NetworkTables

Every Mechanism has a Tuning table and a display table on the robot. Mechanism tables contain the SmartMotorControllers used in that mechanism.

All Mechanism tables are stored under `NT:/Mechanisms` NOT `NT:/SmartDashboard` however `NT:/SmartDashboard` does contain some useful Mechanism2d's that can be shown in your dashboard, and commands like "Live Tuning"

The image below has the Telemetry output of an Elevator with the Telemetry name of `Elevator` and the SmartMotorController TelemetryName of `ElevatorMotor`

## Simulation vs Reality

The simulation view of each mechanism is done with `Mechanism2d`'s. These windows could be outputted while on the actual robot but that is not necessary as all of the data from the `Mechanism2d` is in the Telemetry fields and accessible to the user.

{% embed url="https://docs.wpilib.org/en/stable/docs/software/dashboards/glass/mech2d-widget.html" %}

## Mechanism Telemetry

Mechanism Telemetry is basically only a holder table for SmartMotorController Telemetry allowing for an easier time finding the values.

<figure><img src="../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

## Smart Motor Controller Telemetry

SmartMotorController Telemetry is highly configurable in fact your could enable and disable whatever telemetry you like with the example below.

```java
SmartMotorControllerTelemetryConfig motorTelemetryConfig = new SmartMotorControllerTelemetryConfig()
          .withMechanismPosition()
          .withRotorPosition()
          .withMechanismLowerLimit()
          .withMechanismUpperLimit();
          
SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withClosedLoopController(4, 0, 0, DegreesPerSecond.of(180), DegreesPerSecondPerSecond.of(90))
      .withSoftLimits(Degrees.of(-30), Degrees.of(100))
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
      .withIdleMode(MotorMode.BRAKE)
      .withTelemetry("ElevatorMotor", motorTelemetryConfig)
```

If a field you want doesn't have its own `with*()` method, or you want to enable/disable a whole set of fields in one call, use the `withCustom(...)` escape hatch. It takes a `BooleanTelemetryField`/`DoubleTelemetryField` (or an array of either) plus a boolean to enable or disable them:

```java
SmartMotorControllerTelemetryConfig motorTelemetryConfig = new SmartMotorControllerTelemetryConfig()
          .withTelemetryVerbosity(TelemetryVerbosity.LOW)
          .withCustom(SmartMotorControllerTelemetry.DoubleTelemetryField.SupplyCurrent, true)
          .withCustom(new SmartMotorControllerTelemetry.BooleanTelemetryField[]{
              SmartMotorControllerTelemetry.BooleanTelemetryField.MotorInversion,
              SmartMotorControllerTelemetry.BooleanTelemetryField.EncoderInversion
          }, false);
```

See the `SmartMotorControllerTelemetryConfig` API reference page for the full method list. `withCustom(...)` is currently Java-only.

## StructTelemetry and StructArrayTelemetry

`SmartMotorControllerTelemetry` publishes plain numbers and booleans with `DoubleTelemetry`/`BooleanTelemetry`. Swerve telemetry needs to publish NT4 struct-encoded types instead, a single `Pose2d` or `ChassisSpeeds`, or an array of `SwerveModuleState`, so it's built on two generic counterparts:

* `StructTelemetry<T>` publishes a single struct-serializable value (anything WPILib can encode as an NT4 struct: `Pose2d`, `ChassisSpeeds`, `SwerveModuleState`, etc.).
* `StructArrayTelemetry<T>` publishes an array of that same kind of value, used for the four-module `SwerveModuleState[]` fields.

Both work exactly like `DoubleTelemetry`, they can be enabled/disabled per field, wired to NetworkTables and/or a DataLog entry, and (for fields marked tunable) read back a live-edited value from NetworkTables. You won't normally construct these directly, `SwerveDriveTelemetryConfig` and `SwerveModuleTelemetryConfig` (below) create and manage them for you.

## Swerve Drive Telemetry

`SwerveDrive` and `SwerveModule` don't use `SmartMotorControllerTelemetryConfig`, they have their own dedicated config classes, `SwerveDriveTelemetryConfig` and `SwerveModuleTelemetryConfig`, but the shape is the same: every field is disabled by default, `withTelemetryVerbosity(TelemetryVerbosity)` turns on a cumulative preset (`LOW` → `MID` → `HIGH`), and individual `with*()` methods let you enable/disable fields one at a time on top of (or instead of) a preset. `withoutNetworkTables()`/`withNetworkTables(boolean)` and `withDataLogName(name)` work exactly like they do on `SmartMotorControllerTelemetryConfig`.

```java
SwerveDriveTelemetryConfig driveTelemetryConfig = new SwerveDriveTelemetryConfig()
    .withTelemetryVerbosity(TelemetryVerbosity.HIGH)
    .withPose()
    .withGyro()
    .withCurrentRobotRelativeChassisSpeeds()
    .withFieldRelativeChassisSpeeds()
    .withDesiredRobotRelativeChassisSpeeds()
    .withDesiredModuleStates()
    .withCurrentModuleStates()
    .withDataLogName("swerve");

SwerveDriveConfig driveConfig = new SwerveDriveConfig(this, frontLeft, frontRight, backLeft, backRight)
    .withGyro(gyro.getYaw().asSupplier())
    .withTelemetry("swerve", driveTelemetryConfig);
```

Unlike a single-motor mechanism's `withTelemetry(...)`, `SwerveDriveConfig.withTelemetry(...)` always takes the telemetry name as its first argument: `withTelemetry(String name, TelemetryVerbosity)` or `withTelemetry(String name, SwerveDriveTelemetryConfig)`, there's no name-less overload and no separate `withTelemetryName(...)` setter. There is no `SwerveDriveConfig.withDataLogName(...)` either, DataLog configuration lives entirely on `SwerveDriveTelemetryConfig`, passed in via `withTelemetry(name, SwerveDriveTelemetryConfig)`. If you already have a `TelemetryVerbosity` in hand and don't need to customize anything else, `new SwerveDriveTelemetryConfig(TelemetryVerbosity.HIGH)` is shorthand for `new SwerveDriveTelemetryConfig().withTelemetryVerbosity(TelemetryVerbosity.HIGH)` (same shorthand constructor exists on `SwerveModuleTelemetryConfig`).

`SwerveModuleTelemetryConfig` follows the same pattern for a single module (`withAbsoluteEncoder()`, `withState()` for the module's `SwerveModuleState`), fed to `SwerveModuleConfig.withTelemetry(name, SwerveModuleTelemetryConfig)`. Note that `withTelemetryVerbosity(...)` only enables `withState()` at `HIGH`, at `MID`/`LOW` only the absolute encoder angle is published. `SwerveModuleConfig` does not have its own `withDataLogName(...)` shorthand, DataLog configuration for a module goes through `SwerveModuleTelemetryConfig.withDataLogName(...)` the same way it does for the drive.

Neither `SwerveDriveTelemetryConfig` nor `SwerveModuleTelemetryConfig` cascades down to the drive/azimuth motors inside a module. If you want granular control over an individual drive or azimuth motor's telemetry (including its own DataLog name), build that motor's `SmartMotorControllerConfig` with `.withTelemetry(name, SmartMotorControllerTelemetryConfig)` exactly like you would for any standalone `SmartMotorController`, see [DataLog Best Practices](../details/datalog-best-practices.md) for a full walkthrough.

If a field doesn't have its own `with*()` method, `withCustom(...)` works the same escape hatch as `SmartMotorControllerTelemetryConfig`, taking a `DoubleTelemetryField`/`StructTelemetryField`/`StructArrayTelemetryField`/`BooleanTelemetryField` (or an array of one) plus a boolean.

### Live-tuning auto-align PID from NetworkTables

`SwerveDriveTelemetryConfig`'s `HIGH` verbosity also publishes a tunable target pose (`autoalign/setpoint/x`/`autoalign/setpoint/y` in meters, `autoalign/setpoint/rot` in degrees, gated by an `autoalign/enabled` boolean) and tunable translation/rotation PID gains under a `Tuning` NetworkTables table. `SwerveDrive` publishes a "Live Tuning"-style command to SmartDashboard at `Mechanisms/<name>/tuning/driveToPose`; running it resets the translation/rotation PID controllers, then every loop it checks `autoalign/enabled` and, while `true`, reads the tunable target pose and P/I/D gains back from NetworkTables and drives the robot toward that pose live, useful for dialing in auto-align gains from a dashboard without redeploying code. `SwerveDrive.setRotationPID(...)`/`setTranslationPID(...)` only reset and replace a PID controller if a gain actually changed, so tuning doesn't wipe the integrator every single loop.

The same `HIGH` verbosity also publishes a shared set of drive-motor (`modules/drive/feedback/p,i,d`, `modules/drive/feedforward/s,v,a`, `modules/drive/velocity`, `modules/drive/enabled`, `modules/drive/inplace`) and azimuth-motor (`modules/azimuth/feedback/p,i,d`, `modules/azimuth/feedforward/s,v,a`, `modules/azimuth/angle`, `modules/azimuth/enabled`) tuning fields, read from the same `Mechanisms/<name>/tuning/driveToPose` command. These apply to **every module on the drive at once** (there's no per-module variant), which is normally what you want since a swerve chassis's modules are built identically. As with the auto-align gains, a `modules/*/feedback/*` PID gain or `modules/*/feedforward/*` feedforward gain only calls `SmartMotorController.setKp(...)`/`setKi(...)`/`setKd(...)`/`setKs(...)`/`setKv(...)`/`setKa(...)` on each module's drive or azimuth motor when that gain's NetworkTables value actually changes, not every loop; the velocity/angle setpoint, by contrast, is re-applied every loop while its `enabled` toggle is `true`, same as the auto-align target pose. Both setpoints are applied via a `SwerveModuleState[]` (one entry per module) passed to `SwerveDrive.setSwerveModuleStates(...)` in a single call, rather than the drive/azimuth `SmartMotorController`s directly: while `modules/drive/inplace` is `false` (the default), drive tuning builds each module's state with the tuned velocity and an angle of `0°`; while `modules/drive/inplace` is `true`, it instead orients each module tangent to its position around the robot center (that module's location angle plus `90°`), so a positive `modules/drive/velocity` spins the whole robot counter-clockwise in place, matching WPILib's positive rotation direction. Azimuth tuning builds every module's state with the tuned angle and `0` speed (so it never also drives). The feedback and feedforward gains are pre-seeded from the first module's configured PID controller and `SimpleMotorFeedforward`, not `0`.

`autoalign/enabled`, `modules/drive/enabled`, and `modules/azimuth/enabled` are mutually exclusive: `ApplyTuningValues` only ever acts on one (priority order auto-align, then drive tuning, then azimuth tuning) and forces the others' NetworkTables value back to `false` if more than one is `true` at once. Turning off `modules/drive/enabled` (while auto-align isn't active) also commands every module's drive motor to `0` m/s every loop, so it doesn't keep coasting at the last tuned velocity. `modules/drive/inplace` only matters while `modules/drive/enabled` is `true`; it doesn't have its own auto-off behavior.

## Colors

You may notice there are different colors in the Mechanism windows displayed for simulation. These are there to provide you with reference points of your Soft and Hard Limits.

* Green is the upper hard limit
* Red is the lower hard limit
* Pink is the upper soft limit
* Yellow is the lower soft limit

These are provided to you so you can identify relative motion easily between the simulation and reality.

## Telemetry Units

All telemetry values in YAMS are logged with specific units. Understanding these units helps you interpret data correctly and configure your visualization tools.

### Standard Telemetry Units

| Telemetry Field     | Unit                         | Description                                           |
| ------------------- | ---------------------------- | ----------------------------------------------------- |
| MechanismPosition   | Rotations (rot)              | Position of the mechanism output shaft                |
| RotorPosition       | Rotations (rot)              | Position of the motor rotor (before gearing)          |
| MechanismVelocity   | Rotations per Second (rot/s) | Velocity of the mechanism output shaft                |
| RotorVelocity       | Rotations per Second (rot/s) | Velocity of the motor rotor                           |
| MechanismLowerLimit | Rotations (rot)              | Lower soft limit position                             |
| MechanismUpperLimit | Rotations (rot)              | Upper soft limit position                             |
| ClosedLoopTarget    | Rotations (rot) or rot/s     | Target position or velocity depending on control mode |
| ClosedLoopError     | Rotations (rot) or rot/s     | Error from target (same unit as target)               |
| AppliedVoltage      | Volts (V)                    | Voltage currently being applied to the motor          |
| SupplyCurrent       | Amps (A)                     | Current drawn from the battery                        |
| StatorCurrent       | Amps (A)                     | Current through the motor windings                    |
| Temperature         | Celsius (°C)                 | Motor controller temperature                          |

### Linear Mechanism Units

When you configure a mechanism with a circumference (using `.withMechanismCircumference()`, or one of the `.withWheelDiameter()`/`.withWheelRadius()`/`.withDrumRadius()` helpers that set it internally), linear telemetry fields become available:

| Telemetry Field  | Unit                    | Description                      |
| ---------------- | ----------------------- | -------------------------------- |
| LinearPosition   | Meters (m)              | Linear position of the mechanism |
| LinearVelocity   | Meters per Second (m/s) | Linear velocity of the mechanism |
| LinearLowerLimit | Meters (m)              | Lower soft limit in linear units |
| LinearUpperLimit | Meters (m)              | Upper soft limit in linear units |

{% hint style="info" %}
YAMS logs rotational units by default. Linear units are calculated from rotational units using your configured circumference: `linear = rotational × circumference`
{% endhint %}

## Using AdvantageScope for Unit Conversion

AdvantageScope provides powerful unit conversion capabilities that let you view telemetry data in whatever units make sense for your analysis, without changing your robot code.

### Changing Display Units

To change how a value is displayed in AdvantageScope:

1. Add the telemetry field to a graph (Line Graph, Discrete, etc.)
2. Right-click on the field in the legend
3. Select **"Convert Units..."**
4. Choose your desired output unit from the list

For example, you can convert:

* Rotations → Degrees, Radians, or custom mechanism units
* Rotations per Second → RPM, Degrees per Second, Radians per Second
* Meters → Inches, Feet, Centimeters
* Meters per Second → Feet per Second, Inches per Second

### Multiple Units on the Same Axis

AdvantageScope allows you to plot values with **different units on the same axis**, which is incredibly useful for comparing related measurements:

**Example: Comparing Position and Velocity** You can plot both `MechanismPosition` (in degrees) and `MechanismVelocity` (in degrees/second) on the same graph to see how velocity changes relative to position during a motion profile.

**Example: Comparing Setpoint vs Actual** Plot `ClosedLoopTarget` and `MechanismPosition` with the same unit conversion to directly compare your target trajectory against actual mechanism movement.

**Example: Analyzing Current Draw** Plot `SupplyCurrent` and `StatorCurrent` together to understand the relationship between battery load and motor torque output.

### Setting Up Unit Conversion in AdvantageScope

1. **Open Line Graph**: Drag fields from the sidebar to a Line Graph tab
2. **Access Unit Settings**: Right-click on any field in the graph legend
3. **Select Convert Units**: Choose the conversion you want
4. **Apply to Multiple Fields**: Each field can have its own conversion, allowing mixed-unit comparisons

{% hint style="success" %}
**Pro Tip**: When tuning a mechanism, convert position to your natural units (degrees for arms, inches for elevators) and plot alongside velocity. This helps you visualize how your motion profile executes and identify where the mechanism accelerates, cruises, and decelerates.
{% endhint %}

### Common Unit Conversions for FRC

| Original Unit | Useful Conversions  | Use Case                             |
| ------------- | ------------------- | ------------------------------------ |
| Rotations     | Degrees, Radians    | Arm angles, turret position          |
| Rotations     | Inches, Centimeters | Elevator height (with circumference) |
| rot/s         | RPM                 | Flywheel speed                       |
| rot/s         | deg/s, rad/s        | Arm angular velocity                 |
| Meters        | Inches, Feet        | Elevator/linear mechanism position   |
| m/s           | ft/s, in/s          | Linear mechanism velocity            |

### Exporting Data with Units

When exporting data from AdvantageScope, the exported values use the **converted units** you have selected in the display. This makes it easy to:

* Share data with team members in familiar units
* Import into spreadsheets for further analysis
* Create documentation with consistent unit conventions
