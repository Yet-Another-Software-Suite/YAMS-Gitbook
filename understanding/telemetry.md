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

<figure><img src="../.gitbook/assets/127.0.0.1, AdvantageScope 9_2_2025 1_06_09 PM.png" alt=""><figcaption></figcaption></figure>

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

<figure><img src="../.gitbook/assets/127.0.0.1, AdvantageScope 9_2_2025 1_03_52 PM.png" alt=""><figcaption></figcaption></figure>

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

## Swerve Drive Telemetry

`SwerveDrive` and `SwerveModule` do **not** use `SmartMotorControllerTelemetryConfig` to pick individual fields, and there is no `withoutNetworkTables()` equivalent for a swerve drive, NetworkTables publishing for the drive-level fields (pose, gyro, chassis speeds, module states) and each module's fields (drive/azimuth motor telemetry, absolute encoder) always happens. Instead you choose a `TelemetryVerbosity` (`LOW`/`MID`/`HIGH`) with `SwerveDriveConfig.withTelemetry(...)` and `SwerveModuleConfig.withTelemetry(name, ...)`, and YAMS logs everything useful for that verbosity level automatically.

You can still log to DataLog:

* `SwerveDriveConfig.withDataLogName(name)` logs the drive's pose, gyro, and chassis speeds/module states.
* `SwerveModuleConfig.withDataLogName(name)` logs that module's absolute encoder reading.

Neither of these cascades down to the drive/azimuth motors of a module. If you want granular control over an individual drive or azimuth motor's telemetry (including its own DataLog name), build that motor's `SmartMotorControllerConfig` with `.withTelemetry(name, SmartMotorControllerTelemetryConfig)` exactly like you would for any standalone `SmartMotorController`, see [DataLog Best Practices](../details/datalog-best-practices.md) for a full walkthrough.

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
