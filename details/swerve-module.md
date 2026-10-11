---
icon: gear
---

# Swerve Module

Every `SwerveModuleConfig` option, with a graphic and an example, is on the [SwerveModuleConfig](mechanism-configs/swerve-module-config.md) page. This page walks through a complete mechanism.

See [Swerve Drive](swerve-drive.md) first for how modules fit into the drive as a whole. This page goes deeper on `SwerveModule` and `SwerveModuleConfig` themselves.

Both are in `yams.core` and are shared by Commands v2 and Commands v3, so the imports are the same whichever vendordep you use:

```java
import yams.core.mechanisms.config.SwerveModuleConfig;
import yams.core.mechanisms.swerve.SwerveModule;
```

Module states are WPILib 2027 `SwerveModuleVelocity`s (the 2027 name for `SwerveModuleState`).

## A Fully Configured Module

Here's every `SwerveModuleConfig` option from this page in one place, the sections below explain each call:

```java
SwerveModuleConfig moduleConfig = new SwerveModuleConfig(driveSMC, azimuthSMC)
      .withAbsoluteEncoder(encoder.getAbsolutePosition().asSupplier())
      .withAbsoluteEncoderOffset(Rotations.of(0.25))
      .withAbsoluteEncoderGearing(new GearBox(new double[]{1.5}))
      .withLocation(Meters.of(0.3), Meters.of(0.3))
      .withOptimization(true)
      .withCosineCompensation(true)
      .withVelocityDeadband(MetersPerSecond.of(0.05))
      .withTelemetry("frontleft", TelemetryVerbosity.HIGH);

SwerveModule fl = new SwerveModule(moduleConfig);
```

