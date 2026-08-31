---
icon: car-side
---

# Swerve Drive

## Why `SwerveDrive` is different from other mechanisms

Every other mechanism in YAMS (`Arm`, `Elevator`, `Pivot`, `FlyWheel`) wraps a single `SmartMotorController`. `SwerveDrive` is a mechanism made of mechanisms: it owns an array of `SwerveModule`s, and each `SwerveModule` owns two `SmartMotorController`s of its own (drive and azimuth). `SwerveDriveConfig` is where the two levels meet: you build each `SwerveModule` (and its own `SwerveModuleConfig`) first, then hand the finished modules to `SwerveDriveConfig`, which derives the robot's kinematics automatically from where you told each module it physically sits.

## Basic Swerve Config

Given the [tutorial](../tutorials/swerve-drive.md), here is the shape of a four-module config. Modules are ordered clockwise from front-left: FL, FR, BL, BR.

```java
SmartMotorControllerConfig driveCfg = new SmartMotorControllerConfig(this)
      .withWheelDiameter(Inches.of(4))
      .withGearing(new MechanismGearing(6.75))
      .withClosedLoopController(0.3, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0, 1.9, 0.01))
      .withStatorCurrentLimit(Amps.of(40))
      .withTelemetry("driveMotor", TelemetryVerbosity.HIGH);

SmartMotorControllerConfig azimuthCfg = new SmartMotorControllerConfig(this)
      .withGearing(new MechanismGearing(12.8))
      .withClosedLoopController(1.0, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0, 1))
      .withStatorCurrentLimit(Amps.of(20))
      .withTelemetry("angleMotor", TelemetryVerbosity.HIGH);

SmartMotorController driveSMC   = new SparkWrapper(driveMotor,   DCMotor.getNEO(1), driveCfg);
SmartMotorController azimuthSMC = new SparkWrapper(azimuthMotor, DCMotor.getNEO(1), azimuthCfg);

SwerveModuleConfig moduleConfig = new SwerveModuleConfig(driveSMC, azimuthSMC)
      .withAbsoluteEncoder(encoder.getAbsolutePosition().asSupplier())
      .withLocation(Meters.of(0.3), Meters.of(0.3)) // front, left, from robot center
      .withOptimization(true)
      .withTelemetry("frontleft", TelemetryVerbosity.HIGH);

SwerveModule fl = new SwerveModule(moduleConfig);
// ...repeat for fr, bl, br with their own locations...

SwerveDriveConfig config = new SwerveDriveConfig(this, fl, fr, bl, br)
      .withGyro(gyro.getYaw().asSupplier())
      .withMaximumChassisSpeed(MetersPerSecond.of(4.5), DegreesPerSecond.of(360))
      .withTranslationController(new PIDController(1.0, 0, 0))
      .withRotationController(new PIDController(1.0, 0, 0))
      .withStartingPose(new Pose2d());

SwerveDrive drive = new SwerveDrive(config);
```

## Module Location Drives the Kinematics

`SwerveModuleConfig.withLocation(front, left)` (or `withDistanceFromCenterOfRotation`) is not just bookkeeping. `SwerveDrive` builds its `SwerveDriveKinematics` directly from the locations of every module you pass it, so the drive/rotate math is always correct for your specific chassis dimensions without you ever constructing a `SwerveDriveKinematics` yourself.

```java
SwerveModuleConfig moduleConfig = new SwerveModuleConfig(driveSMC, azimuthSMC)
      .withLocation(Meters.of(0.3), Meters.of(0.3)); // X = forward, Y = left, from robot center
```

{% hint style="warning" %}
Every module needs its location set before it's handed to `SwerveDriveConfig`. A missing or wrong location silently produces incorrect kinematics rather than a compile error, so double check trackwidth/wheelbase math when a robot doesn't drive straight.
{% endhint %}

## Cosine Compensation and Optimization

Two `SwerveModuleConfig` flags exist purely to make modules track setpoints more efficiently:

* `withOptimization(true)` calls WPILib's `SwerveModuleState.optimize`, so a module never rotates more than 90 degrees to reach a target heading (it will reverse the drive direction instead of spinning the long way around).
* `withCosineCompensation(true)` scales the commanded drive velocity by the cosine of the angle error between the module's current heading and its desired heading, so a module that hasn't finished rotating yet doesn't also try to drive at full speed in the wrong direction.

Both are safe to leave enabled for essentially every swerve module; they exist as flags mainly so you can disable them while debugging module behavior in isolation.

## Gyro Integration and Field vs. Robot Relative Speeds

`SwerveDriveConfig.withGyro(Supplier<Angle>)` is what lets `SwerveDrive` convert between field-relative and robot-relative `ChassisSpeeds`. Internally, `SwerveDrive` always drives the modules with robot-relative speeds; `setFieldRelativeChassisSpeeds` just rotates the requested speeds into the robot frame using the current gyro angle before handing them to the same code path robot-relative callers use.

