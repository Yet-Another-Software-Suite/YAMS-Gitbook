---
icon: car-side
---

# Swerve Drive

Every `SwerveDriveConfig` option, with a graphic and an example, is on the [SwerveDriveConfig](mechanism-configs/swerve-drive-config.md) page under this one.

## How a swerve drive works

<figure><img src="../.gitbook/assets/mech-swerve-how.png" alt="A flow diagram. You ask for a ChassisVelocities with vx, vy and omega. The kinematics, built from the module locations, turn it into four module states, a speed and an angle each. Odometry combines wheel travel and the gyro into the robot's pose, which feeds field relative driving, auto-align and paths."><figcaption><p>From one chassis velocity to four wheels, and back to a pose.</p></figcaption></figure>

You tell the drive how the whole robot should move, as a `ChassisVelocities`: forward speed, sideways speed and turn rate. The drive's kinematics, built from where each module is, turn that into a speed and an angle for every wheel. Each [swerve module](swerve-module.md) then steers and drives its own wheel.

Going the other way, the drive adds up how far each wheel has rolled and combines it with the gyro to estimate where the robot is on the field. That pose is what makes field relative driving, auto-align and path following possible.

## Why `SwerveDrive` is different from other mechanisms

Every other mechanism in YAMS (`Arm`, `Elevator`, `Pivot`, `FlyWheel`) wraps a single `SmartMotorController`. `SwerveDrive` is a mechanism made of mechanisms: it owns an array of `SwerveModule`s, and each `SwerveModule` owns two `SmartMotorController`s of its own (drive and azimuth). `SwerveDriveConfig` is where the two levels meet: you build each `SwerveModule` (and its own `SwerveModuleConfig`) first, then hand the finished modules to `SwerveDriveConfig`, which derives the robot's kinematics automatically from where you told each module it physically sits.

## Basic Swerve Config

Given the [tutorial](../tutorials/swerve-drive.md), here is the shape of a four-module config. Modules are ordered clockwise from front-left: FL, FR, BL, BR.

Robot code imports `SmartMotorControllerConfig`, `SwerveDriveConfig`, `SwerveDrive` and `SwerveInputStream` from the command framework package that matches your vendordep, and everything else from `yams.core`:

{% tabs %}
{% tab title="Commands v2" %}
```java
import yams.commands2.config.SmartMotorControllerConfig;
import yams.commands2.config.SwerveDriveConfig;
import yams.commands2.swerve.SwerveDrive;
import yams.commands2.swerve.SwerveInputStream;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.SwerveModuleConfig;
import yams.core.mechanisms.swerve.SwerveModule;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.local.SparkWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;
```
{% endtab %}

{% tab title="Commands v3" %}
```java
import yams.commands3.config.SmartMotorControllerConfig;
import yams.commands3.config.SwerveDriveConfig;
import yams.commands3.swerve.SwerveDrive;
import yams.commands3.swerve.SwerveInputStream;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.SwerveModuleConfig;
import yams.core.mechanisms.swerve.SwerveModule;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.local.SparkWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;
```

In Commands v3, `this` is your `Mechanism` instead of a `Subsystem`.
{% endtab %}
{% endtabs %}

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
      .withGyro(gyro::getRotation3d) // e.g. a Pigeon2 or the SystemCore's OnboardIMU
      .withMaximumChassisSpeed(MetersPerSecond.of(4.5), DegreesPerSecond.of(360))
      .withTranslationController(new PIDController(1.0, 0, 0))
      .withRotationController(new PIDController(1.0, 0, 0))
      .withStartingPose(new Pose2d());