{% hint style="info" %}
`withWheelRadius(Distance)`/`withWheelDiameter(Distance)` also exist on `SwerveModuleConfig`, but the source recommends setting wheel size on the drive motor's `SmartMotorControllerConfig` instead (see [Swerve Drive's Basic Swerve Config](swerve-drive.md#basic-swerve-config)), so they're left out above.
{% endhint %}

## Two Motors, One Module

A `SwerveModule` pairs exactly two `SmartMotorController`s: a drive motor (velocity control, drives the wheel) and an azimuth motor (position control, steers the wheel). `SwerveModuleConfig`'s constructor takes both directly:

```java
SwerveModuleConfig moduleConfig = new SwerveModuleConfig(driveSMC, azimuthSMC);
SwerveModule fl = new SwerveModule(moduleConfig);
```

See [Swerve Drive's Basic Swerve Config](swerve-drive.md#basic-swerve-config) for a full worked example with the rest of the `withAbsoluteEncoder`/`withLocation`/`withOptimization`/`withTelemetry` chain filled in.

{% hint style="info" %}
There's also a no-arg `SwerveModuleConfig()` constructor paired with `withSmartMotorController(drive, azimuth)`, for cases where the config needs to exist before both motor controllers are ready. Calling `withSmartMotorController` twice on the same config throws, it's meant to be set exactly once.
{% endhint %}

```java
SwerveModuleConfig moduleConfig = new SwerveModuleConfig();
moduleConfig.withSmartMotorController(driveSMC, azimuthSMC);
```

## Absolute Encoder Wiring

Unlike an `Arm` or `Elevator`, a swerve azimuth motor has no safe "starting position" to assume, the wheel could be facing any direction when the robot powers on. That's why `SwerveModule` reads the absolute encoder and seeds the azimuth motor's relative encoder from it at construction time (`seedAzimuthEncoder()`), rather than requiring a homing routine.

`SwerveModuleConfig.withAbsoluteEncoder(...)` has two forms:

* Pass a same-vendor absolute encoder object directly, and YAMS wires it into the azimuth `SmartMotorControllerConfig` as an external feedback encoder.
* Pass a `Supplier<Angle>` (or `DoubleSupplier`) for any other absolute encoder, such as a CANcoder read through its own API on a non-CTRE azimuth motor.

A CANcoder feeding a `TalonFXWrapper` azimuth motor is same-vendor, so it can be passed directly:

```java
CANBus canBus = new CANBus(CANPort.CAN_S0);
CANcoder canCoder = new CANcoder(3, canBus);
SmartMotorController azimuthSMC = new TalonFXWrapper(new TalonFX(2, canBus), DCMotor.getKrakenX60(1), azimuthCfg);

SwerveModuleConfig moduleConfig = new SwerveModuleConfig(driveSMC, azimuthSMC)
      .withAbsoluteEncoder(canCoder)
      .withAbsoluteEncoderOffset(Rotations.of(0.25)); // bevel-left zero offset
      
azimuthSMC.applyConfig(azimuthCfg); // Needed to apply the offset to the azimuthSmc
```

Setting the external encoder inside of the azimuth config is recommended so the settings are propagated to the external encoder. `withExternalEncoderZeroOffset(...)` is read back with `getExternalEncoderZeroOffset()` (formerly `getZeroOffset()`).

```java
SmartMotorControllerConfig azimuthCfg = new SmartMotorControllerConfig(this)
      .withExternalEncoder(canCoder)
      .withExternalEncoderZeroOffset(Rotations.of(0.25))
      .withExternalEncoderInverted(true)
      .withExternalEncoderDiscontinuityPoint(Rotations.of(0.5))
      .withUseExternalFeedbackEncoder(true);

SmartMotorController azimuthSMC = new TalonFXWrapper(new TalonFX(2, canBus), DCMotor.getKrakenX60(1), azimuthCfg);

SwerveModuleConfig moduleConfig = new SwerveModuleConfig(driveSMC, azimuthSMC);
```

`withExternalEncoderDiscontinuityPoint(Angle)` must be `Rotations.of(0.5)` or `Rotations.of(1)`. `withUseExternalFeedbackEncoder(true)` is what actually makes the azimuth's closed-loop controller run on the absolute encoder continuously; without it, the absolute encoder is only used to seed the relative encoder once at construction (via `seedAzimuthEncoder()` above), and closed-loop control runs on the relative encoder from then on.

A CANcoder feeding a REV azimuth motor is a different vendor, so it goes through the `Supplier<Angle>` form instead:

```java
.withAbsoluteEncoder(canCoder.getAbsolutePosition().asSupplier())
.withAbsoluteEncoderOffset(Rotations.of(0.25)) // bevel-left zero offset
```

`getAbsoluteEncoderAngle()` returns the angle with gearing and offset applied, the value actually used for seeding and optimization. `getRawAbsoluteEncoderAngle()` returns the same reading with no offset or gearing conversion applied, useful when you're determining what offset to configure in the first place: point every wheel forward with the bevel gears facing the same direction, read the raw angle from each module, and that's your per-module offset.

```java
Angle angle = moduleConfig.getAbsoluteEncoderAngle();
Angle raw = moduleConfig.getRawAbsoluteEncoderAngle().get();
```

## Optimization and Cosine Compensation

Two `SwerveModuleConfig` flags exist purely to make a module track its setpoint more efficiently:

* `withOptimization(true)` optimizes each state like WPILib's `SwerveModuleVelocity.optimize(...)`, so a module never rotates more than 90 degrees to reach a target heading (it reverses the drive direction instead of steering the long way around). `SwerveDriveConfig.withModuleStateOptimization(boolean)` overrides this flag on every module of a drive at once, even while it is running.
* `withCosineCompensation(true)` scales the commanded drive velocity by the cosine of the angle error between the module's current heading and its desired heading, so a module that hasn't finished rotating yet doesn't also try to drive at full speed in the wrong direction.

```java
moduleConfig.withOptimization(true)
            .withCosineCompensation(true);
```

Both are safe to leave enabled for essentially every swerve module; they exist as flags mainly so you can disable them while debugging module behavior in isolation. `SwerveModuleConfig.getOptimizedState(...)` is what `SwerveModule.setSwerveModuleState(...)` calls every loop to apply both before commanding the motors.

### The Flip-Decision Feedback Loop

When optimization is enabled, the module decides every loop whether to flip 180 degrees (and reverse the drive) or steer straight to the target.

Kinematics recomputes the raw desired angle from scratch every loop with no memory of a prior flip, and a wheel that is still turning can sit near 90 degrees from both orientations of a target that keeps moving, such as a path starting off in a new direction. Choosing by the measured angle alone would reverse the wheel back and forth. `SwerveModuleConfig` therefore remembers `lastCommandedAngle`, the orientation it last actually sent, and keeps that orientation unless the other one is better by more than 30 degrees of hysteresis (see `SwerveModuleVelocityWithAzimuth.optimize(...)`). That avoids the module chattering between two headings.

## Velocity Deadband

`SwerveModuleConfig.withVelocityDeadband(LinearVelocity)` sets a wheel speed at or below which the module stops and holds its current angle, instead of steering to the angle the kinematics ask for. The default is 0.1 m/s.

<figure><img src="../.gitbook/assets/swerve-velocity-deadband.gif" alt="Animation of a wheel whose requested speed sweeps from 0 to 0.3 meters per second and back. While the request is inside the 0.1 meters per second deadband the commanded speed is zero and the wheel is stopped. Above it the commanded speed equals the request"><figcaption><p>Requested and commanded wheel speed with the default 0.1 m/s deadband.</p></figcaption></figure>

**Why it exists.** When the driver lets go of the stick the requested speed is nearly zero, and the direction of a nearly zero velocity is mostly noise. Without the deadband every module would keep steering to chase that noise, which looks like constant, pointless wheel chatter.

**What it costs.** The robot cannot move slower than the deadband:

| Requested wheel speed | What the module does |
| --- | --- |
| Above the deadband | Drives at that speed and steers normally. |
| At or below the deadband | Stops, and holds the angle it already has. |

That matters whenever the robot is meant to creep:

* **Lining up slowly by hand.** Gentle stick pushes do nothing until the request passes the deadband, then the robot starts with a small jump.
* **The end of auto-align or a path.** As a PID closes in on its target it asks for smaller and smaller speeds. Once they drop inside the deadband the wheels stop, and the robot can settle a little short of the target.
* **Slow turns.** Wheels closer to the center of rotation move slower, so they can fall inside the deadband while the others keep driving.

If the robot stops short or will not creep, lower the deadband. If the wheels twitch at rest, raise it. Pass `null` to turn it off and always steer.

```java
moduleConfig.withVelocityDeadband(MetersPerSecond.of(0.05));
```

The deadband does not apply when the drive locks its wheels with [`lockPose()`](swerve-drive.md#locking-the-wheels-x-pattern): `SwerveModule.lock(Rotation2d)` uses `SwerveModuleConfig.getLockedState(...)`, which turns the wheel to the lock angle at zero speed instead of holding it where it is.

## Steer to Drive Coupling

On most modules the drive gearing turns the wheel when the azimuth rotates, even with the drive motor still, so steering alone would drive the wheel and skew its speed and odometry. `withCouplingRatio(...)` compensates for this the same way the CTRE swerve API does: the drive setpoint includes the motion steering causes, and that motion is removed from the measured wheel speed and distance.

```java
moduleConfig.withCouplingRatio(3.5); // drive motor rotations per azimuth rotation, e.g. kCoupleRatio from CTRE Tuner X
```

`withCouplingRatio(GearBox)` takes the same ratio as a `GearBox` from the drive motor (input) to the azimuth (output). A ratio of `0` disables compensation.

## Module Telemetry

`SwerveModuleConfig.withTelemetry(name, TelemetryVerbosity)` (or the `SwerveModuleTelemetryConfig` overload for finer control) is separate from the parent drive's telemetry, but `SwerveDrive` wires it up for you: `SwerveDriveTelemetry.setupTelemetry(...)` calls `SwerveModule.setupTelemetry(driveName)` for every module, which nests each module's NetworkTables entries under `Mechanisms/<driveName>/modules/<moduleName>`.

`SwerveModuleTelemetryConfig` publishes two fields:

| Field                  | NetworkTables key | `LOW` | `MID` | `HIGH` |
| ---------------------- | ----------------- | :---: | :---: | :----: |
| Absolute encoder angle | `encoder`         |   ✓   |   ✓   |    ✓   |
| SwerveModuleVelocity   | `state`           |       |       |    ✓   |

See [Telemetry](../understanding/telemetry.md) for how struct-encoded fields and verbosity presets work across YAMS generally.

### Using `SwerveModuleTelemetryConfig` Directly

Pass a `SwerveModuleTelemetryConfig` instead of a verbosity level when you need to route a single module's telemetry somewhere specific, such as nesting its DataLog entries under the same prefix as the parent drive:

```java
SwerveModuleTelemetryConfig telemetryCfg =
    new SwerveModuleTelemetryConfig(TelemetryVerbosity.HIGH)
        .withDataLogName("swerve/modules/frontleft");

SwerveModuleConfig moduleConfig = new SwerveModuleConfig(driveSMC, azimuthSMC)
      .withAbsoluteEncoder(encoder.getAbsolutePosition().asSupplier())
      .withLocation(Meters.of(0.3), Meters.of(0.3))
      .withOptimization(true)
      .withTelemetry("frontleft", telemetryCfg);
```

`SwerveModuleTelemetryConfig(TelemetryVerbosity)` is a shorthand constructor equivalent to `new SwerveModuleTelemetryConfig().withTelemetryVerbosity(verbosity)`.

## How SwerveDrive Uses Each Module

`SwerveDrive` computes one `SwerveModuleVelocity[]` per loop from the requested `ChassisVelocities` via kinematics, then calls `setSwerveModuleState(...)` on each module. That call is also where [optimization and cosine compensation](swerve-module.md#optimization-and-cosine-compensation) are actually applied, so the state `SwerveDrive` asked for and the state a module actually drives to can differ slightly by design.

## Code Reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/SwerveSubsystem.java" %}
