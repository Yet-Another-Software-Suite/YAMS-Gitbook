---
description: We will learn how to program a Swerve Drive!
---

# Swerve Drive

{% hint style="danger" %}
Swerve drive requires more hardware configuration than a tank drive. Make sure you have installed YAMS (see [Installation](../README.md#installation); this tutorial uses the Commands v2 vendordep), REVLib (for SparkMax motors) and Phoenix 6 (for CANcoder and Pigeon2) before starting.
{% endhint %}

## Intro

At the end of this tutorial you will have a full swerve drive subsystem that works in both real life and simulation with the same code!

## Details

This swerve drive will be using the following hardware and control details:

* 4x `SparkMax` (NEO) drive motors, CAN IDs 1, 4, 7, 10
* 4x `SparkMax` (NEO) azimuth motors, CAN IDs 2, 5, 8, 11
* 4x `CANcoder` absolute encoders, CAN IDs 3, 6, 9, 12
* `Pigeon2` gyro, CAN ID 14
* SDS MK4i modules: drive gear ratio `12.75:1`, steer gear ratio `6.75:1`, 4-inch wheels
* Left stick controls translation, right stick X controls rotation

{% hint style="warning" %}
**Simulation Processing Requirements**: YAMS simulates **each motor individually**. With 4 modules containing 2 motors each, that is 8 `SmartMotorController` instances running physics simulation simultaneously. This provides highly accurate simulation but requires more processing power than simplified kinematic-only simulations.
{% endhint %}

## Let's make a Swerve Drive!

{% stepper %}
{% step %}
#### Configure the Drive Motor Controllers

We start with the `SmartMotorControllerConfig` for the **drive** motors. The drive motors use velocity control, so we configure a PID controller and a feedforward.

<pre class="language-java" data-title="SwerveSubsystem.java" data-line-numbers data-full-width="true"><code class="lang-java">package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.MetersPerSecond;

<strong>import org.wpilib.math.controller.SimpleMotorFeedforward;
</strong><strong>import org.wpilib.math.geometry.Translation2d;
</strong>import org.wpilib.command2.SubsystemBase;
<strong>import yams.core.gearing.MechanismGearing;
</strong><strong>import yams.commands2.config.SmartMotorControllerConfig;
</strong><strong>import yams.core.telemetry.enums.TelemetryVerbosity;
</strong>
public class SwerveSubsystem extends SubsystemBase {

<strong>  // kP=0.3 trims velocity error left after the feedforward.
</strong><strong>  // kV = 12 V / (max wheel speed in rad/s): at full speed the feedforward
</strong><strong>  // alone commands 12 V, leaving PID to correct only disturbances.
</strong><strong>  private SmartMotorControllerConfig buildDriveCfg() {
</strong><strong>    return new SmartMotorControllerConfig(this)
</strong><strong>        .withWheelDiameter(Inches.of(4))
</strong><strong>        .withClosedLoopController(0.3, 0, 0)
</strong><strong>        .withGearing(new MechanismGearing(12.75))  // MK4i L1 ratio
</strong><strong>        .withFeedforward(new SimpleMotorFeedforward(0, 12.0 / (MetersPerSecond.of(1).in(MetersPerSecond) / Inches.of(4).in(Meters)), 0.01))
</strong><strong>        .withStatorCurrentLimit(Amps.of(40))        // prevents belt slip
</strong><strong>        .withTelemetry("driveMotor", TelemetryVerbosity.HIGH);
</strong><strong>  }
</strong>
}
</code></pre>

The `withFeedforward` calculates `kV` from your maximum linear velocity and wheel diameter so the feedforward alone handles most of the work.
{% endstep %}

{% step %}
#### Configure the Azimuth Motor Controllers

The **azimuth** (steering) motors use position control. A single `kP=1` is typically enough at a `6.75:1` reduction because the gear ratio makes the output stiff.

<pre class="language-java" data-title="SwerveSubsystem.java" data-line-numbers data-full-width="true"><code class="lang-java">package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.MetersPerSecond;

import org.wpilib.math.controller.SimpleMotorFeedforward;
import org.wpilib.math.geometry.Translation2d;
import org.wpilib.command2.SubsystemBase;
import yams.core.gearing.MechanismGearing;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class SwerveSubsystem extends SubsystemBase {

  private SmartMotorControllerConfig buildDriveCfg() {
    return new SmartMotorControllerConfig(this)
        .withWheelDiameter(Inches.of(4))
        .withClosedLoopController(0.3, 0, 0)
        .withGearing(new MechanismGearing(12.75))
        .withFeedforward(new SimpleMotorFeedforward(0, 12.0 / (MetersPerSecond.of(1).in(MetersPerSecond) / Inches.of(4).in(Meters)), 0.01))
        .withStatorCurrentLimit(Amps.of(40))
        .withTelemetry("driveMotor", TelemetryVerbosity.HIGH);
  }

<strong>  // kP=1 with a 6.75:1 reduction gives a stiff, responsive steer.
</strong><strong>  // 20 A is enough, the steer motor almost never stalls during normal driving.
</strong><strong>  private SmartMotorControllerConfig buildAzimuthCfg() {
</strong><strong>    return new SmartMotorControllerConfig(this)
</strong><strong>        .withClosedLoopController(1, 0, 0)
</strong><strong>        .withFeedforward(new SimpleMotorFeedforward(0, 1))
</strong><strong>        .withGearing(new MechanismGearing(6.75))   // MK4i steer ratio
</strong><strong>        .withStatorCurrentLimit(Amps.of(20))
</strong><strong>        .withTelemetry("angleMotor", TelemetryVerbosity.HIGH);
</strong><strong>  }
</strong>

}
</code></pre>
{% endstep %}

{% step %}
#### Build a `SwerveModule` Factory

Now we write a helper method that takes the two `SparkMax` objects and a `CANcoder`, wraps them in `SmartMotorController`s, and returns a configured `SwerveModule`.

<pre class="language-java" data-title="SwerveSubsystem.java" data-line-numbers data-full-width="true"><code class="lang-java">package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.MetersPerSecond;

import com.ctre.phoenix6.hardware.CANcoder;
import com.revrobotics.spark.SparkMax;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import org.wpilib.math.geometry.Translation2d;
import org.wpilib.math.system.DCMotor;
import org.wpilib.command2.SubsystemBase;
import yams.core.gearing.MechanismGearing;
<strong>import yams.core.mechanisms.config.SwerveModuleConfig;
</strong><strong>import yams.core.mechanisms.swerve.SwerveModule;
</strong><strong>import yams.core.motorcontrollers.SmartMotorController;
</strong>import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.telemetry.enums.TelemetryVerbosity;
<strong>import yams.core.motorcontrollers.local.SparkWrapper;
</strong>

public class SwerveSubsystem extends SubsystemBase {

  private SmartMotorControllerConfig buildDriveCfg() { /* ... same as before ... */ }
  private SmartMotorControllerConfig buildAzimuthCfg() { /* ... same as before ... */ }

<strong>  /**
</strong><strong>   * All four modules share the same gearing and PID gains, so one factory
</strong><strong>   * method creates all of them. Pass each module a unique name and location.
</strong><strong>   */
</strong><strong>  public SwerveModule createModule(SparkMax drive, SparkMax azimuth,
</strong><strong>                                   CANcoder absoluteEncoder, String moduleName,
</strong><strong>                                   Translation2d location) {
</strong><strong>    SmartMotorController driveSMC   = new SparkWrapper(drive,   DCMotor.getNEO(1), buildDriveCfg());
</strong><strong>    SmartMotorController azimuthSMC = new SparkWrapper(azimuth, DCMotor.getNEO(1), buildAzimuthCfg());
</strong>
<strong>    return new SwerveModule(new SwerveModuleConfig(driveSMC, azimuthSMC)
</strong><strong>        // CANcoder eliminates the need to home the steer motor at startup.
</strong><strong>        .withAbsoluteEncoder(absoluteEncoder.getAbsolutePosition().asSupplier())
</strong><strong>        .withTelemetry(moduleName, TelemetryVerbosity.HIGH)
</strong><strong>        .withLocation(location)
</strong><strong>        // State optimization rotates the module at most 90 deg instead of 180 deg + reversing drive.
</strong><strong>        .withOptimization(true));
</strong><strong>  }
</strong>

}
</code></pre>

The `withLocation(Translation2d)` call uses WPILib's coordinate convention: `+X` is toward the front of the robot, `+Y` is toward the left side.
{% endstep %}

{% step %}
#### Instantiate the Four Modules and Create `SwerveDrive`

In the constructor, we create all four modules and pass them to `SwerveDriveConfig`. The gyro supplier gives YAMS the robot's attitude as a `Rotation3d`: its yaw is the heading reference for field-relative driving.

<pre class="language-java" data-title="SwerveSubsystem.java" data-line-numbers data-full-width="true"><code class="lang-java">package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.MetersPerSecond;

<strong>import com.ctre.phoenix6.CANBus;
</strong>import com.ctre.phoenix6.hardware.CANcoder;
<strong>import com.ctre.phoenix6.hardware.Pigeon2;
</strong><strong>import com.revrobotics.spark.SparkLowLevel.MotorType;
</strong>import com.revrobotics.spark.SparkMax;
<strong>import org.wpilib.hardware.bus.CANPort;
</strong>import org.wpilib.math.controller.PIDController;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import org.wpilib.math.geometry.Pose2d;
import org.wpilib.math.geometry.Rotation2d;
import org.wpilib.math.geometry.Translation2d;
import org.wpilib.math.system.DCMotor;
import org.wpilib.command2.SubsystemBase;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.SwerveModuleConfig;
<strong>import yams.commands2.config.SwerveDriveConfig;
</strong><strong>import yams.commands2.swerve.SwerveDrive;
</strong>import yams.core.mechanisms.swerve.SwerveModule;
import yams.core.motorcontrollers.SmartMotorController;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.telemetry.enums.TelemetryVerbosity;
import yams.core.motorcontrollers.local.SparkWrapper;

public class SwerveSubsystem extends SubsystemBase {

<strong>  private final SwerveDrive drive;
</strong>
<strong>  // CAN ID 14, update to match your robot's Pigeon2 CAN ID.
</strong><strong>  private final Pigeon2 gyro = new Pigeon2(14, new CANBus(CANPort.CAN_S0));
</strong>
  // ... buildDriveCfg(), buildAzimuthCfg(), createModule() from previous steps ...

  public SwerveSubsystem() {
<strong>    // Module locations: +X forward, +Y left. 24-inch offsets assume module
</strong><strong>    // centers are 24 in from the robot center, update to match your chassis.
</strong><strong>    // CAN IDs are grouped as (drive, steer, CANcoder) per module.
</strong><strong>    var fl = createModule(new SparkMax(CANPort.CAN_S0, 1, MotorType.kBrushless),
</strong><strong>                          new SparkMax(CANPort.CAN_S0, 2, MotorType.kBrushless),
</strong><strong>                          new CANcoder(3, new CANBus(CANPort.CAN_S0)), "frontleft",
</strong><strong>                          new Translation2d(Inches.of(24), Inches.of(24)));
</strong><strong>    var fr = createModule(new SparkMax(CANPort.CAN_S0, 4, MotorType.kBrushless),
</strong><strong>                          new SparkMax(CANPort.CAN_S0, 5, MotorType.kBrushless),
</strong><strong>                          new CANcoder(6, new CANBus(CANPort.CAN_S0)), "frontright",
</strong><strong>                          new Translation2d(Inches.of(24), Inches.of(-24)));
</strong><strong>    var bl = createModule(new SparkMax(CANPort.CAN_S0, 7, MotorType.kBrushless),
</strong><strong>                          new SparkMax(CANPort.CAN_S0, 8, MotorType.kBrushless),
</strong><strong>                          new CANcoder(9, new CANBus(CANPort.CAN_S0)), "backleft",
</strong><strong>                          new Translation2d(Inches.of(-24), Inches.of(24)));
</strong><strong>    var br = createModule(new SparkMax(CANPort.CAN_S0, 10, MotorType.kBrushless),
</strong><strong>                          new SparkMax(CANPort.CAN_S0, 11, MotorType.kBrushless),
</strong><strong>                          new CANcoder(12, new CANBus(CANPort.CAN_S0)), "backright",
</strong><strong>                          new Translation2d(Inches.of(-24), Inches.of(-24)));
</strong>
<strong>    SwerveDriveConfig config = new SwerveDriveConfig(this, fl, fr, bl, br)
</strong><strong>        // The gyro's Rotation3d: its yaw is the heading used for field-relative driving.
</strong><strong>        .withGyro(gyro::getRotation3d)
</strong><strong>        .withStartingPose(new Pose2d(0, 0, Rotation2d.fromDegrees(0)))
</strong><strong>        // Translation and rotation PIDs are used by driveToPose(); kP=1 is a conservative start.
</strong><strong>        .withTranslationController(new PIDController(1, 0, 0))
</strong><strong>        .withRotationController(new PIDController(1, 0, 0));
</strong>
<strong>    drive = new SwerveDrive(config);
</strong>  }

}
</code></pre>

{% hint style="info" %}
If your gyro is mounted so that the robot turning shows up as its roll or pitch instead of its yaw, add `.withGyroHeadingAxis(GyroAxis.ROLL)` or `.withGyroHeadingAxis(GyroAxis.PITCH)` to the `SwerveDriveConfig` (`yams.core.mechanisms.config.enums.GyroAxis`, the default is `GyroAxis.YAW`). Read the heading back with `drive.getGyroRotation3d().toRotation2d()`.
{% endhint %}
{% endstep %}

{% step %}
#### Add Periodic Updates and Drive Commands

`periodic()` must call `drive.updateTelemetry()` to keep odometry and telemetry fresh. `simulationPeriodic()` must call `drive.simIterate()` to step the physics engine, and then feeds the simulated heading back into the Pigeon2.

<pre class="language-java" data-title="SwerveSubsystem.java" data-line-numbers data-full-width="true"><code class="lang-java">  // ... fields and constructor from previous steps ...

  /**
   * Drive the robot with field-relative chassis speeds.
   */
<strong>  public Command drive(Supplier&#x3C;ChassisVelocities> speedsSupplier) {
</strong><strong>    return run(() -> drive.setFieldRelativeChassisSpeeds(speedsSupplier.get()))
</strong><strong>        .withName("Field Oriented Drive");
</strong><strong>  }
</strong>
  /** Drive to a specific field pose. */
<strong>  public Command driveToPose(Pose2d pose) { return drive.driveToPose(pose); }
</strong>
  /** Lock wheels in X pattern to resist pushing. */
<strong>  public Command lock() { return run(drive::lockPose); }
</strong>

  @Override
  public void periodic() {
<strong>    drive.updateTelemetry();           // updates pose estimator, telemetry and the drive's Field2d
</strong>  }

  @Override
  public void simulationPeriodic() {
<strong>    drive.simIterate();  // steps the motor physics simulation
</strong><strong>    // Keep the simulated Pigeon2 in step with the simulated robot heading.
</strong><strong>    gyro.getSimState().setRawYaw(drive.getSimPose().getRotation().getMeasure());
</strong>  }
</code></pre>

These methods use `org.wpilib.math.kinematics.ChassisVelocities` (WPILib 2027's name for `ChassisSpeeds`), `org.wpilib.command2.Command` and `java.util.function.Supplier`. `lock()` turns every module toward the robot center (the X pattern) and holds it there.

You can also expose `getPose()`, `resetOdometry()`, and `addVisionMeasurement()` as needed for autonomous and vision integration.

{% hint style="info" %}
`SwerveDrive` builds and publishes its own `Field2d` internally (to `Mechanisms/<name>/field`) and updates it in `updateTelemetry()`, so do not create another one. Call `drive.getField2d()` when vision or autonomous code needs to plot extra objects (e.g. detected AprilTags, path previews) onto the same widget the drive already publishes.
{% endhint %}

{% hint style="info" %}
In simulation, `drive.getSimPose()` returns a ground-truth `Pose2d` computed by assuming every module hit its last-commanded `SwerveModuleVelocity` perfectly, separate from `getPose()`'s noisy, fused odometry estimate. It's a convenient "known truth" source for a simulated vision subsystem to generate synthetic target detections from, letting you test vision-dependent code without physical hardware. It only advances during `simIterate()` and stays at the configured starting pose on a real robot.
{% endhint %}

{% hint style="success" %}
**Tuning the `driveToPose()` PID gains live**: when both `withTranslationController(...)` and `withRotationController(...)` are configured, YAMS publishes a "Live Tuning"-style command at `Mechanisms/<name>/tuning`, along with a tunable target pose (`autoalign/setpoint/x`/`autoalign/setpoint/y` in meters, `autoalign/setpoint/rot` in degrees) gated by an `autoalign/enabled` boolean, plus tunable translation/rotation P/I/D gains, all under the `Tuning` NetworkTables table. Schedule that command from your dashboard, set `autoalign/setpoint/x`/`autoalign/setpoint/y`/`autoalign/setpoint/rot` to the pose you want and flip `autoalign/enabled` to `true`, and it repeatedly drives toward that pose using whatever gains you edit live in NetworkTables, so you can dial in `withTranslationController(...)`/`withRotationController(...)` without redeploying code. `setRotationPID(...)`/`setTranslationPID(...)` only reset a PID controller when a gain actually changes, so tuning doesn't wipe out the integrator every loop.
{% endhint %}
{% endstep %}

{% step %}
#### Create a `SwerveInputStream` and Bind the Default Command

`SwerveInputStream` (`yams.commands2.swerve.SwerveInputStream`) converts raw joystick values into field-relative `ChassisVelocities` with deadband, cubing, and alliance-relative field orientation applied automatically. Set it as the subsystem's default command in `RobotContainer`.

<pre class="language-java" data-title="SwerveSubsystem.java" data-line-numbers><code class="lang-java">  // Add this method to SwerveSubsystem
<strong>  public SwerveInputStream getChassisSpeedsSupplier(DoubleSupplier translationX,
</strong><strong>                                                     DoubleSupplier translationY,
</strong><strong>                                                     DoubleSupplier rotation) {
</strong><strong>    return SwerveInputStream.of(drive, translationX, translationY)
</strong><strong>        .withControllerRotationAxis(rotation)
</strong><strong>        .withMaximumAngularVelocity(DegreesPerSecond.of(360))
</strong><strong>        .withMaximumLinearVelocity(MetersPerSecond.of(1))
</strong><strong>        .withDeadband(0.01)
</strong><strong>        .withCubeRotationControllerAxis()     // non-linear rotation response
</strong><strong>        .withCubeTranslationControllerAxis()  // non-linear translation response
</strong><strong>        .withAllianceRelativeControl()        // alliance-aware field-relative
</strong><strong>        // Publishes the stream's inputs and outputs, and lets you tune it live.
</strong><strong>        .withTelemetry("Driver", TelemetryVerbosity.HIGH);
</strong><strong>  }
</strong>
</code></pre>

<pre class="language-java" data-title="RobotContainer.java" data-line-numbers><code class="lang-java">public class RobotContainer {

<strong>  private final SwerveSubsystem drive = new SwerveSubsystem();
</strong><strong>  private final CommandGamepad controller = new CommandGamepad(0);
</strong>
  public RobotContainer() {
<strong>    // Left stick translates, right stick X rotates.
</strong><strong>    // Pushing a stick forward or left reads negative, so negate each axis to match
</strong><strong>    // WPILib's +X forward, +Y left, counterclockwise positive convention.
</strong><strong>    drive.setDefaultCommand(drive.drive(
</strong><strong>        drive.getChassisSpeedsSupplier(
</strong><strong>            () -> -controller.getLeftY(),
</strong><strong>            () -> -controller.getLeftX(),
</strong><strong>            () -> -controller.getRightX())));
</strong>
    configureBindings();
  }

  private void configureBindings() {
<strong>    controller.leftBumper().whileTrue(drive.driveToPose(
</strong><strong>        new Pose2d(Meters.of(3), Meters.of(3), Rotation2d.fromDegrees(30))));
</strong>  }
}
</code></pre>

`CommandGamepad` comes from `org.wpilib.command2.button.CommandGamepad`. See [Swerve Input Stream](../details/swerve-input-stream.md) for everything the stream can do, including its telemetry and live tuning.
{% endstep %}
{% endstepper %}

## Complete Example

The following files from the YAMS `simple_robot-2027` Commands v2 example show a complete swerve drive implementation matching the hardware described in this tutorial.

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/SwerveSubsystem.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/RobotContainer.java" %}

For the Commands v3 version of the same drive (`yams.commands3`, with a `Mechanism` class and coroutine commands), see `SwerveMechanism.java` in [`examples/commands3/simple_robot-2027`](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands3/simple_robot-2027).

## Next Steps

* Learn about [AdvantageKit Integration](../details/advantagekit-integration.md) for swerve
* See [Organizing Configs in Constants](../details/config-organization.md) for cleaner code
* Record this drive's telemetry to a file and review it after a match, see [How do I view my DataLog in AdvantageScope?](../how-to/how-do-i-view-my-datalog-in-advantagescope.md)