`SwerveDriveConfig.withGyroOffset` and `withGyroInverted` exist so the gyro's raw zero and sign can be corrected without touching your drive code, and `withGyroVelocity`/`withGyroAngularVelocityScaleFactor` let you feed the gyro's own angular velocity measurement into pose estimation instead of relying solely on module odometry, which is usually more accurate during fast rotation.

{% hint style="info" %}
`SwerveInputStream` is the recommended way to turn raw joystick axes into a `ChassisSpeeds` supplier for `drive(...)`; it handles deadbanding, axis cubing, and alliance-relative flipping so you don't have to write that math by hand. See the [tutorial](../tutorials/swerve-drive.md) for a worked example.
{% endhint %}

## Auto-Align: `driveToPoseSetpoint` and Live PID Tuning

`SwerveDrive` includes a built-in field-relative auto-align: `driveToPoseSetpoint(Pose2d targetPose)` uses the translation and rotation `PIDController`s from `SwerveDriveConfig` to compute the `ChassisSpeeds` needed to drive toward a target pose, and `driveToPose(Pose2d)` wraps that into a ready-to-schedule `Command`.

What makes this different from wiring up the PID controllers yourself is the telemetry integration: `SwerveDriveTelemetryConfig` publishes a live-tunable target pose and the translation/rotation PID gains to NetworkTables, and `SwerveDrive` publishes a matching tuning command to SmartDashboard (`Mechanisms/<name>/tuning/driveToPose`). Running that command from the dashboard repeatedly reads the tunable pose and gains and drives the real (or simulated) robot toward them, so you can tune auto-align PID gains live without redeploying code. Changing a gain only resets that controller's integrator if the gain actually changed, so tuning doesn't wipe accumulated state every loop.

## Telemetry Verbosity

Like every other mechanism, `SwerveDriveConfig.withTelemetry(name, TelemetryVerbosity)` picks a preset bundle of fields to publish, from pose and gyro angle at `LOW` up to full desired/measured chassis speeds and module states at `HIGH`. Each level includes everything the level below it publishes.

| Field                                 | NetworkTables key           | `LOW` | `MID` | `HIGH` |
| -------------------------------------- | ---------------------------- | :---: | :---: | :----: |
| Pose                                   | `pose`                        |   ✓   |   ✓   |    ✓   |
| Gyro angle                             | `gyro`                        |   ✓   |   ✓   |    ✓   |
| Current robot-relative ChassisSpeeds   | `chassis/current`             |       |   ✓   |    ✓   |
| Field-relative ChassisSpeeds           | `chassis/field`               |       |   ✓   |    ✓   |
| Current module states                  | `states/current`              |       |   ✓   |    ✓   |
| Desired robot-relative ChassisSpeeds   | `chassis/desired`             |       |       |    ✓   |
| Desired module states                  | `states/desired`              |       |       |    ✓   |
| Translation P/I/D (tunable)            | `autoalign/translation/p,i,d` |       |       |    ✓   |
| Rotation P/I/D (tunable)               | `autoalign/rotation/p,i,d`    |       |       |    ✓   |
| Target pose (tunable)                  | `tuning/driveToPose`          |       |       |        |

{% hint style="warning" %}
Target pose is never enabled by a verbosity preset, at any level. Even at `HIGH`, the auto-align PID gains become live-tunable but the target pose that `driveToPose` tuning drives toward does not; you must opt in explicitly with `withCustom(SwerveDriveTelemetry.StructTelemetryField.TargetPose, true)`. Without it, `SwerveDrive` never subscribes to that NetworkTables entry, so publishing a pose there has no effect on the robot.
{% endhint %}

### Using `SwerveDriveTelemetryConfig` Directly

Pass a `SwerveDriveTelemetryConfig` instead of a verbosity level when you need finer control than a preset gives you, such as enabling the target pose for live tuning, or routing telemetry to a DataLog instead of NetworkTables during competition:

```java
SwerveDriveTelemetryConfig telemetryCfg =
    new SwerveDriveTelemetryConfig(TelemetryVerbosity.HIGH)
        // HIGH doesn't include this; opt in explicitly to enable live driveToPose tuning.
        .withCustom(SwerveDriveTelemetry.StructTelemetryField.TargetPose, true)
        .withoutNetworkTables()
        .withDataLogName("swerve");

SwerveDriveConfig config = new SwerveDriveConfig(this, fl, fr, bl, br)
      .withGyro(gyro.getYaw().asSupplier())
      .withTranslationController(new PIDController(1.0, 0, 0))
      .withRotationController(new PIDController(1.0, 0, 0))
      .withTelemetry("swerve", telemetryCfg);
```

`SwerveDriveTelemetryConfig(TelemetryVerbosity)` is a shorthand constructor equivalent to `new SwerveDriveTelemetryConfig().withTelemetryVerbosity(verbosity)`, useful when you're about to layer more `with*()` calls on top anyway. See [Telemetry](../understanding/telemetry.md) for the shared telemetry architecture these configs build on.

## Code Reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/swerve_drive/java/frc/robot/subsystems/SwerveSubsystem.java" %}
