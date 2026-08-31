---
icon: gear
---

# Swerve Module

See [Swerve Drive](swerve-drive.md) first for how modules fit into the drive as a whole. This page goes deeper on `SwerveModule` and `SwerveModuleConfig` themselves.

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
      .withMinimumVelocity(MetersPerSecond.of(0.05))
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
CANcoder canCoder = new CANcoder(3);
SmartMotorController azimuthSMC = new TalonFXWrapper(new TalonFX(2), DCMotor.getKrakenX60(1), azimuthCfg);

SwerveModuleConfig moduleConfig = new SwerveModuleConfig(driveSMC, azimuthSMC)
      .withAbsoluteEncoder(canCoder)
      .withAbsoluteEncoderOffset(Rotations.of(0.25)); // bevel-left zero offset
```

{% hint style="info" %}
For a same-vendor encoder, `withAbsoluteEncoder(Object)` and `withAbsoluteEncoderOffset(Angle)` do exactly the same thing as calling `SmartMotorControllerConfig.withExternalEncoder(...)` and `withExternalEncoderZeroOffset(...)` on the azimuth motor's config yourself; they're forwarded there for you. `SwerveModuleConfig` has no equivalent wrapper for inversion, the wrap-around discontinuity point, or switching closed-loop feedback over to the external encoder, so those still need to be set directly on the azimuth `SmartMotorControllerConfig`.
{% endhint %}

Set those on the azimuth `SmartMotorControllerConfig` before constructing the azimuth `SmartMotorController`, then hand the same `CANcoder` object to `withAbsoluteEncoder(...)` as usual; it only ever touches the encoder object itself, so it won't disturb the inversion, discontinuity point, or feedback-source settings you configured directly on `azimuthCfg`:

```java
SmartMotorControllerConfig azimuthCfg = new SmartMotorControllerConfig(this)
      .withExternalEncoderInverted(true)
      .withExternalEncoderDiscontinuityPoint(Rotations.of(0.5))
      .withUseExternalFeedbackEncoder(true);

SmartMotorController azimuthSMC = new TalonFXWrapper(new TalonFX(2), DCMotor.getKrakenX60(1), azimuthCfg);

SwerveModuleConfig moduleConfig = new SwerveModuleConfig(driveSMC, azimuthSMC)
      .withAbsoluteEncoder(canCoder)
      .withAbsoluteEncoderOffset(Rotations.of(0.25));
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

* `withOptimization(true)` calls WPILib's `SwerveModuleState.optimize(...)`, so a module never rotates more than 90 degrees to reach a target heading (it reverses the drive direction instead of steering the long way around).
* `withCosineCompensation(true)` scales the commanded drive velocity by the cosine of the angle error between the module's current heading and its desired heading, so a module that hasn't finished rotating yet doesn't also try to drive at full speed in the wrong direction.

```java
moduleConfig.withOptimization(true)
            .withCosineCompensation(true);
```

Both are safe to leave enabled for essentially every swerve module; they exist as flags mainly so you can disable them while debugging module behavior in isolation. `SwerveModuleConfig.getOptimizedState(...)` is what `SwerveModule.setSwerveModuleState(...)` calls every loop to apply both before commanding the motors.

### The Flip-Decision Feedback Loop

When optimization is enabled, WPILib's `SwerveModuleState.optimize(...)` decides whether to flip a module 180 degrees, and that decision needs a reference angle to compare against.

`SwerveModuleConfig` deliberately compares against `lastCommandedAngle`, the angle it last actually commanded, rather than the live absolute encoder reading. Kinematics recomputes the raw desired angle from scratch every loop with no memory of a prior flip. If the flip decision were re-derived from the encoder instead, a debounced flip in progress would get pulled back toward the raw angle by the encoder's own reading before the debounce could ever confirm it, a feedback loop that could leave the module chattering between two headings. Comparing against the last commanded angle avoids that entirely.

## Minimum Velocity

`SwerveModuleConfig.withMinimumVelocity(LinearVelocity)` sets a deadband below which `getOptimizedState(...)` zeroes the commanded speed and holds the module at its current angle instead of the kinematically-desired one. This keeps a module from spinning to face a new heading every time the requested drive speed is nearly zero, which otherwise reads as constant, pointless wheel chatter when the driver lets go of the stick.

```java
moduleConfig.withMinimumVelocity(MetersPerSecond.of(0.05));
```

{% hint style="warning" %}
`SwerveModuleConfig` also has a `couplingRatio` field intended for differential swerve modules where azimuth rotation mechanically couples into drive rotation, but it isn't wired into any public setter yet (tracked as a TODO in the source). Don't rely on it being applied.
{% endhint %}

## Module Telemetry

`SwerveModuleConfig.withTelemetry(name, TelemetryVerbosity)` (or the `SwerveModuleTelemetryConfig` overload for finer control) is separate from the parent drive's telemetry, but `SwerveDrive` wires it up for you: `SwerveDriveTelemetry.setupTelemetry(...)` calls `SwerveModule.setupTelemetry(driveName)` for every module, which nests each module's NetworkTables entries under `Mechanisms/<driveName>/modules/<moduleName>`.

`SwerveModuleTelemetryConfig` publishes two fields:

| Field                  | NetworkTables key | `LOW` | `MID` | `HIGH` |
| ----------------------- | ------------------- | :---: | :---: | :----: |
| Absolute encoder angle  | `encoder`            |   ✓   |   ✓   |    ✓   |
| SwerveModuleState       | `state`              |       |       |    ✓   |

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

`SwerveDrive` computes one `SwerveModuleState[]` per loop from the requested `ChassisSpeeds` via kinematics, then calls `setSwerveModuleState(...)` on each module. That call is also where [optimization and cosine compensation](#optimization-and-cosine-compensation) are actually applied, so the state `SwerveDrive` asked for and the state a module actually drives to can differ slightly by design.

## Code Reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/swerve_drive/java/frc/robot/subsystems/SwerveSubsystem.java" %}
