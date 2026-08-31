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

{% hint style="info" %}
`clone()` returns an independent copy of a stream, useful for deriving a second variant (e.g. a heading-snap mode below) from a shared base configuration without one affecting the other.
{% endhint %}

## Modes Are Chosen Automatically

A stream is always in exactly one of five internal modes: `ANGULAR_VELOCITY` (the default), `TRANSLATION_ONLY`, `HEADING`, `AIM`, or `DRIVE_TO_POSE`. You never select a mode directly. `get()` re-derives it on every call from which optional pieces are configured and whether the relevant `BooleanSupplier` trigger currently reports true, then transitions cleanly (resetting the relevant PID controllers) if the mode changed since the last call.

`withControllerHeadingAxis(x, y)` plus `withHeadingControl(trigger)` together enable `HEADING`, snapping to a heading derived from a second stick axis instead of rotating at a velocity:

```java
SwerveInputStream headingStream = input.clone()
    .withControllerHeadingAxis(driver::getRightX, driver::getRightY)
    .withHeadingControl(driver::getAButton);
```

`withTranslationOnly(trigger)` enables `TRANSLATION_ONLY` (locks the current heading, translates only), and `withAim(targetPoseSupplier, trigger)` enables `AIM` (rotates to face a target pose while translating normally) the same way, by supplying a trigger for a mode that's otherwise unreachable.

{% hint style="warning" %}
`DRIVE_TO_POSE` exists as an internal mode, but `SwerveInputStream` has no public setter for its target pose or PID controllers yet, so it can't currently be reached from outside the class (and `atTargetPose(...)` can't report anything meaningful either). Use [`SwerveDrive.driveToPose(...)`](swerve-drive.md#auto-align-drivetoposesetpoint-and-live-pid-tuning) for auto-align instead.
{% endhint %}

## Robot-Relative and Alliance-Relative Output

By default a stream outputs field-relative `ChassisSpeeds`. `withRobotRelative()` switches it to robot-relative instead, and `withAllianceRelativeControl()` (already shown above) flips the field-relative translation 180 degrees on the Red alliance, so "forward" always means away from your own driver station regardless of which side you're on.

```java
input.withRobotRelative(); // output robot-relative speeds instead of field-relative
```

{% hint style="warning" %}
`withAllianceRelativeControl()` and `withRobotRelative()` are incompatible outside of `DRIVE_TO_POSE` mode; combining them throws a `RuntimeException` at runtime, not a compile error.
{% endhint %}

## Code Reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/swerve_drive/java/frc/robot/subsystems/SwerveSubsystem.java" %}
