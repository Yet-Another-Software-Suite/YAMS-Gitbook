---
description: We will learn how to program an Elevator!
---

# Elevator

{% hint style="danger" %}
Mechanism classes are meant to be used with "tightly coupled" mechanisms where the Mechanism has 1 or more motor controlling it on a connected shaft, gearbox, or other linkage.

**IF** your mechanism is "loosely coupled", you **CAN** still use YAMS. **HOWEVER** you have to create and control the `SmartMotorController` directly as shown in [how-do-i-control-a-mechanism-without-a-mechanism-class.md](../how-to/how-do-i-control-a-mechanism-without-a-mechanism-class.md "mention") OR use `SmartMotorControllerConfig.withLooselyCoupledFollowers(SmartMotorController...)`
{% endhint %}

## Intro

At the end of this tutorial you will have an `Elevator` that will work in both real life and simulation with the same code!

{% columns %}
{% column %}
#### Simulation

<figure><img src="../.gitbook/assets/Robot Simulation 2025-08-30 18-27-07.gif" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/Robot Simulation 2025-08-30 18-37-59.gif" alt=""><figcaption></figcaption></figure>
{% endcolumn %}

{% column %}
#### Real Life

{% embed url="https://www.youtube.com/shorts/XKmsW8-85Dg" %}
{% endcolumn %}
{% endcolumns %}

## Details

This `Elevator` will be using the following hardware specs and control details

* `SparkMax` controlling the `Elevator`
* `12:1` GearBox on the `Elevator`
* Pressing the bottom face button (`A` on an Xbox controller) will make the `Elevator` go to 0.5m
* Pressing the right face button (`B` on an Xbox controller) will make the `Elevator` go to 1
* Pressing the left face button (`X` on an Xbox controller) will make the `Elevator` go up.
* Pressing the top face button (`Y` on an Xbox controller) will make the `Elevator` go down.

## Lets create a WPILib Command-Based Project!

