---
description: We will learn how to program a Double FlyWheel shooter!
---

# Double FlyWheel

{% hint style="danger" %}
Mechanism classes are meant to be used with "tightly coupled" mechanisms where the Mechanism has 1 or more motor controlling it on a connected shaft, gearbox, or other linkage.

**IF** your mechanism is "loosely coupled", you **CAN** still use YAMS. **HOWEVER** you have to create and control the `SmartMotorController` directly as shown in [how-do-i-control-a-mechanism-without-a-mechanism-class.md](../how-to/how-do-i-control-a-mechanism-without-a-mechanism-class.md "mention") OR use `SmartMotorControllerConfig.withLooselyCoupledFollowers(SmartMotorController...)`
{% endhint %}

## Intro

At the end of this tutorial you will have a **Double FlyWheel** shooter with two independently controlled flywheels that will work in both real life and simulation with the same code! Double flywheels are commonly used in FRC games where spin control on the game piece is important - one wheel spinning faster than the other can add backspin or topspin to shots.

## What is a Double FlyWheel?

A double flywheel is a shooter mechanism with two separate flywheels, typically arranged as:

* **Upper FlyWheel** - Controls the top surface of the game piece
* **Lower FlyWheel** - Controls the bottom surface of the game piece

By running these at different speeds, you can:

* Add **backspin** for lob shots (upper faster than lower)
* Add **topspin** for line drive shots (lower faster than upper)
* Achieve **neutral spin** for consistent shots (equal speeds)

## Details

This `DoubleFlyWheel` will use the following hardware specs and control details:

* Two `TalonFX` (Kraken X60) motors controlling each flywheel independently
* `3:1` GearBox on each flywheel
* 4 inch diameter, 1 pound flywheels
* Pressing the bottom face button (`A` on an Xbox controller) will set both flywheels to 3000 RPM (neutral spin)
* Pressing the right face button (`B` on an Xbox controller) will set upper to 3500 RPM, lower to 2500 RPM (backspin)
* Pressing the left face button (`X` on an Xbox controller) will set upper to 2500 RPM, lower to 3500 RPM (topspin)
* Pressing the top face button (`Y` on an Xbox controller) will stop both flywheels

## Lets create a WPILib Command-Based Project!

