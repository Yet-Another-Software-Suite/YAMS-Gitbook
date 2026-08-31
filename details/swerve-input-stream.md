---
icon: gamepad
---

# Swerve Input Stream

`SwerveInputStream` bridges raw joystick axes (each in the range `[-1, 1]`) and the `ChassisSpeeds` that [`SwerveDrive.drive(...)`](swerve-drive.md) expects. It implements `Supplier<ChassisSpeeds>`, so once built it gets passed straight to `drive(...)` and re-evaluated every scheduler loop; you never call `.get()` on it yourself.

## Building a Stream

`SwerveInputStream.of(drive, x, y)` is the base factory, translation axes only. Passing a third `DoubleSupplier` to a constructor adds a rotation axis; passing two more instead adds heading axes for point-and-hold control. From there, `with*()` methods layer on deadbanding, scaling, non-linear response, and alliance-relative flipping:

```java
XboxController driver = new XboxController(0);

SwerveInputStream input =
    SwerveInputStream.of(drive, () -> -driver.getLeftY(), () -> -driver.getLeftX())
        .withControllerRotationAxis(driver::getRightX)
        .withDeadband(0.05)
        .withScaleTranslation(0.8)
        .withScaleRotation(0.6)
        .withMaximumLinearVelocity(MetersPerSecond.of(4.5))
        .withMaximumAngularVelocity(DegreesPerSecond.of(360))
        .withCubeTranslationControllerAxis()
        .withAllianceRelativeControl();

setDefaultCommand(drive.drive(input));
```

## Mutations Apply Immediately

Every `with*()` method mutates the stream in place and returns `this`; none of them return a new copy. That matters because a `SwerveInputStream` is usually stored once and handed to `drive(...)` as a long-lived `Supplier<ChassisSpeeds>`. Calling a mutator on that same reference later, from anywhere else in your code, changes what the already-running default command does on its very next loop; you never need to rebuild the stream or call `setDefaultCommand(...)` again.

For example, auto-aim can be retargeted on the fly from a completely separate button binding, without touching the drive command at all:

```java
input.withAim(() -> FieldConstants.SPEAKER_POSE, driver::getRightBumperButton);
setDefaultCommand(drive.drive(input));

new Trigger(driver::getLeftBumperButton).onTrue(Commands.runOnce(() ->
    input.withAim(() -> FieldConstants.AMP_POSE, driver::getRightBumperButton)));
```

Pressing the left bumper swaps the aim target from the speaker to the amp. The default command bound back in `setDefaultCommand` keeps running the exact same `input` instance; it just starts computing `ChassisSpeeds` toward a different pose the moment the mutation happens.

{% hint style="info" %}
`clone()` is the one method here that doesn't mutate: it returns an independent copy, useful for deriving a second variant of a stream from a shared base configuration without one affecting the other. See [Copying a Stream](#copying-a-stream) below.
{% endhint %}

## Velocity Limits

`withMaximumLinearVelocity(LinearVelocity)` and `withMaximumAngularVelocity(AngularVelocity)` cap the chassis speed a fully-deflected stick produces, defaulting to 4 m/s and 1 rotation/s if never set.

```java
input.withMaximumLinearVelocity(MetersPerSecond.of(4.5))
     .withMaximumAngularVelocity(DegreesPerSecond.of(360));
```

{% hint style="info" %}
A stream copies `SwerveDriveConfig.withMaximumChassisSpeed(...)`'s value (if set) into its own maximum fields once, at construction time; if the drive has no maximum configured, the stream starts from its own defaults (4 m/s, 1 rotation/s) instead. Calling `withMaximumLinearVelocity(...)`/`withMaximumAngularVelocity(...)` afterward overrides that starting value for this stream only and takes effect immediately; the drive's config is never re-checked after construction, so the override sticks.
{% endhint %}

## Controller Axes

`withControllerRotationAxis(DoubleSupplier)` sets or replaces the rotation axis after construction, and `withControllerHeadingAxis(DoubleSupplier, DoubleSupplier)` sets or replaces the pair of axes used to compute a target heading for `HEADING` mode. Both are useful when a stream starts out with `SwerveInputStream.of(drive, x, y)` (no rotation source at all) and gains one later.

```java
input.withControllerRotationAxis(driver::getRightX);
input.withControllerHeadingAxis(driver::getRightX, driver::getRightY);
```