{% hint style="success" %}
IF you already have a project and know how to place the `Elevator` mechanism into your own subsystem please skip to [#install-yams](elevator.md#install-yams "mention")
{% endhint %}

### Setup our Command-Based Project

Here we will follow [WPILib's tutorial ](https://docs.wpilib.org/en/stable/docs/zero-to-robot/step-4/creating-test-drivetrain-program-cpp-java-python.html)on how to create a Command-Based project.

Bring up the Visual Studio Code command palette with <kbd>Ctrl+Shift+P</kbd>. Then, type “WPILib” into the prompt. Since all WPILib commands start with “WPILib”, this will bring up the list of WPILib-specific VS Code commands. Now, select the “Create a new project” command:

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

This will bring up the “New Project Creator Window:”

<figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

1. Click on **Select a project type (Example or Template)**
2. Select **Template** then **Java** then **Command v2 Robot**
3. Click on **Select a new project folder** and select a folder to store your robot project in.
4. Fill in **Project Name** with the name of your robot code project.
5. Enter your **Team Number** in so you can deploy to your robot.
6. Be sure to check **Enable Desktop Support** so we can run simulations!

If you followed these instructions it should look something like whats filled out below.

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

Congratulations! You now have a Command Based robot project!

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

### Install YAMS!

YAMS ships as two vendordeps, one for each WPILib command framework. This tutorial uses Commands v2, so install **Yet Another Mechanism System (Commands v2)**. Only install one of the two; they conflict with each other.

Click on the **WPILib logo** on the **left** pane. Scroll down to **Yet Another Mechanism System (Commands v2)** and click **Install**!

<figure><img src="../.gitbook/assets/RobotContainer.java - simple_robot - Visual Studio Code 9_2_2025 3_40_21 PM.png" alt=""><figcaption></figcaption></figure>

If it is not in the list, open the command palette with <kbd>Ctrl+Shift+P</kbd>, run **WPILib: Manage Vendor Libraries**, choose **Install new libraries (online)** and paste `https://cdn.yassrobotics.com/yams_commands2.json`. See [Installation](../README.md#installation) for the Commands v3 vendordep and the offline install.

Congratulations you have now installed YAMS! :tada:

## Lets make an Elevator move!

{% stepper %}
{% step %}
#### Create a `SmartMotorControllerConfig`

We are going to start by configuring out motor controller.

<pre class="language-java" data-title="ExampleSubsystem.java" data-line-numbers data-full-width="true"><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.subsystems;

<strong>import static org.wpilib.units.Units.Amps;
</strong><strong>import static org.wpilib.units.Units.Inches;
</strong><strong>import static org.wpilib.units.Units.MetersPerSecond;
</strong><strong>import static org.wpilib.units.Units.MetersPerSecondPerSecond;
</strong><strong>import static org.wpilib.units.Units.Seconds;
</strong>
<strong>import org.wpilib.math.controller.ElevatorFeedforward;
</strong>import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
<strong>import yams.core.mechanisms.SmartMechanism;
</strong><strong>import yams.commands2.config.SmartMotorControllerConfig;
</strong><strong>import yams.core.motorcontrollers.enums.ControlMode;
</strong><strong>import yams.core.motorcontrollers.enums.MotorMode;
</strong><strong>import yams.core.telemetry.enums.TelemetryVerbosity;
</strong>
public class ExampleSubsystem extends SubsystemBase {

<strong>  private SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
</strong><strong>  .withControlMode(ControlMode.CLOSED_LOOP)
</strong><strong>  // Drum radius is required for elevators. Chain-driven: specify chain pitch and tooth count.
</strong><strong>  .withDrumRadius(Inches.of(0.25), 22)
</strong><strong>  // Feedback Constants (PID Constants)
</strong><strong>  .withClosedLoopController(4, 0, 0)
</strong><strong>  .withTrapezoidalProfile(MetersPerSecond.of(0.5), MetersPerSecondPerSecond.of(0.5))
</strong><strong>  .withSimClosedLoopController(4, 0, 0)
</strong><strong>  // Feedforward Constants
</strong><strong>  .withFeedforward(new ElevatorFeedforward(0, 0, 0))
</strong><strong>  .withSimFeedforward(new ElevatorFeedforward(0, 0, 0))
</strong><strong>  // Telemetry name and verbosity level
</strong><strong>  .withTelemetry("ElevatorMotor", TelemetryVerbosity.HIGH)
</strong><strong>  // Gearing from the motor rotor to final shaft.
</strong><strong>  // In this example GearBox.fromReductionStages(3,4) is the same as GearBox.fromStages("3:1","4:1") which corresponds to the gearbox attached to your motor.
</strong><strong>  // You could also use .withGearing(12) which does the same thing.
</strong><strong>  .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
</strong><strong>  // Motor properties to prevent over currenting.
</strong><strong>  .withMotorInverted(false)
</strong><strong>  .withZeroPower(MotorMode.BRAKE)
</strong><strong>  .withStatorCurrentLimit(Amps.of(40))
</strong><strong>  .withClosedLoopRampRate(Seconds.of(0.25))
</strong><strong>  .withOpenLoopRampRate(Seconds.of(0.25));
</strong>
  /** Creates a new ExampleSubsystem. */
  public ExampleSubsystem() {}

  /**
   * Example command factory method.
   *
   * @return a command
   */
  public Command exampleMethodCommand() {
    // Inline construction of command goes here.
    // Subsystem::RunOnce implicitly requires `this` subsystem.
    return runOnce(
        () -> {
          /* one-time action goes here */
        });
  }

  /**
   * An example method querying a boolean state of the subsystem (for example, a digital sensor).
   *
   * @return value of some boolean subsystem state, such as a digital sensor.
   */
  public boolean exampleCondition() {
    // Query some boolean state, such as a digital sensor.
    return false;
  }

  @Override
  public void periodic() {
    // This method will be called once per scheduler run
  }

  @Override
  public void simulationPeriodic() {
    // This method will be called once per scheduler run during simulation
  }
}

</code></pre>

{% hint style="info" %}
**`withDrumRadius` is required for all elevators**, it tells YAMS how far the mechanism travels per motor revolution.

**Chain-driven elevator (sprocket + chain):** pass the chain pitch and tooth count.

* `#25 chain` has a pitch of **0.25 in**
* `#35 chain` has a pitch of **0.375 in**

```java
.withDrumRadius(Inches.of(0.25), 22)   // #25 chain, 22-tooth sprocket
.withDrumRadius(Inches.of(0.375), 16)  // #35 chain, 16-tooth sprocket
```

**Spool or direct-drive drum:** pass the physical radius of the drum.

```java
.withDrumRadius(Inches.of(0.875))  // drum with 0.875" radius
```

**Cascading elevators:** call `.withCascadingElevatorStages(n)` in addition to `withDrumRadius`. This divides the gearing by the number of stages so position tracking remains correct.

```java
.withDrumRadius(Inches.of(0.25), 22)
.withCascadingElevatorStages(2)  // 2-stage cascade; divides gearing by 2
```
{% endhint %}

{% endstep %}

{% step %}
#### Create our motor controller

To control our `Elevator` motor we will create the vendor motor controller object.

First we install `REVLib` by clicking on the WPILib logo in the left bar.

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

Now we add it to the `ExampleSubsystem.java`

<pre class="language-java" data-title="ExampleSubsystem.java" data-line-numbers><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.MetersPerSecond;
import static org.wpilib.units.Units.MetersPerSecondPerSecond;
import static org.wpilib.units.Units.Seconds;

<strong>import com.revrobotics.spark.SparkLowLevel.MotorType;
</strong><strong>import com.revrobotics.spark.SparkMax;
import org.wpilib.hardware.bus.CANPort;
</strong>
import org.wpilib.math.controller.ElevatorFeedforward;
import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.SmartMechanism;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class ExampleSubsystem extends SubsystemBase {

  private SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Drum radius is required for elevators. Chain-driven: specify chain pitch and tooth count.
  .withDrumRadius(Inches.of(0.25), 22)
  // Feedback Constants (PID Constants)
  .withClosedLoopController(4, 0, 0)
  .withTrapezoidalProfile(MetersPerSecond.of(0.5), MetersPerSecondPerSecond.of(0.5))
  .withSimClosedLoopController(4, 0, 0)
  // Feedforward Constants
  .withFeedforward(new ElevatorFeedforward(0, 0, 0))
  .withSimFeedforward(new ElevatorFeedforward(0, 0, 0))
  // Telemetry name and verbosity level
  .withTelemetry("ElevatorMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example GearBox.fromReductionStages(3,4) is the same as GearBox.fromStages("3:1","4:1") which corresponds to the gearbox attached to your motor.
  // You could also use .withGearing(12) which does the same thing.
  .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
  // Motor properties to prevent over currenting.
  .withMotorInverted(false)
  .withZeroPower(MotorMode.BRAKE)
  .withStatorCurrentLimit(Amps.of(40))
  .withClosedLoopRampRate(Seconds.of(0.25))
  .withOpenLoopRampRate(Seconds.of(0.25));

<strong>  // Vendor motor controller object
</strong><strong>  private SparkMax spark = new SparkMax(CANPort.CAN_S0, 4, MotorType.kBrushless);
</strong>

  /** Creates a new ExampleSubsystem. */
  public ExampleSubsystem() {}

  /**
   * Example command factory method.
   *
   * @return a command
   */
  public Command exampleMethodCommand() {
    // Inline construction of command goes here.
    // Subsystem::RunOnce implicitly requires `this` subsystem.
    return runOnce(
        () -> {
          /* one-time action goes here */
        });
  }

  /**
   * An example method querying a boolean state of the subsystem (for example, a digital sensor).
   *
   * @return value of some boolean subsystem state, such as a digital sensor.
   */
  public boolean exampleCondition() {
    // Query some boolean state, such as a digital sensor.
    return false;
  }

  @Override
  public void periodic() {
    // This method will be called once per scheduler run
  }

  @Override
  public void simulationPeriodic() {
    // This method will be called once per scheduler run during simulation
  }
}
</code></pre>
{% endstep %}

{% step %}
#### Create our `SmartMotorController`

Our `SmartMotorController` will easily configure and interface with the vendor motor controller object.

<pre class="language-java" data-title="ExampleSubsystem.java" data-line-numbers><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.MetersPerSecond;
import static org.wpilib.units.Units.MetersPerSecondPerSecond;
import static org.wpilib.units.Units.Seconds;

import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;
import org.wpilib.hardware.bus.CANPort;

import org.wpilib.math.controller.ElevatorFeedforward;
<strong>import org.wpilib.math.system.DCMotor;
</strong>import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.SmartMechanism;
<strong>import yams.core.motorcontrollers.SmartMotorController;
</strong>import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.telemetry.enums.TelemetryVerbosity;
<strong>import yams.core.motorcontrollers.local.SparkWrapper;
</strong>
public class ExampleSubsystem extends SubsystemBase {

  private SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Drum radius is required for elevators. Chain-driven: specify chain pitch and tooth count.
  .withDrumRadius(Inches.of(0.25), 22)
  // Feedback Constants (PID Constants)
  .withClosedLoopController(4, 0, 0)
  .withTrapezoidalProfile(MetersPerSecond.of(0.5), MetersPerSecondPerSecond.of(0.5))
  .withSimClosedLoopController(4, 0, 0)
  // Feedforward Constants
  .withFeedforward(new ElevatorFeedforward(0, 0, 0))
  .withSimFeedforward(new ElevatorFeedforward(0, 0, 0))
  // Telemetry name and verbosity level
  .withTelemetry("ElevatorMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example GearBox.fromReductionStages(3,4) is the same as GearBox.fromStages("3:1","4:1") which corresponds to the gearbox attached to your motor.
  // You could also use .withGearing(12) which does the same thing.
  .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
  // Motor properties to prevent over currenting.
  .withMotorInverted(false)
  .withZeroPower(MotorMode.BRAKE)
  .withStatorCurrentLimit(Amps.of(40))
  .withClosedLoopRampRate(Seconds.of(0.25))
  .withOpenLoopRampRate(Seconds.of(0.25));

  // Vendor motor controller object
  private SparkMax spark = new SparkMax(CANPort.CAN_S0, 4, MotorType.kBrushless);

<strong>  // Create our SmartMotorController from our Spark and config with the NEO.
</strong><strong>  private SmartMotorController sparkSmartMotorController = new SparkWrapper(spark, DCMotor.getNEO(1), smcConfig);
</strong>
  /** Creates a new ExampleSubsystem. */
  public ExampleSubsystem() {}

  /**
   * Example command factory method.
   *
   * @return a command
   */
  public Command exampleMethodCommand() {
    // Inline construction of command goes here.
    // Subsystem::RunOnce implicitly requires `this` subsystem.
    return runOnce(
        () -> {
          /* one-time action goes here */
        });
  }

  /**
   * An example method querying a boolean state of the subsystem (for example, a digital sensor).
   *
   * @return value of some boolean subsystem state, such as a digital sensor.
   */
  public boolean exampleCondition() {
    // Query some boolean state, such as a digital sensor.
    return false;
  }

  @Override
  public void periodic() {
    // This method will be called once per scheduler run
  }

  @Override
  public void simulationPeriodic() {
    // This method will be called once per scheduler run during simulation
  }
}

</code></pre>
{% endstep %}

{% step %}
#### Create and Configure our `Elevator`

Our `Elevator` will easily configure the `SmartMotorController` and create a simple and intuitive interface.

<pre class="language-java" data-title="ExampleSubsystem.java" data-line-numbers><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.MetersPerSecond;
import static org.wpilib.units.Units.MetersPerSecondPerSecond;
<strong>import static org.wpilib.units.Units.Feet;
</strong><strong>import static org.wpilib.units.Units.Pounds;
</strong>import static org.wpilib.units.Units.Seconds;

import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;
import org.wpilib.hardware.bus.CANPort;

import org.wpilib.math.controller.ElevatorFeedforward;
import org.wpilib.math.system.DCMotor;
import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.SmartMechanism;
<strong>import yams.core.mechanisms.config.ElevatorConfig;
</strong><strong>import yams.commands2.mechanisms.Elevator;
</strong>import yams.core.motorcontrollers.SmartMotorController;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.telemetry.enums.TelemetryVerbosity;
import yams.core.motorcontrollers.local.SparkWrapper;

public class ExampleSubsystem extends SubsystemBase {

  private SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Drum radius is required for elevators. Chain-driven: specify chain pitch and tooth count.
  .withDrumRadius(Inches.of(0.25), 22)
  // Feedback Constants (PID Constants)
  .withClosedLoopController(4, 0, 0)
  .withTrapezoidalProfile(MetersPerSecond.of(0.5), MetersPerSecondPerSecond.of(0.5))
  .withSimClosedLoopController(4, 0, 0)
  // Feedforward Constants
  .withFeedforward(new ElevatorFeedforward(0, 0, 0))
  .withSimFeedforward(new ElevatorFeedforward(0, 0, 0))
  // Telemetry name and verbosity level
  .withTelemetry("ElevatorMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example GearBox.fromReductionStages(3,4) is the same as GearBox.fromStages("3:1","4:1") which corresponds to the gearbox attached to your motor.
  // You could also use .withGearing(12) which does the same thing.
  .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
  // Motor properties to prevent over currenting.
  .withMotorInverted(false)
  .withZeroPower(MotorMode.BRAKE)
  .withStatorCurrentLimit(Amps.of(40))
  .withClosedLoopRampRate(Seconds.of(0.25))
  .withOpenLoopRampRate(Seconds.of(0.25));

  // Vendor motor controller object
  private SparkMax spark = new SparkMax(CANPort.CAN_S0, 4, MotorType.kBrushless);

  // Create our SmartMotorController from our Spark and config with the NEO.
  private SmartMotorController sparkSmartMotorController = new SparkWrapper(spark, DCMotor.getNEO(1), smcConfig);

<strong>  private ElevatorConfig elevconfig = new ElevatorConfig()
</strong><strong>      .withHardLimits(Meters.of(0), Meters.of(3))
</strong><strong>      .withTelemetry("Elevator", TelemetryVerbosity.HIGH)
</strong><strong>      .withCarriageWeight(Pounds.of(16));
</strong>
<strong>  // Elevator Mechanism
</strong><strong>  private Elevator elevator = new Elevator(elevconfig, sparkSmartMotorController);
</strong>
  /** Creates a new ExampleSubsystem. */
  public ExampleSubsystem() {}

  /**
   * Example command factory method.
   *
   * @return a command
   */
  public Command exampleMethodCommand() {
    // Inline construction of command goes here.
    // Subsystem::RunOnce implicitly requires `this` subsystem.
    return runOnce(
        () -> {
          /* one-time action goes here */
        });
  }

  /**
   * An example method querying a boolean state of the subsystem (for example, a digital sensor).
   *
   * @return value of some boolean subsystem state, such as a digital sensor.
   */
  public boolean exampleCondition() {
    // Query some boolean state, such as a digital sensor.
    return false;
  }

  @Override
  public void periodic() {
<strong>    // This method will be called once per scheduler run
</strong><strong>    elevator.updateTelemetry();
</strong>  }

  @Override
  public void simulationPeriodic() {
<strong>    // This method will be called once per scheduler run during simulation
</strong><strong>    elevator.simIterate();
</strong>  }
}
</code></pre>
{% endstep %}

{% step %}
#### Create `Command`s with our `Elevator`

We use the `Elevator` class as a interface to create commands!

<pre class="language-java" data-title="ExampleSubsystem.java" data-line-numbers><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.MetersPerSecond;
import static org.wpilib.units.Units.MetersPerSecondPerSecond;
import static org.wpilib.units.Units.Feet;
import static org.wpilib.units.Units.Pounds;
<strong>import static org.wpilib.units.Units.Seconds;
</strong>
import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;
import org.wpilib.hardware.bus.CANPort;

import org.wpilib.math.controller.ElevatorFeedforward;
import org.wpilib.math.system.DCMotor;
<strong>import org.wpilib.units.measure.Distance;
</strong><strong>import org.wpilib.units.measure.Voltage;
</strong>import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.SmartMechanism;
import yams.core.mechanisms.config.ElevatorConfig;
import yams.commands2.mechanisms.Elevator;
import yams.core.motorcontrollers.SmartMotorController;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.telemetry.enums.TelemetryVerbosity;
import yams.core.motorcontrollers.local.SparkWrapper;

public class ExampleSubsystem extends SubsystemBase {

  private SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
    .withControlMode(ControlMode.CLOSED_LOOP)
    // Drum radius is required for elevators. Chain-driven: specify chain pitch and tooth count.
    .withDrumRadius(Inches.of(0.25), 22)
    // Feedback Constants (PID Constants)
    .withClosedLoopController(4, 0, 0)
    .withTrapezoidalProfile(MetersPerSecond.of(0.5), MetersPerSecondPerSecond.of(0.5))
    .withSimClosedLoopController(4, 0, 0)
    // Feedforward Constants
    .withFeedforward(new ElevatorFeedforward(0, 0, 0))
    .withSimFeedforward(new ElevatorFeedforward(0, 0, 0))
    // Telemetry name and verbosity level
    .withTelemetry("ElevatorMotor", TelemetryVerbosity.HIGH)
    // Gearing from the motor rotor to final shaft.
    // In this example GearBox.fromReductionStages(3,4) is the same as GearBox.fromStages("3:1","4:1") which corresponds to the gearbox attached to your motor.
    // You could also use .withGearing(12) which does the same thing.
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
    // Motor properties to prevent over currenting.
    .withMotorInverted(false)
    .withZeroPower(MotorMode.BRAKE)
    .withStatorCurrentLimit(Amps.of(40))
    .withClosedLoopRampRate(Seconds.of(0.25))
    .withOpenLoopRampRate(Seconds.of(0.25));

  // Vendor motor controller object
  private SparkMax spark = new SparkMax(CANPort.CAN_S0, 4, MotorType.kBrushless);

  // Create our SmartMotorController from our Spark and config with the NEO.
  private SmartMotorController sparkSmartMotorController = new SparkWrapper(spark, DCMotor.getNEO(1), smcConfig);

  private ElevatorConfig elevconfig = new ElevatorConfig()
      .withHardLimits(Meters.of(0), Meters.of(3))
      .withTelemetry("Elevator", TelemetryVerbosity.HIGH)
      .withCarriageWeight(Pounds.of(16));

  // Elevator Mechanism
  private Elevator elevator = new Elevator(elevconfig, sparkSmartMotorController);

<strong>  /**
</strong><strong>   * Runs the elevator to the given height and does not end the command when reached.
</strong><strong>   * @param height Distance to go to.
</strong><strong>   * @return a Command
</strong><strong>   */
</strong><strong>  public Command run(Distance height) { return elevator.run(height);}
</strong>  
<strong>  /**
</strong><strong>   * Runs the elevator to the given height and ends the command when reached, but not the closed loop controller.
</strong><strong>   * @param height Distance to go to.
</strong><strong>   * @param tolerance Distance tolerance for completion.
</strong><strong>   * @return A Command
</strong><strong>   */
</strong><strong>  public Command runTo(Distance height, Distance tolerance) { return elevator.runTo(height, tolerance);}
</strong>  
<strong>  /**
</strong><strong>   * Set the elevators closed loop controller setpoint.
</strong><strong>   * @param angle Distance to go to.
</strong><strong>   */
</strong><strong>  public void setHeightSetpoint(Distance height) { elevator.setMeasurementPositionSetpoint(height);}
</strong>
<strong>  /**
</strong><strong>   * Move the elevator up and down.
</strong><strong>   * @param dutycycle [-1, 1] speed to set the elevator too.
</strong><strong>   */
</strong><strong>  public Command set(double dutycycle) { return elevator.set(dutycycle);}
</strong>
<strong>  /**
</strong><strong>   * Set the voltage of the elevator, useful for feeding a WPILib {@link org.wpilib.command2.sysid.SysIdRoutine}.
</strong><strong>   * @param volts Voltage to apply.
</strong><strong>   */
</strong><strong>  public Command setVoltage(Voltage volts) { return elevator.setVoltage(volts);}
</strong>
  /** Creates a new ExampleSubsystem. */
  public ExampleSubsystem() {}

  /**
   * Example command factory method.
   *
   * @return a command
   */
  public Command exampleMethodCommand() {
    // Inline construction of command goes here.
    // Subsystem::RunOnce implicitly requires `this` subsystem.
    return runOnce(
        () -> {
          /* one-time action goes here */
        });
  }

  /**
   * An example method querying a boolean state of the subsystem (for example, a digital sensor).
   *
   * @return value of some boolean subsystem state, such as a digital sensor.
   */
  public boolean exampleCondition() {
    // Query some boolean state, such as a digital sensor.
    return false;
  }

  @Override
  public void periodic() {
    // This method will be called once per scheduler run
    elevator.updateTelemetry();
  }

  @Override
  public void simulationPeriodic() {
    // This method will be called once per scheduler run during simulation
    elevator.simIterate();
  }
}

</code></pre>
{% endstep %}

{% step %}
#### Bind buttons to our `Elevator`

We bind buttons to use the `Command` from our `Elevator`

<pre class="language-java" data-title="RobotContainer.java" data-line-numbers><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot;

import first.robot.Constants.OperatorConstants;
import first.robot.commands.Autos;
import first.robot.commands.ExampleCommand;
import first.robot.subsystems.ExampleSubsystem;

<strong>import static org.wpilib.units.Units.Meters;
</strong>
import org.wpilib.command2.Command;
import org.wpilib.command2.button.CommandGamepad;
import org.wpilib.command2.button.Trigger;

/**
 * This class is where the bulk of the robot should be declared. Since Command-based is a
 * "declarative" paradigm, very little robot logic should actually be handled in the {@link Robot}
 * periodic methods (other than the scheduler calls). Instead, the structure of the robot (including
 * subsystems, commands, and trigger mappings) should be declared here.
 */
public class RobotContainer {
  // The robot's subsystems and commands are defined here...
  private final ExampleSubsystem exampleSubsystem = new ExampleSubsystem();

  private final CommandGamepad driverController =
      new CommandGamepad(OperatorConstants.DRIVER_CONTROLLER_PORT);

  /** The container for the robot. Contains subsystems, OI devices, and commands. */
  public RobotContainer() {
    // Configure the trigger bindings
    configureBindings();

<strong>    // Set the default command to force the elevator to go to 0.
</strong><strong>    exampleSubsystem.setDefaultCommand(exampleSubsystem.run(Meters.of(0)));
</strong>  }

  /**
   * Use this method to define your trigger->command mappings. Triggers can be created via the
   * {@link Trigger#Trigger(java.util.function.BooleanSupplier)} constructor with an arbitrary
   * predicate, or via the named factories in {@link org.wpilib.command2.button.CommandGenericHID}'s
   * subclasses for {@link CommandGamepad Gamepad} gamepads or {@link
   * org.wpilib.command2.button.CommandJoystick Flight joysticks}.
   */
  private void configureBindings() {
    
<strong>    // Schedule `run` while the Gamepad's bottom (A) or right (B) face button is held,
</strong><strong>    // cancelling on release.
</strong><strong>    driverController.faceDown().whileTrue(exampleSubsystem.run(Meters.of(0.5)));
</strong><strong>    driverController.faceRight().whileTrue(exampleSubsystem.run(Meters.of(1)));
</strong><strong>    // Schedule `set` while the Gamepad's left (X) or top (Y) face button is held,
</strong><strong>    // cancelling on release.
</strong><strong>    driverController.faceLeft().whileTrue(exampleSubsystem.set(0.3));
</strong><strong>    driverController.faceUp().whileTrue(exampleSubsystem.set(-0.3));
</strong>
  }

  /**
   * Use this to pass the autonomous command to the main {@link Robot} class.
   *
   * @return the command to run in autonomous
   */
  public Command getAutonomousCommand() {
    // An example command will be run in autonomous
    return Autos.exampleAuto(exampleSubsystem);
  }
}

</code></pre>
{% endstep %}

{% step %}
#### Simulate our Elevator!

We can use our `Elevator` in simulation, with the exact same code that will control the real robot!

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

Connect a controller to your system and drag and drop from **System Joysticks** to **Joystick\[0]**

Open up the Simulated mechanism with **NetworkTables -> SmartDashboard -> Elevator-> mechanism**

<figure><img src="../.gitbook/assets/Robot Simulation 8_30_2025 7_06_08 PM.png" alt=""><figcaption></figcaption></figure>

Resize **Elevator/mechanism** to your liking

Press **Teleoperated** in **Robot State** then you can use your controller like its controlling the real robot!

<figure><img src="../.gitbook/assets/Robot Simulation 2025-08-30 18-37-59.gif" alt=""><figcaption></figcaption></figure>

Congratulations on successfully programming your Elevator!! :tada::tada:
{% endstep %}
{% endstepper %}

## Complete Example

The following files show a complete, working Elevator from the YAMS `simple_robot-2027` Commands v2 example. It uses an exponential profile instead of the trapezoidal profile in this tutorial.

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/ElevatorSubsystem.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/RobotContainer.java" %}

For the Commands v3 version of the same robot (a `Mechanism` class and coroutine commands with `yams.commands3`), see [`examples/commands3/simple_robot-2027`](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands3/simple_robot-2027).
