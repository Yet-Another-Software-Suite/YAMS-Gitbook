---
icon: elevator
---

# Elevators

Every `ElevatorConfig` option, with a graphic and an example, is on the [ElevatorConfig](mechanism-configs/elevator-config.md) page. This page walks through a complete mechanism.

## Basic Elevator Config

Given the [tutorial](../tutorials/elevator.md#create-a-smartmotorcontrollerconfig) and [how to find your mechanism circumference](../how-to/how-to-find-your-mechanism-circumference.md), we have a basic config below.

```java
import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;
import com.revrobotics.util.CANPorts;
import org.wpilib.math.controller.ElevatorFeedforward;
import org.wpilib.math.system.DCMotor;
import yams.commands2.config.SmartMotorControllerConfig; // yams.commands3.config for Commands v3
import yams.commands2.mechanisms.Elevator;               // yams.commands3.mechanisms for Commands v3
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.ElevatorConfig;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.local.SparkWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
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
  // In this example gearbox(3,4) is the same as gearbox("3:1","4:1") which corresponds to the gearbox attached to your motor.
  .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
  // Motor properties to prevent over currenting.
  .withMotorInverted(false)
  .withZeroPower(MotorMode.BRAKE)
  .withStatorCurrentLimit(Amps.of(40))
  .withClosedLoopRampRate(Seconds.of(0.25))
  .withOpenLoopRampRate(Seconds.of(0.25))
  .withStartingPosition(Meters.of(0.5));// Starting height of the Elevator

// Vendor motor controller object
SparkMax spark = new SparkMax(CANPorts.fromBusId(1), 4, MotorType.kBrushless);

// Create our SmartMotorController from our Spark and config with the NEO.
SmartMotorController sparkSmartMotorController = new SparkWrapper(spark, DCMotor.getNEO(1), smcConfig);

ElevatorConfig elevconfig = new ElevatorConfig()
      .withHardLimits(Meters.of(0), Meters.of(3)) // Hard limits defined
      .withTelemetry("Elevator", TelemetryVerbosity.HIGH) // Telemetry Name
      .withCarriageWeight(Pounds.of(16)); // Mass of the carriage

Elevator m_elevator = new Elevator(elevconfig, sparkSmartMotorController);
```

## Carriage Weight

The carriage weight defined via `.withCarriageWeight` is used for simulation of the elevator. The weight given should be the carriage weight. Any changes in carriage weight will require a retuning of the PID and FF, and could necessitate a [sim only PID and FF](editor/simulation-only-pid-+-feedforward.md) to maintain an easy comparison.

```java
ElevatorConfig elevconfig = new ElevatorConfig()
      .withCarriageWeight(Pounds.of(16)); // Weight of the carriage.
```

## Cascading Elevators

Cascading elevators need their gearings divided by the number of stages there are in the elevator.

```java
// Cascading Elevator with a 3:1 and 4:1 gearbox and 2 stages.
new MechanismGearing(GearBox.fromReductionStages(3, 4).div(2));
new MechanismGearing(GearBox.fromReductionStages(3, 4)).div(2);
```

## Linear Slides and Slanted Elevators

A linear slide is still an elevator! `.withAngle` sets the angle it is drawn at, and `.withHorizontalElevator()` turns gravity off in the simulation. The angle alone does not change the simulated gravity.

```java
ElevatorConfig elevconfig = new ElevatorConfig()
      .withAngle(Degrees.of(0)) // Drawn parallel to the ground.
      .withHorizontalElevator(); // No gravity in simulation: a linear slide.
```

A linear slide is guaranteed to have a [different sim PID and FF](editor/simulation-only-pid-+-feedforward.md), so be sure to set them!

## Hard Limits and Soft Limits

The difference between hard limits and soft limits are ubiquitous among all mechanisms.

* Hard Limits are where physical stops that should not break are located along the mechanisms movement.
* Soft Limits are where the motor controller PID should limit itself to. It does not always limit itself within these limits which is why Hard Limits should also be used.

Hard limits are the simulation maximum boundaries.

```java
SmartMotorControllerConfig config = config.withSoftLimits(Meters.of(0), Meters.of(3)); // Soft limits
...
ElevatorConfig elevconfig = new ElevatorConfig()
      .withHardLimits(Meters.of(0), Meters.of(3)) // Hard limits defined 
```

## Code Reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/ElevatorSubsystem.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/ExponentiallyProfiledElevatorSubsystem.java" %}