SwerveDrive drive = new SwerveDrive(config);
```

## Commands

`SwerveDrive` has the same commands in Commands v2 and Commands v3, and Commands v3 adds a few more.

| Command or function | Commands v2 | Commands v3 | What it does |
| --- | --- | --- | --- |
| `drive(Supplier<ChassisVelocities>)` | ✅ | ✅ | Drives with **robot relative** speeds until interrupted. |
| `driveToPose(Pose2d)` | ✅ | ✅ | Drives to a pose with the [auto-align controllers](mechanism-configs/swerve-drive-config.md#auto-align-controllers) until interrupted. |
| `driveToPose(Pose2d, Distance, Angle)` | ❌ | ✅ | The same, ending once the robot is within the tolerances. |
| `isNear(Pose2d, Distance, Angle)` | ❌ | ✅ | Whether the robot is within the tolerances of a pose. |
| `stop()` | ❌ | ✅ | Commands zero speed. |

In Commands v2, use `getDistanceFromPose` and `getAngleDifferenceFromPose` to decide when a `driveToPose` has arrived.

Field relative driving from a controller is a command you write around a [`SwerveInputStream`](swerve-input-stream.md):

{% tabs %}
{% tab title="Commands v2" %}
```java
public class SwerveSubsystem extends SubsystemBase
{
  private final SwerveDrive drive = new SwerveDrive(config);

  public Command driveWith(SwerveInputStream input)
  {
    return run(() -> drive.setFieldRelativeChassisSpeeds(input.get()));
  }

  @Override
  public void periodic()
  {
    drive.updateTelemetry(); // also updates the pose estimate
  }

  @Override
  public void simulationPeriodic()
  {
    drive.simIterate();
  }
}
```
{% endtab %}

{% tab title="Commands v3" %}
```java
public class SwerveMechanism implements Mechanism
{
  private final SwerveDrive drive = new SwerveDrive(config);

  public Command driveWith(SwerveInputStream input)
  {
    return run(coroutine -> {
      while (true)
      {
        drive.setFieldRelativeChassisSpeeds(input.get());
        coroutine.yield();
      }
    }).named("Drive");
  }

  public void periodic() // call from Robot.robotPeriodic()
  {
    drive.updateTelemetry(); // also updates the pose estimate
  }

  public void simulationPeriodic() // call from Robot.robotPeriodic() in simulation
  {
    drive.simIterate();
  }
}
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
`updateTelemetry()` is what updates the pose estimate, so it must be called every loop even if you do not look at the telemetry.
{% endhint %}

## Driving functions

These set what the drive does and return at once. They are what the commands above are built on.

| Function | What it does |
| --- | --- |
| `setFieldRelativeChassisSpeeds(ChassisVelocities)` | Drive with speeds measured against the field. This is what a `SwerveInputStream` gives you. |
| `setRobotRelativeChassisSpeeds(ChassisVelocities)` | Drive with speeds measured against the robot. An overload adds a feedforward force per module, for path followers. |
| `setSwerveModuleStates(SwerveModuleVelocity[])` | Set every module directly, skipping the kinematics. |
| `lockPose()` | Turn the wheels into an X so the robot is hard to push. |
| `resetOdometry(Pose2d)` | Tell the drive where it is, and set the gyro to that heading. |
| `zeroGyro()` | Make the direction the robot faces now the new zero heading. |
| `addVisionMeasurement(Pose2d, double)` | Correct the pose estimate with a vision measurement. |

## Asking where it is

| Function | Returns |
| --- | --- |
| `getPose()` | The estimated position and heading on the field. |
| `getRobotRelativeSpeed()`, `getFieldRelativeSpeed()` | How the robot is moving now. |
| `getModuleStates()`, `getModulePositions()` | Each wheel's speed and angle, and how far it has rolled. |
| `getGyroRotation3d()` | The robot's attitude from the gyro. |
| `getDistanceFromPose(Pose2d)`, `getAngleDifferenceFromPose(Pose2d)` | How far the robot is from a pose. |
| `getField2d()` | The `Field2d` the drive publishes its pose to. Use this one instead of making your own. |

## Configuring and tuning a swerve drive, step by step

Work from the wheels outward: a drive cannot be tuned until its modules are.

{% stepper %}
{% step %}
#### Configure each module

Set each module's two motors, its location, its absolute encoder and offset: see [Swerve Module](swerve-module.md#configuring-and-tuning-a-swerve-module-step-by-step).
{% endstep %}

{% step %}
#### Check the module order and locations

