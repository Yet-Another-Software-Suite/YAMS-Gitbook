---
icon: car-side
---

# Swerve Drive

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

## Auto-Align: `driveToPoseSetpoint` and Live PID Tuning

`SwerveDrive` includes a built-in field-relative auto-align: `driveToPoseSetpoint(Pose2d targetPose)` uses the translation and rotation `PIDController`s from `SwerveDriveConfig` to compute the `ChassisVelocities` needed to drive toward a target pose, and `driveToPose(Pose2d)` wraps that into a ready-to-schedule `Command` (a Commands v2 or Commands v3 `Command`, depending on which `SwerveDrive` you use) that runs until it is interrupted.

```java
driver.a().whileTrue(drive.driveToPose(new Pose2d(2, 4, Rotation2d.ZERO)));
```

What makes this different from wiring up the PID controllers yourself is the telemetry integration: `SwerveDriveTelemetryConfig` publishes a live-tunable target pose (as separate `autoalign/setpoint/x`, `autoalign/setpoint/y`, `autoalign/setpoint/rot` entries, gated by an `autoalign/enabled` toggle) and the translation/rotation PID gains to NetworkTables, and `SwerveDrive` publishes a matching tuning command at `Mechanisms/<name>/tuning` (only when both the translation and rotation controllers are configured). In Commands v3 the tuning command is published through `CommandTunable`. Running that command from the dashboard reads `autoalign/enabled` every loop; while it's `true`, it also reads the tunable pose fields and PID gains and drives the real (or simulated) robot toward them, so you can tune auto-align PID gains (or drive to an arbitrary pose) live without redeploying code. Changing a gain only resets that controller's integrator if the gain actually changed, so tuning doesn't wipe accumulated state every loop.

### How to Use Live Tuning

1. Make sure the fields you need are enabled.
   1. &#x20;`TranslationP/I/D`, `RotationP/I/D`, `autoalign/setpoint`, and `autoalign/enabled` are all already live-tunable at `HIGH` verbosity, so no extra opt-in is needed.
2. Deploy the robot code, then open NetworkTables in AdvantageScope, Shuffleboard, Elastic, or a similar dashboard tool and browse to `Tuning/<name>/` (the name you passed to `withTelemetry`). You should see `autoalign/translation/p`, `i`, `d`, `autoalign/rotation/p`, `i`, `d`, `autoalign/setpoint`, and `autoalign/enabled`. The gain entries start pre-seeded with whatever you passed to `withTranslationController(...)`/`withRotationController(...)`, not `0`, the same way `SmartMotorControllerTelemetry` seeds its `kP`/`kI`/`kD` tuning entries from the motor's configured PID.
3. On Elastic, find the command at `Mechanisms/<name>/tuning` and schedule it, either by binding it to a button or running it directly from your dashboard's command widget. Starting it resets both PID controllers; it then checks `autoalign/enabled` every loop and, while `true`, calls `driveToPoseSetpoint(...)` using whatever `autoalign/setpoint/x`/`autoalign/setpoint/y`/`autoalign/setpoint/rot` and gains currently sit in `Tuning/<name>/`.

<figure><img src="../.gitbook/assets/autoalign-tuning-setup.png" alt=""><figcaption><p>AdvantageScope 2D field with the <code>driveToPose</code> command scheduled and the <code>Tuning/swerve/autoalign</code> tree expanded, showing <code>enabled</code>, <code>pose/x,y,rot</code>, and the translation/rotation gain entries.</p></figcaption></figure>

4. With the tuning command running, edit `autoalign/translation/p` (and the other gain entries) live from your dashboard. Set `autoalign/setpoint/x`/`autoalign/setpoint/y` (meters) and `autoalign/setpoint/rot` (degrees) to the field-relative pose you want, then flip `autoalign/enabled` to `true`. The robot immediately drives toward the target using the current gains, no redeploy needed.

