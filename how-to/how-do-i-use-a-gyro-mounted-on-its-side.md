---
icon: compass
---

# How do I use a gyro mounted on its side?

A swerve drive needs the robot's heading, which is rotation about the axis pointing up through the robot. A gyro mounted flat reports that as its yaw. If packaging forces you to mount the gyro on its side (for example vertically on a superstructure plate), the robot turning shows up as the gyro's roll or pitch instead, and a `SwerveDrive` reading yaw will see almost no heading change. `SwerveDriveConfig.withGyroHeadingAxis(GyroAxis)` tells YAMS which gyro axis points up, so the heading and tilt are read correctly without any math in your robot code.

{% hint style="info" %}
This is currently a Java only feature. The C++ `SwerveDriveConfig` takes a heading supplier in `WithGyro(...)`, so in C++ return the correct axis from that supplier yourself.
{% endhint %}

## Give the drive the gyro's full attitude

`withGyro(...)` takes a `Supplier<Rotation3d>`: the gyro's full 3D attitude, not just a yaw angle. Most gyros expose this directly, such as `pigeon::getRotation3d`.

## Pick the heading axis

`GyroAxis` lives in `yams.core.mechanisms.config.enums` and names the gyro axis that points up through the robot:

<figure><img src="../.gitbook/assets/gyro-axes.png" alt="Three drawings of a gyro on a robot chassis with its X, Y and Z axes. Mounted flat, Z points up, X to the front and Y to the left: GyroAxis.YAW. Standing on its side with X pointing up, Y to the robot's left and Z to the back: GyroAxis.ROLL. Standing on its side with Y pointing up, X to the robot's front and Z to the right: GyroAxis.PITCH"><figcaption><p>Find the gyro axis that points up, and pick the matching <code>GyroAxis</code>.</p></figcaption></figure>

| `GyroAxis` | Gyro mounting | Robot turning shows up as |
|---|---|---|
| `YAW` (default) | Flat, the gyro's Z axis points up | The gyro's yaw |
| `ROLL` | On its side, the gyro's X axis points up. The gyro's Y axis is taken to point to the robot's left | The gyro's roll |
| `PITCH` | On its side, the gyro's Y axis points up. The gyro's X axis is taken to point to the robot's front | The gyro's pitch |

```java
import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.Pigeon2;
import org.wpilib.hardware.bus.CANPort;
import yams.commands2.config.SwerveDriveConfig;
import yams.commands2.swerve.SwerveDrive;
import yams.core.mechanisms.config.enums.GyroAxis;

private final Pigeon2 gyro = new Pigeon2(14, new CANBus(CANPort.CAN_S0));

SwerveDriveConfig config = new SwerveDriveConfig(this, fl, fr, bl, br)
    .withGyro(gyro::getRotation3d)
    // The gyro is mounted with its X axis pointing up, so the robot's heading is the gyro's roll.
    .withGyroHeadingAxis(GyroAxis.ROLL)
    .withStartingPose(new Pose2d(0, 0, Rotation2d.fromDegrees(0)))
    .withTranslationController(new PIDController(1, 0, 0))
    .withRotationController(new PIDController(1, 0, 0));
drive = new SwerveDrive(config);
```

For Commands v3, import `SwerveDriveConfig` and `SwerveDrive` from `yams.commands3` instead. `GyroAxis` is the same class for both.

YAMS rotates the gyro's attitude into the robot's frame before anything reads it. After that, the yaw of the result is the robot's heading, and its roll and pitch are the robot's tilt (which anti-tipping uses), whichever way the gyro is mounted. `withGyroInverted(...)` and `withGyroOffset(...)` apply to the resulting heading.

## Read the heading

`getGyroAngle()` was removed from `SwerveDrive` and `SwerveDriveConfig`. Use `getGyroRotation3d()`, which returns the robot's attitude with the heading axis already applied:

```java
Rotation2d heading = drive.getGyroRotation3d().toRotation2d();
```

The heading wraps at ±180 degrees. Use `drive.getPose().getRotation()` if you want the odometry heading instead of the raw gyro heading.

## Check it

1. Put the robot on blocks or a cart and open the drive's telemetry.
2. Rotate the robot counterclockwise (viewed from above). The gyro heading should increase.
3. If the heading barely changes, the wrong axis is selected: try the other of `ROLL` and `PITCH`.
4. If it changes in the wrong direction, add `.withGyroInverted(true)`.

## Related pages

* [Swerve Drive](../tutorials/swerve-drive.md)