With the robot on blocks, drive forward slowly. Every wheel should point forward and spin forward. Then spin in place: the wheels should form a circle. A wheel pointing the wrong way means a wrong [location](mechanism-configs/swerve-module-config.md#withlocation) or [encoder offset](mechanism-configs/swerve-module-config.md#the-absolute-encoder).
{% endstep %}

{% step %}
#### Check the gyro direction

Turn the robot counter-clockwise by hand and watch the heading. It must go up. If it goes down, set [`withGyroInverted`](mechanism-configs/swerve-drive-config.md#the-gyro).
{% endstep %}

{% step %}
#### Tune the modules

Tune the azimuth motors, then the drive motors, with the swerve tuning command: see [Tune the swerve modules](live-tuning/README.md#tune-the-swerve-modules).
{% endstep %}

{% step %}
#### Set the speed limits

Measure the real top speed by driving flat out, and set [`withMaximumModuleSpeed` and `withMaximumChassisSpeed`](mechanism-configs/swerve-drive-config.md#speed-limits) a little below it.
{% endstep %}

{% step %}
#### Correct the drift while spinning

Drive in a straight line while spinning. If the path bends, set [`withDiscretizationTime`](mechanism-configs/swerve-drive-config.md#withdiscretizationtime) to the loop time, then adjust [`withGyroAngularVelocityScaleFactor`](mechanism-configs/swerve-drive-config.md#withgyroangularvelocityscalefactor) if some remains.
{% endstep %}

{% step %}
#### Tune auto-align last

Tune the translation and rotation controllers once the robot drives well by hand: see [Tune auto-align](live-tuning/README.md#tune-auto-align).
{% endstep %}
{% endstepper %}

## Module Location Drives the Kinematics

`SwerveModuleConfig.withLocation(front, left)` (or `withDistanceFromCenterOfRotation`) is not just bookkeeping. `SwerveDrive` builds its `SwerveDriveKinematics` directly from the locations of every module you pass it, so the drive/rotate math is always correct for your specific chassis dimensions without you ever constructing a `SwerveDriveKinematics` yourself.

```java
SwerveModuleConfig moduleConfig = new SwerveModuleConfig(driveSMC, azimuthSMC)
      .withLocation(Meters.of(0.3), Meters.of(0.3)); // X = forward, Y = left, from robot center
```

`withDistanceFromCenterOfRotation` is an equivalent alternate form:

```java
moduleConfig.withDistanceFromCenterOfRotation(Meters.of(0.3), Meters.of(0.3));
```

{% hint style="warning" %}
Every module needs its location set before it's handed to `SwerveDriveConfig`. A missing or wrong location silently produces incorrect kinematics rather than a compile error, so double check trackwidth/wheelbase math when a robot doesn't drive straight.
{% endhint %}

The `withOptimization(true)` seen in the config above is a per-module flag, not a drive-level one. See [Optimization and Cosine Compensation](swerve-module.md#optimization-and-cosine-compensation) on the Swerve Module page for what it (and its counterpart `withCosineCompensation`) actually does.

## Gyro Integration and Field vs. Robot Relative Speeds

`SwerveDriveConfig.withGyro(Supplier<Rotation3d>)` is what lets `SwerveDrive` convert between field-relative and robot-relative `ChassisVelocities` (WPILib 2027's name for `ChassisSpeeds`). The supplier returns the gyro's full attitude, such as `pigeon::getRotation3d` or `imu::getRotation3d`: its yaw is the robot's heading, and its roll and pitch are the robot's tilt. Internally, `SwerveDrive` always drives the modules with robot-relative speeds; `setFieldRelativeChassisSpeeds` just rotates the requested speeds into the robot frame using the current heading before handing them to the same code path robot-relative callers use.

```java
drive.setFieldRelativeChassisSpeeds(new ChassisVelocities(1.0, 0, 0));
```

`SwerveDriveConfig.withGyroOffset` and `withGyroInverted` exist so the gyro's raw zero and sign can be corrected without touching your drive code. Both apply to the heading (the yaw):

```java
config.withGyroOffset(Degrees.of(90))
      .withGyroInverted(true);
```

### Gyros Mounted on Their Side

If the gyro is not mounted flat, the robot turning shows up as the gyro's roll or pitch instead of its yaw. `withGyroHeadingAxis(GyroAxis)` tells `SwerveDriveConfig` which gyro axis points up through the robot. `GyroAxis` lives in `yams.core.mechanisms.config.enums` and has three values:

| `GyroAxis` | Gyro axis pointing up | Use when                                                  |
| ---------- | --------------------- | --------------------------------------------------------- |
| `YAW`      | Z                     | The gyro is mounted flat. This is the default.            |
| `ROLL`     | X                     | The gyro is on its side, with its Y axis to robot left.   |
| `PITCH`    | Y                     | The gyro is on its side, with its X axis to robot front.  |

```java
import yams.core.mechanisms.config.enums.GyroAxis;

config.withGyro(pigeon::getRotation3d)
      .withGyroHeadingAxis(GyroAxis.ROLL);
```

The gyro's attitude is rotated into the robot's frame first, then `withGyroInverted` and `withGyroOffset` are applied to the resulting heading.

### Reading the Heading

`SwerveDrive.getGyroRotation3d()` (and `SwerveDriveConfig.getGyroRotation3d()`) returns the robot's attitude after the heading axis, inversion and offset are applied. Its yaw (`getZ()`, or `toRotation2d()`) is the heading, wrapped to +/-180 degrees, and its roll and pitch are the robot's tilt. In simulation, `SwerveDrive.getGyroRotation3d()` returns the simulated heading with no tilt.

```java
Rotation2d heading = drive.getGyroRotation3d().toRotation2d();
```

{% hint style="warning" %}
**Breaking change:** `getGyroAngle()` was removed from the Java `SwerveDriveConfig` and `SwerveDrive`. Use `getGyroRotation3d()` instead; its heading wraps at +/-180 degrees. `GyroAxis`, `withGyroHeadingAxis` and `getGyroRotation3d` are Java only: C++ still takes a heading with `WithGyro(std::function<wpi::units::degree_t()>)` and reads it back with `GetGyroAngle()`.
{% endhint %}

`withGyroVelocity`/`withGyroAngularVelocityScaleFactor` let you feed the gyro's own angular velocity measurement into pose estimation instead of relying solely on module odometry, which is usually more accurate during fast rotation:

```java
config.withGyroVelocity(() -> DegreesPerSecond.of(gyro.getRate()))
      .withGyroAngularVelocityScaleFactor(1.0);
```

{% hint style="info" %}
`SwerveInputStream` is the recommended way to turn raw joystick axes into a field-relative `ChassisVelocities` supplier; it handles deadbanding, axis cubing, and alliance-relative flipping so you don't have to write that math by hand. See [Swerve Input Stream](swerve-input-stream.md) for the full picture, or the [tutorial](../tutorials/swerve-drive.md) for a worked example.
{% endhint %}

```java
SwerveInputStream input =
    SwerveInputStream.of(drive, () -> -controller.getLeftY(), () -> -controller.getLeftX())
        .withControllerRotationAxis(() -> -controller.getRightX());
// SwerveInputStream output is field relative.
setDefaultCommand(run(() -> drive.setFieldRelativeChassisSpeeds(input.get())));
```

## Locking the Wheels (X Pattern)

`SwerveDrive.lockPose()` turns every wheel to point at the robot center (each module's angle from its `withLocation(...)`) without driving it. On a rectangular chassis the four wheels form an X, so the robot resists being pushed. Call it every loop for as long as the lock should hold:

{% tabs %}
{% tab title="Commands v2" %}
```java
public Command lockPose() {
  return run(drive::lockPose).withName("Lock Pose");
}
```
{% endtab %}

{% tab title="Commands v3" %}
```java
public Command lock() {
  return run(coroutine -> {
    while (true) {
      drive.lockPose();
      coroutine.yield();
    }
  }).named("Swerve Lock");
}
```
{% endtab %}
{% endtabs %}

In Java each module is locked through `SwerveModule.lock(Rotation2d)`, which uses `SwerveModuleConfig.getLockedState(...)`. Unlike a normal zero-speed module state, the lock skips the module's [velocity deadband](swerve-module.md#velocity-deadband), which would otherwise hold the wheels where they are, so the wheels actually turn into the X. With optimization enabled each wheel turns to whichever of the lock angle and its opposite is nearer, since both hold the robot the same way. C++ `LockPose()` commands the same X pattern.

{% hint style="info" %}
`lockPose()` is not compatible with AdvantageKit, like the other `SwerveDrive` methods that command the modules directly.
{% endhint %}

## PathPlanner Integration and Feedforward Forces

`setRobotRelativeChassisSpeeds` has an overload that accepts a per-module drive-wheel feedforward `Force[]` (FL, FR, BL, BR order, matching module order) alongside the `ChassisVelocities`:

```java
drive.setRobotRelativeChassisSpeeds(speeds, feedforwardForces);
```

This exists specifically to plug into PathPlanner's `AutoBuilder.configure(...)`, whose `driveRobotRelative` consumer hands back a `DriveFeedforwards` alongside the requested `ChassisVelocities`. Its `linearForces()` is exactly the `Force[]` this overload expects:

```java
AutoBuilder.configure(
    drive::getPose,
    drive::resetOdometry,
    drive::getRobotRelativeSpeed,
    (speeds, feedforwards) -> drive.setRobotRelativeChassisSpeeds(speeds, feedforwards.linearForces()),
    new PPHolonomicDriveController(new PIDConstants(5.0, 0.0, 0.0), new PIDConstants(5.0, 0.0, 0.0)),
    RobotConfig.fromGUISettings(),
    () -> MatchState.getAlliance().map(alliance -> alliance == Alliance.RED).orElse(false),
    this);
```

Each module's drive `SmartMotorController` converts its share of the `Force[]` into the correct feedforward for whatever control mode it's using (voltage or torque-current) via `SmartMotorControllerConfig.convertToVoltage(...)`/`convertToCurrent(...)`, which need `withGearing(...)` and `withMechanismCircumference(...)`/`withWheelRadius(...)` configured on that motor to do the conversion. Passing an empty `Force[]` (or just calling the original single-argument overload) skips the feedforward entirely, so existing code that only calls `setRobotRelativeChassisSpeeds(speeds)` keeps working unchanged.

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/BroncBotz_2026/java/first/robot/subsystems/Swerve.java#L185-L200" %}

{% hint style="info" %}
`SmartMotorController.setVelocity(velocity, force)` is the underlying per-motor API this builds on, and it isn't specific to swerve: any velocity-controlled `SmartMotorController` with gearing and a mechanism circumference configured can take a feedforward `Force` the same way. A `null`/unset force falls back to plain velocity control, which is what `getSetpointFeedforwardForce()` and the `SetpointForce` telemetry field then read back as empty/`0`.
{% endhint %}

## Auto-Align

`SwerveDrive` has a built-in field relative auto-align.

| Method | What it does |
| --- | --- |
| `driveToPose(Pose2d)` | Returns a `Command` that drives to the pose until it is interrupted. |
| `driveToPoseSetpoint(Pose2d)` | Computes the `ChassisVelocities` that drive toward the pose, for use in your own command. |

Both use the translation and rotation `PIDController`s from `SwerveDriveConfig.withTranslationController(...)` and `withRotationController(...)`. The command is a Commands v2 or Commands v3 `Command`, depending on which `SwerveDrive` you use.

```java
driver.a().whileTrue(drive.driveToPose(new Pose2d(2, 4, Rotation2d.ZERO)));
```

{% hint style="success" %}
The auto-align gains, and the drive and azimuth motor gains, can be tuned from a dashboard without redeploying. See [Live tuning: Swerve drive](live-tuning/README.md#swerve-drive).
{% endhint %}

## Telemetry Verbosity

Like every other mechanism, `SwerveDriveConfig.withTelemetry(name, TelemetryVerbosity)` picks a preset bundle of fields to publish, from pose and gyro angle at `LOW` up to full desired/measured chassis speeds and module states at `HIGH`. Each level includes everything the level below it publishes.

```java
config.withTelemetry("swerve", TelemetryVerbosity.HIGH);
```

| Field                                      | NetworkTables key                   | `LOW` | `MID` | `HIGH` |
| ------------------------------------------ | ----------------------------------- | :---: | :---: | :----: |
| Pose                                       | `pose`                              |   ✓   |   ✓   |    ✓   |
| Gyro angle                                 | `gyro`                              |   ✓   |   ✓   |    ✓   |
| Current robot-relative ChassisVelocities   | `chassis/current`                   |       |   ✓   |    ✓   |
| Field-relative ChassisVelocities           | `chassis/field`                     |       |   ✓   |    ✓   |
| Current module states                      | `states/current`                    |       |   ✓   |    ✓   |
| Desired robot-relative ChassisVelocities   | `chassis/desired`                   |       |       |    ✓   |
| Desired module states                      | `states/desired`                    |       |       |    ✓   |
| Translation P/I/D (tunable)                | `autoalign/translation/p,i,d`       |       |       |    ✓   |
| Rotation P/I/D (tunable)                   | `autoalign/rotation/p,i,d`          |       |       |    ✓   |
| Auto-align target X (tunable)              | `autoalign/setpoint/x`              |       |       |    ✓   |
| Auto-align target Y (tunable)              | `autoalign/setpoint/y`              |       |       |    ✓   |
| Auto-align target rotation (tunable)       | `autoalign/setpoint/rot`            |       |       |    ✓   |
| Auto-align enabled (tunable)               | `autoalign/enabled`                 |       |       |    ✓   |
| Modules drive P/I/D (tunable, shared)      | `modules/drive/feedback/p,i,d`      |       |       |    ✓   |
| Modules drive kS/kV/kA (tunable, shared)   | `modules/drive/feedforward/s,v,a`   |       |       |    ✓   |
| Modules drive velocity (tunable, shared)   | `modules/drive/velocity`            |       |       |    ✓   |
| Modules drive tuning enabled (tunable)     | `modules/drive/enabled`             |       |       |    ✓   |
| Modules drive in place (tunable)           | `modules/drive/inplace`             |       |       |    ✓   |
| Modules azimuth P/I/D (tunable, shared)    | `modules/azimuth/feedback/p,i,d`    |       |       |    ✓   |
| Modules azimuth kS/kV/kA (tunable, shared) | `modules/azimuth/feedforward/s,v,a` |       |       |    ✓   |
| Modules azimuth angle (tunable, shared)    | `modules/azimuth/angle`             |       |       |    ✓   |
| Modules azimuth tuning enabled (tunable)   | `modules/azimuth/enabled`           |       |       |    ✓   |

{% hint style="info" %}
`autoalign/setpoint/x`/`autoalign/setpoint/y`/`autoalign/setpoint/rot` and `autoalign/enabled`, and the `modules/drive/*`/`modules/azimuth/*` tuning fields, only take effect while the `Mechanisms/<name>/tuning` command is scheduled, since that's the loop that actually reads them. Publishing values to them with the tuning command idle has no effect on the robot.
{% endhint %}

{% hint style="warning" %}
The `modules/drive/*` and `modules/azimuth/*` fields are **shared across every module**, not per-module. There is no way to tune a single module's PID or setpoint independently through this mechanism; it applies the same gains and setpoint to all modules on the drive at once.
{% endhint %}

### Using `SwerveDriveTelemetryConfig` Directly

Pass a `SwerveDriveTelemetryConfig` instead of a verbosity level when you need finer control than a preset gives you, such as disabling the auto-align tuning fields, or routing telemetry to a DataLog instead of NetworkTables during competition:

```java
SwerveDriveTelemetryConfig telemetryCfg =
    new SwerveDriveTelemetryConfig(TelemetryVerbosity.HIGH)
        // HIGH enables this by default; opt back out if you don't want live driveToPose tuning exposed.
        .withCustom(SwerveDriveTelemetry.BooleanTelemetryField.AutoAlignEnabled, false)
        .withoutNetworkTables()
        .withDataLogName("swerve");

SwerveDriveConfig config = new SwerveDriveConfig(this, fl, fr, bl, br)
      .withGyro(gyro::getRotation3d)
      .withTranslationController(new PIDController(1.0, 0, 0))
      .withRotationController(new PIDController(1.0, 0, 0))
      .withTelemetry("swerve", telemetryCfg);
```

`SwerveDriveTelemetryConfig(TelemetryVerbosity)` is a shorthand constructor equivalent to `new SwerveDriveTelemetryConfig().withTelemetryVerbosity(verbosity)`, useful when you're about to layer more `with*()` calls on top anyway. See [Telemetry](../understanding/telemetry.md) for the shared telemetry architecture these configs build on.

## Code Reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/SwerveSubsystem.java" %}