<figure><img src="../.gitbook/assets/autoalign-tuning.gif" alt=""><figcaption><p>Live tuning in simulation: the robot drives from its current pose to a target of <code>autoalign/setpoint/x=12</code>, <code>autoalign/setpoint/y=4</code>, <code>autoalign/setpoint/rot=90</code> while <code>autoalign/enabled</code> is <code>true</code>.</p></figcaption></figure>

5. Once a gain feels right, copy its value back into the matching `SwerveDriveConfig.withTranslationController(...)` / `withRotationController(...)` call in code. Values edited live in NetworkTables are not persisted and reset the next time the robot code restarts.
6. Flip `autoalign/enabled` back to `false` (or unschedule the tuning command) before handing control back to the driver.

<figure><img src="../.gitbook/assets/autoalign-tuning-converged.png" alt=""><figcaption><p>Robot settled at the target pose (<code>X: 11.975m, Y: 3.977m, θ: 90.00°</code> against a target of <code>12, 4, 90</code>).</p></figcaption></figure>

## Module Drive/Azimuth Live PID Tuning

`SwerveDriveTelemetryConfig` also exposes a shared set of tunable feedback (PID) gains, feedforward (`kS`/`kV`/`kA`) gains, and a setpoint for every module's drive motor (`modules/drive/feedback/p,i,d`, `modules/drive/feedforward/s,v,a`, `modules/drive/velocity`, `modules/drive/inplace`) and azimuth motor (`modules/azimuth/feedback/p,i,d`, `modules/azimuth/feedforward/s,v,a`, `modules/azimuth/angle`), gated by `modules/drive/enabled` and `modules/azimuth/enabled` toggles respectively. These are **shared across every module** on the drive, not per-module fields: flipping `modules/drive/enabled` to `true` commands the same velocity setpoint (and applies the same gains) to every module's drive motor simultaneously, which is the normal case for tuning a symmetric swerve chassis without repeating the same numbers four times.

`SwerveDrive` reads these fields from the same `Mechanisms/<name>/tuning` command used for auto-align tuning (see above); scheduling that command is what makes both auto-align and module tuning live.

{% hint style="success" %}
**You don't need a gyro wired up yet.** In simulation, `SwerveDrive` tracks its own simulated gyro angle internally and never touches `SwerveDriveConfig.withGyro(...)`, so you can bring up and tune drive/azimuth PID with this feature before you've written any gyro-reading code at all. On a real robot, though, `SwerveDrive`'s constructor calls into the gyro supplier immediately and throws `IllegalStateException` if `withGyro(...)` was never called, so add it before you deploy to hardware. A sensible order: tune drive/azimuth PID in sim first, wire up the real gyro, then move on to auto-align and field-relative driving.
{% endhint %}

1. `modules/drive/feedback/p,i,d`, `modules/drive/feedforward/s,v,a`, `modules/drive/velocity`, `modules/drive/enabled`, `modules/drive/inplace`, `modules/azimuth/feedback/p,i,d`, `modules/azimuth/feedforward/s,v,a`, `modules/azimuth/angle`, and `modules/azimuth/enabled` are all already live-tunable at `HIGH` verbosity. The gain entries are pre-seeded from the first module's configured drive/azimuth PID and `SimpleMotorFeedforward`, not `0`.
2. Schedule `Mechanisms/<name>/tuning` (the same command as auto-align tuning).
3. Edit `modules/drive/feedback/p`/`i`/`d` and `modules/drive/feedforward/s`/`v`/`a` live; each gain only propagates to every module's drive motor the moment it actually changes, so tuning doesn't spam `SmartMotorController.setKp(...)`/`setKs(...)` every loop. Set `modules/drive/velocity` (meters per second) to the speed you want to test, then flip `modules/drive/enabled` to `true`, every module immediately targets that velocity at an angle of `0°`, using the current gains.
4. To test drive tuning while spinning the chassis in place instead of driving straight, flip `modules/drive/inplace` to `true` while `modules/drive/enabled` is `true`. Each module's angle is recomputed to point tangent to its position around the robot center, so a positive `modules/drive/velocity` spins the whole robot counter-clockwise, matching WPILib's positive rotation direction. Flipping `modules/drive/inplace` back to `false` returns every module to `0°`.