## Deadband

`withDeadband(double)` (an alias for `deadband(double)`) applies the same deadband to every axis before any scaling or cubing happens. `0` clears it.

```java
input.withDeadband(0.05);
```

## Axis Scaling

`withScaleTranslation(double)` and `withScaleRotation(double)` multiply the translation and rotation axis output by a constant in `(0, 1]`, useful for a "slow mode" that doesn't require re-mapping the physical stick range.

```java
input.withScaleTranslation(0.5)
     .withScaleRotation(0.5); // e.g. bound to a "slow mode" trigger
```

## Non-Linear (Cubic) Response

`withCubeTranslationControllerAxis()` and `withCubeRotationControllerAxis()` cube the translation magnitude and rotation axis respectively, giving finer control near the center of the stick without sacrificing full speed at the edges. Each also has a `BooleanSupplier` overload to toggle the effect live.

```java
input.withCubeTranslationControllerAxis()
     .withCubeRotationControllerAxis(driver::getBButton);
```

## Modes Are Chosen Automatically

A stream is always in exactly one of four internal modes: `ANGULAR_VELOCITY` (the default), `TRANSLATION_ONLY`, `HEADING`, or `AIM`. You never select a mode directly. `get()` re-derives it on every call from which optional pieces are configured and whether the relevant `BooleanSupplier` trigger currently reports true, then transitions cleanly (resetting the relevant PID controllers) if the mode changed since the last call. Mode priority, highest to lowest: `TRANSLATION_ONLY` > `AIM` > `HEADING` > `ANGULAR_VELOCITY`.

{% hint style="info" %}
`SwerveInputStream` used to have a `DRIVE_TO_POSE` mode and a matching `atTargetPose(...)` method; both have been removed. Use [`SwerveDrive.driveToPose(...)`](swerve-drive.md#auto-align-drivetoposesetpoint-and-live-pid-tuning) for auto-align instead.
{% endhint %}

### Heading-Snap Control

`withHeadingControl(trigger)` enables `HEADING` while `trigger` reports true, snapping to a heading derived from `withControllerHeadingAxis(x, y)` instead of rotating at a velocity:

```java
SwerveInputStream headingStream = input.clone()
    .withControllerHeadingAxis(driver::getRightX, driver::getRightY)
    .withHeadingControl(driver::getAButton);
```

### Aim At a Target

`withAim(Supplier<Pose2d> targetPose, BooleanSupplier trigger)` enables `AIM` while `trigger` reports true, rotating to face `targetPose` while translation keeps working normally. See [Mutations Apply Immediately](#mutations-apply-immediately) above for an example that retargets it live.

```java
input.withAim(() -> FieldConstants.SPEAKER_POSE, driver::getRightBumperButton);
```

### Translation Only

`withTranslationOnly(trigger)` enables `TRANSLATION_ONLY` while `trigger` reports true, locking the current heading and suppressing rotation entirely:

```java
input.withTranslationOnly(driver::getStartButton);
```

## Robot-Relative Output

By default a stream outputs field-relative `ChassisSpeeds`. `withRobotRelative()` switches it to robot-relative instead.

```java
input.withRobotRelative(); // output robot-relative speeds instead of field-relative
```

## Alliance-Relative Output

`withAllianceRelativeControl()` flips the field-relative translation 180 degrees on the Red alliance, so "forward" always means away from your own driver station regardless of which side you're on.

```java
input.withAllianceRelativeControl();
```

{% hint style="warning" %}
`withAllianceRelativeControl()` and `withRobotRelative()` are incompatible; combining them throws a `RuntimeException` at runtime, not a compile error.
{% endhint %}

## Translation Heading Offset

`withTranslationHeadingOffset(Rotation2d)` rotates the translation direction by a fixed angle before it's output, useful when a secondary driver's "forward" should map to something other than the robot's own forward.

```java
input.withTranslationHeadingOffset(Rotation2d.fromDegrees(90));
```

## Copying a Stream

`clone()` is the only method on this page that doesn't mutate the original; it returns an independent copy that shares the base stream's configuration at the moment it was called. Changing the copy afterward, or changing the original, doesn't affect the other.

```java
SwerveInputStream headingStream = input.clone();
```

## Code Reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/swerve_drive/java/frc/robot/subsystems/SwerveSubsystem.java" %}