{% hint style="success" %}
IF you already have a project and know how to place the double flywheel into your own subsystem please skip to [#install-yams](double-flywheel.md#install-yams "mention")
{% endhint %}

### Setup our Command-Based Project

Follow [WPILib's tutorial](https://docs.wpilib.org/en/stable/docs/zero-to-robot/step-4/creating-test-drivetrain-program-cpp-java-python.html) on how to create a Command-Based project.

Bring up the Visual Studio Code command palette with <kbd>Ctrl+Shift+P</kbd>. Then, type "WPILib" into the prompt and select "Create a new project".

1. Click on **Select a project type (Example or Template)**
2. Select **Template** then **Java** then **Command v2 Robot**
3. Click on **Select a new project folder** and select a folder to store your robot project in.
4. Fill in **Project Name** with the name of your robot code project.
5. Enter your **Team Number**
6. Be sure to check **Enable Desktop Support** so we can run simulations!

### Install YAMS!

YAMS ships as two vendordeps, one for each WPILib command framework. This tutorial uses Commands v2, so install **Yet Another Mechanism System (Commands v2)**. Only install one of the two; they conflict with each other.

Click on the **WPILib logo** on the **left** pane. Scroll down to **Yet Another Mechanism System (Commands v2)** and click **Install**!

If it is not in the list, open the command palette with <kbd>Ctrl+Shift+P</kbd>, run **WPILib: Manage Vendor Libraries**, choose **Install new libraries (online)** and paste `https://cdn.yassrobotics.com/yams_commands2.json`. See [Installation](../README.md#installation) for the Commands v3 vendordep and the offline install.

Install the Phoenix 6 vendor library the same way, since this tutorial uses `TalonFX` motor controllers.

## Let's make a Double FlyWheel!

{% hint style="info" %}
A subsystem should hold at most one YAMS mechanism class, so that live tuning keeps working. Both flywheels here are driven together by one subsystem, so this tutorial controls the two `SmartMotorController`s directly instead of creating two `FlyWheel` mechanisms. This is the same approach as the `DoubleFlyWheelSubsystem` in the YAMS examples.

If you would rather have a `FlyWheel` mechanism (with its `Mechanism2d` and Trigger factories) for each wheel, give each wheel its own subsystem and follow the [Shooter/FlyWheels](shooter-flywheels.md) tutorial for both.
{% endhint %}

{% stepper %}
{% step %}
#### Create our Subsystem with two `SmartMotorControllerConfig`s

We need separate configurations for each flywheel since they will be controlled independently.

```java
// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Pounds;
import static org.wpilib.units.Units.Seconds;

import org.wpilib.command2.SubsystemBase;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class DoubleFlywheelSubsystem extends SubsystemBase {

  // Upper FlyWheel Motor Config
  private SmartMotorControllerConfig upperConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      // PID Constants for velocity control
      .withClosedLoopController(0.1, 0, 0)
      .withSimClosedLoopController(0.1, 0, 0)
      // Feedforward Constants - helps track changing RPM goals
      .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withSimFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      // Telemetry name and verbosity level
      .withTelemetry("UpperFlyWheel", TelemetryVerbosity.HIGH)
      // Gearing from the motor rotor to final shaft (3:1 reduction)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3)))
      // Moment of inertia for the simulation, estimated from the 4 inch, 1 pound flywheel
      .withMomentOfInertia(Inches.of(4), Pounds.of(1))
      // Motor properties
      .withMotorInverted(false)
      .withZeroPower(MotorMode.COAST)
      .withStatorCurrentLimit(Amps.of(40))
      .withClosedLoopRampRate(Seconds.of(0.25))
      .withOpenLoopRampRate(Seconds.of(0.25));

  // Lower FlyWheel Motor Config
  private SmartMotorControllerConfig lowerConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withClosedLoopController(0.1, 0, 0)
      .withSimClosedLoopController(0.1, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withSimFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withTelemetry("LowerFlyWheel", TelemetryVerbosity.HIGH)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3)))
      .withMomentOfInertia(Inches.of(4), Pounds.of(1))
      .withMotorInverted(true) // Often inverted relative to upper for proper game piece direction
      .withZeroPower(MotorMode.COAST)
      .withStatorCurrentLimit(Amps.of(40))
      .withClosedLoopRampRate(Seconds.of(0.25))
      .withOpenLoopRampRate(Seconds.of(0.25));

  /** Creates a new DoubleFlywheelSubsystem. */
  public DoubleFlywheelSubsystem() {}
}
```
{% endstep %}

{% step %}
#### Create our motor controllers

Create the vendor motor controller objects for each flywheel and wrap each one in a `SmartMotorController`. Then call `updateTelemetry()` and `simIterate()` on both every loop.

```java
package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Pounds;
import static org.wpilib.units.Units.Seconds;

import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.TalonFX;
import org.wpilib.command2.SubsystemBase;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import org.wpilib.math.system.DCMotor;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.remote.TalonFXWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class DoubleFlywheelSubsystem extends SubsystemBase {

  // Upper FlyWheel Motor Config
  private SmartMotorControllerConfig upperConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withClosedLoopController(0.1, 0, 0)
      .withSimClosedLoopController(0.1, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withSimFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withTelemetry("UpperFlyWheel", TelemetryVerbosity.HIGH)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3)))
      .withMomentOfInertia(Inches.of(4), Pounds.of(1))
      .withMotorInverted(false)
      .withZeroPower(MotorMode.COAST)
      .withStatorCurrentLimit(Amps.of(40))
      .withClosedLoopRampRate(Seconds.of(0.25))
      .withOpenLoopRampRate(Seconds.of(0.25));

  // Lower FlyWheel Motor Config
  private SmartMotorControllerConfig lowerConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withClosedLoopController(0.1, 0, 0)
      .withSimClosedLoopController(0.1, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withSimFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withTelemetry("LowerFlyWheel", TelemetryVerbosity.HIGH)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3)))
      .withMomentOfInertia(Inches.of(4), Pounds.of(1))
      .withMotorInverted(true)
      .withZeroPower(MotorMode.COAST)
      .withStatorCurrentLimit(Amps.of(40))
      .withClosedLoopRampRate(Seconds.of(0.25))
      .withOpenLoopRampRate(Seconds.of(0.25));

  // Vendor motor controller objects
  private TalonFX upperTalon = new TalonFX(1, new CANBus(CANPort.CAN_S0));
  private TalonFX lowerTalon = new TalonFX(2, new CANBus(CANPort.CAN_S0));

  // Create our SmartMotorControllers with Kraken X60 motors
  private SmartMotorController upperMotor = new TalonFXWrapper(upperTalon, DCMotor.getKrakenX60(1), upperConfig);
  private SmartMotorController lowerMotor = new TalonFXWrapper(lowerTalon, DCMotor.getKrakenX60(1), lowerConfig);

  /** Creates a new DoubleFlywheelSubsystem. */
  public DoubleFlywheelSubsystem() {
    // There is no mechanism class to do this for us, so set up the telemetry (and live tuning) ourselves.
    upperMotor.setupTelemetry();
    lowerMotor.setupTelemetry();
  }

  @Override
  public void periodic() {
    // Update telemetry for both flywheels
    upperMotor.updateTelemetry();
    lowerMotor.updateTelemetry();
  }

  @Override
  public void simulationPeriodic() {
    // Run simulation for both flywheels
    upperMotor.simIterate();
    lowerMotor.simIterate();
  }
}
```
{% endstep %}

{% step %}
#### Create `Command`s for our Double FlyWheel

Now we create command factory methods for different shot types. A single command sets both velocity setpoints, so both wheels always change together.

```java
package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Pounds;
import static org.wpilib.units.Units.RPM;
import static org.wpilib.units.Units.Seconds;

import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.TalonFX;
import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import org.wpilib.math.system.DCMotor;
import org.wpilib.units.measure.AngularVelocity;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.remote.TalonFXWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class DoubleFlywheelSubsystem extends SubsystemBase {

  // Upper FlyWheel Motor Config
  private SmartMotorControllerConfig upperConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withClosedLoopController(0.1, 0, 0)
      .withSimClosedLoopController(0.1, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withSimFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withTelemetry("UpperFlyWheel", TelemetryVerbosity.HIGH)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3)))
      .withMomentOfInertia(Inches.of(4), Pounds.of(1))
      .withMotorInverted(false)
      .withZeroPower(MotorMode.COAST)
      .withStatorCurrentLimit(Amps.of(40))
      .withClosedLoopRampRate(Seconds.of(0.25))
      .withOpenLoopRampRate(Seconds.of(0.25));

  // Lower FlyWheel Motor Config
  private SmartMotorControllerConfig lowerConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withClosedLoopController(0.1, 0, 0)
      .withSimClosedLoopController(0.1, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withSimFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withTelemetry("LowerFlyWheel", TelemetryVerbosity.HIGH)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3)))
      .withMomentOfInertia(Inches.of(4), Pounds.of(1))
      .withMotorInverted(true)
      .withZeroPower(MotorMode.COAST)
      .withStatorCurrentLimit(Amps.of(40))
      .withClosedLoopRampRate(Seconds.of(0.25))
      .withOpenLoopRampRate(Seconds.of(0.25));

  // Vendor motor controller objects
  private TalonFX upperTalon = new TalonFX(1, new CANBus(CANPort.CAN_S0));
  private TalonFX lowerTalon = new TalonFX(2, new CANBus(CANPort.CAN_S0));

  // Create our SmartMotorControllers with Kraken X60 motors
  private SmartMotorController upperMotor = new TalonFXWrapper(upperTalon, DCMotor.getKrakenX60(1), upperConfig);
  private SmartMotorController lowerMotor = new TalonFXWrapper(lowerTalon, DCMotor.getKrakenX60(1), lowerConfig);

  /** Creates a new DoubleFlywheelSubsystem. */
  public DoubleFlywheelSubsystem() {
    // There is no mechanism class to do this for us, so set up the telemetry (and live tuning) ourselves.
    upperMotor.setupTelemetry();
    lowerMotor.setupTelemetry();
  }

  /**
   * Set both flywheel speeds independently.
   *
   * @param upperSpeed Upper flywheel target speed
   * @param lowerSpeed Lower flywheel target speed
   * @return Command to set both flywheel speeds
   */
  public Command setFlywheelSpeeds(AngularVelocity upperSpeed, AngularVelocity lowerSpeed) {
    return run(() -> {
      upperMotor.setVelocity(upperSpeed);
      lowerMotor.setVelocity(lowerSpeed);
    }).withName("Set FlyWheel Speeds");
  }

  /**
   * Neutral spin shot - both wheels at same speed.
   *
   * @return Command for neutral spin shot
   */
  public Command neutralSpinShot() {
    return setFlywheelSpeeds(RPM.of(3000), RPM.of(3000));
  }

  /**
   * Backspin shot - upper wheel faster for lob shots.
   *
   * @return Command for backspin shot
   */
  public Command backspinShot() {
    return setFlywheelSpeeds(RPM.of(3500), RPM.of(2500));
  }

  /**
   * Topspin shot - lower wheel faster for line drives.
   *
   * @return Command for topspin shot
   */
  public Command topspinShot() {
    return setFlywheelSpeeds(RPM.of(2500), RPM.of(3500));
  }

  /**
   * Stop both flywheels.
   *
   * @return Command to stop flywheels
   */
  public Command stopFlywheels() {
    return setFlywheelSpeeds(RPM.of(0), RPM.of(0));
  }

  /**
   * Check if both flywheels are near the given target velocities.
   *
   * @param upperTarget Upper flywheel target velocity
   * @param lowerTarget Lower flywheel target velocity
   * @param tolerance Acceptable velocity error tolerance
   * @return true if both flywheels are within tolerance of the given targets
   */
  public boolean atTargetVelocity(AngularVelocity upperTarget, AngularVelocity lowerTarget, AngularVelocity tolerance) {
    return upperMotor.getMechanismVelocity().isNear(upperTarget, tolerance)
        && lowerMotor.getMechanismVelocity().isNear(lowerTarget, tolerance);
  }

  /**
   * Get the current upper flywheel velocity.
   *
   * @return Upper flywheel velocity
   */
  public AngularVelocity getUpperVelocity() {
    return upperMotor.getMechanismVelocity();
  }

  /**
   * Get the current lower flywheel velocity.
   *
   * @return Lower flywheel velocity
   */
  public AngularVelocity getLowerVelocity() {
    return lowerMotor.getMechanismVelocity();
  }

  @Override
  public void periodic() {
    // Update telemetry for both flywheels
    upperMotor.updateTelemetry();
    lowerMotor.updateTelemetry();
  }

  @Override
  public void simulationPeriodic() {
    // Run simulation for both flywheels
    upperMotor.simIterate();
    lowerMotor.simIterate();
  }
}
```
{% endstep %}

{% step %}
#### Configure controller bindings in `RobotContainer`

Wire up the controller buttons to our shot types.

```java
package first.robot;

import first.robot.subsystems.DoubleFlywheelSubsystem;
import org.wpilib.command2.button.CommandGamepad;

public class RobotContainer {

  private final DoubleFlywheelSubsystem doubleFlywheel = new DoubleFlywheelSubsystem();
  private final CommandGamepad controller = new CommandGamepad(0);

  public RobotContainer() {
    configureBindings();
  }

  private void configureBindings() {
    // Bottom face button (A) - Neutral spin shot (both wheels at 3000 RPM)
    controller.faceDown().onTrue(doubleFlywheel.neutralSpinShot());

    // Right face button (B) - Backspin shot (upper faster)
    controller.faceRight().onTrue(doubleFlywheel.backspinShot());

    // Left face button (X) - Topspin shot (lower faster)
    controller.faceLeft().onTrue(doubleFlywheel.topspinShot());

    // Top face button (Y) - Stop flywheels
    controller.faceUp().onTrue(doubleFlywheel.stopFlywheels());
  }
}
```
{% endstep %}
{% endstepper %}

## Using Loosely Coupled Followers

If your double flywheel design has both motors mechanically linked (same shaft), you can use loosely coupled followers instead of commanding each motor yourself:

```java
SmartMotorControllerConfig lowerConfig = new SmartMotorControllerConfig(this)
    // ... config options
    ;

SmartMotorController lowerMotor = new TalonFXWrapper(lowerTalon, DCMotor.getKrakenX60(1), lowerConfig);

SmartMotorControllerConfig upperConfig = new SmartMotorControllerConfig(this)
    // ... config options
    .withLooselyCoupledFollowers(lowerMotor); // Lower follows upper's setpoints

SmartMotorController upperMotor = new TalonFXWrapper(upperTalon, DCMotor.getKrakenX60(1), upperConfig);
```

{% hint style="warning" %}
Loosely coupled followers only forward velocity and position requests, **NOT** DutyCycle or Voltage requests. Command each `SmartMotorController` yourself when you need fully independent control.
{% endhint %}

## Advanced: Dynamic Spin Control

For more advanced shot control, you can create a method that calculates spin ratios:

```java
/**
 * Set flywheel speeds based on spin ratio.
 *
 * @param baseSpeed The average speed of both wheels
 * @param spinRatio Ratio from -1.0 (max topspin) to 1.0 (max backspin)
 * @return Command to set calculated flywheel speeds
 */
public Command setSpinShot(AngularVelocity baseSpeed, double spinRatio) {
  double ratio = Math.max(-1.0, Math.min(1.0, spinRatio)); // Clamp to [-1, 1]
  double upperMultiplier = 1.0 + (ratio * 0.25); // +/- 25% variation
  double lowerMultiplier = 1.0 - (ratio * 0.25);
  
  return setFlywheelSpeeds(
      RPM.of(baseSpeed.in(RPM) * upperMultiplier),
      RPM.of(baseSpeed.in(RPM) * lowerMultiplier)
  );
}
```

## Summary

You now have a fully functional double flywheel shooter with:

* Independent velocity control for upper and lower flywheels
* Preset shot types (neutral, backspin, topspin)
* Full simulation support
* Telemetry for tuning both flywheels

The key differences from a single flywheel are:

1. Two separate `SmartMotorControllerConfig` instances
2. Two separate `SmartMotorController` instances, controlled directly instead of through a `FlyWheel` mechanism
3. The subsystem calls `setupTelemetry()`, `updateTelemetry()` and `simIterate()` on each `SmartMotorController`
4. Commands that set both velocity setpoints from a single command

## Complete Example

The following files from the YAMS `simple_robot-2027` Commands v2 example show a complete double flywheel subsystem with two independent flywheels, and a hood pivot in its own subsystem.

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/doubleflywheel/DoubleFlyWheelSubsystem.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/HoodSubsystem.java" %}