<figure><img src="../.gitbook/assets/module-drive-tuning.png" alt=""><figcaption><p>AdvantageScope with <code>Tuning/swerve/modules/drive/enabled</code> flipped on: the Swerve widget's <code>Module States</code> source shows every module driving at the commanded <code>-3.00 m/s</code>.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/module-drive-inplace.gif" alt=""><figcaption><p>Raising <code>Tuning/swerve/modules/drive/velocity</code> with <code>modules/drive/inplace</code> already <code>true</code>: every module points tangent to the robot center and the whole chassis spins faster in place as the magnitude increases.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/module-drive-inplace.png" alt=""><figcaption><p><code>Tuning/swerve/modules/drive/enabled</code> and <code>modules/drive/inplace</code> both <code>true</code> with <code>velocity</code> at <code>-3</code>: each module is pinwheeled 90° off its radial angle, and the Swerve widget's rotation arrow (and the field view on the left) confirm the robot is spinning cleanly about its center.</p></figcaption></figure>

5. Do the same for `modules/azimuth/feedback/p`/`i`/`d`, `modules/azimuth/feedforward/s`/`v`/`a`, and `modules/azimuth/angle` (degrees) with `modules/azimuth/enabled`, to tune azimuth position control the same way. Drive velocity is always commanded to `0` while azimuth tuning is active, so the wheel only rotates in place, it never also drives.

<figure><img src="../.gitbook/assets/module-azimuth-tuning.gif" alt=""><figcaption><p>Editing <code>Tuning/swerve/modules/azimuth/angle</code> and watching every module rotate to the new heading live.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/module-azimuth-tuning.png" alt=""><figcaption><p>The full <code>Tuning/swerve/modules</code> tree in AdvantageScope: <code>azimuth/angle</code>, <code>enabled</code>, <code>feedback/p,i,d</code>, and <code>feedforward/a,s,v</code>, with <code>drive</code>'s equivalents just below.</p></figcaption></figure>

6. Copy the gains that feel right back into the drive/azimuth motors' `SmartMotorControllerConfig.withClosedLoopController(...)`/`withFeedforward(...)` calls in code, then flip both `enabled` toggles back to `false` before driving normally again.

{% hint style="info" %}
Both modes build a `SwerveModuleVelocity[]` (one entry per module) and command it in one call via `SwerveDrive.setSwerveModuleStates(...)`, not the drive/azimuth `SmartMotorController`s directly, so the same optimization `SwerveModuleConfig.withOptimization(true)` applies during normal driving (shortest-path angle flips, velocity deadband) also applies while tuning, and the drive's desired-module-states telemetry stays consistent with what's actually commanded.
{% endhint %}

{% hint style="warning" %}
**Only one tuning mode can be active at a time.** `autoalign/enabled`, `modules/drive/enabled`, and `modules/azimuth/enabled` are mutually exclusive, if you flip on a second one while another is already `true`, `SwerveDrive` immediately forces the others back to `false` in NetworkTables (priority order: auto-align, then drive tuning, then azimuth tuning), so the dashboard reflects which mode actually won. When `modules/drive/enabled` is `false` (and auto-align isn't driving the chassis instead), every module's drive motor is continuously commanded to `0` m/s so the wheels don't keep spinning at whatever velocity was last set while it was on. `modules/drive/inplace` only changes each module's target angle while `modules/drive/enabled` is `true`: `false` (the default) points every module at `0°`, `true` points them tangent to their position for a counter-clockwise spin in place.&#x20;

There's no equivalent auto-zero for azimuth, turning off `modules/azimuth/enabled` just stops commanding a new angle, it doesn't snap the wheels back to `0°`.
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
