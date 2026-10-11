---
icon: castle
---

# Turrets/Wrists

Every `PivotConfig` option, with a graphic and an example, is on the [PivotConfig](mechanism-configs/pivot-config.md) page. This page walks through a complete mechanism.

## Why are Turrets and Wrists the same thing? Aren't they both Arms?

Turrets and Wrists are known as Pivots in YAMS. They are different from Arms as they are not impacted by gravity to such a significant extent.

## Basic Turret Config

From the YAMS examples we have the following as a basic config.

```java
import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.TalonFXS;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.system.DCMotor;
import yams.commands2.config.SmartMotorControllerConfig; // yams.commands3.config for Commands v3
import yams.commands2.mechanisms.Pivot;                  // yams.commands3.mechanisms for Commands v3
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.PivotConfig;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.remote.TalonFXSWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

TalonFXS                   turretMotor = new TalonFXS(1, new CANBus(CANPort.CAN_S0));
SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withClosedLoopController(4, 0, 0)
      .withTrapezoidalProfile(DegreesPerSecond.of(180), DegreesPerSecondPerSecond.of(90))
      // Configure Motor and Mechanism properties
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
      .withZeroPower(MotorMode.BRAKE)
      .withMotorInverted(false)
      // Setup Telemetry
      .withTelemetry("TurretMotor", TelemetryVerbosity.HIGH)
      // Power Optimization
      .withStatorCurrentLimit(Amps.of(40))
      .withClosedLoopRampRate(Seconds.of(0.25))
      .withOpenLoopRampRate(Seconds.of(0.25))
      .withContinuousWrapping(Degrees.of(0), Degrees.of(360))// Wrapping enabled bc the pivot can spin infinitely
      .withStartingPosition(Degrees.of(0)) // Starting position of the Pivot
      .withMomentOfInertia(Meters.of(0.25), Pounds.of(4)); // MOI Calculation
SmartMotorController       motor       = new TalonFXSWrapper(turretMotor,
                                                             DCMotor.getNEO(1),
                                                             motorConfig);

PivotConfig                m_config         = new PivotConfig() 
      .withHardLimits(Degrees.of(0), Degrees.of(720)) // Hard limit bc wiring prevents infinite spinning
      .withTelemetry("PivotExample", TelemetryVerbosity.HIGH); // Telemetry
Pivot                      m_pivot          = new Pivot(m_config, motor);
```

## Hard Limits

Hard limits are useful for simulation to emulate hard limits that may or may not be imposed on your robot. It is best to give this a wide range if your mechanism is undefined that way you can see exactly what might happen and notice "that could break the robot" beforehand.

```java
PivotConfig m_config = new PivotConfig()
      .withHardLimits(Degrees.of(0), Degrees.of(720)) // Hard limit bc wiring prevents infinite spinning
```

## MOI

The moment of inertia is only used in simulation to estimate the behavior of the Pivot. If you have your MOI similar to the real mechanism than the PID and FF might be similar, however this is unlikely and you will probably have to [set a sim only PID and FF](editor/simulation-only-pid-+-feedforward.md).

MOI is configured on `SmartMotorControllerConfig` via `.withMomentOfInertia(Distance, Mass)`.

```java
SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withMomentOfInertia(Meters.of(0.25), Pounds.of(4)); // MOI Calculation
```

{% hint style="info" %}
MOI also drives the current draw reported to [Battery Simulation](../understanding/battery-simulation.md), a mechanism with an unrealistic MOI will sag the simulated battery unrealistically too, on top of throwing off PID/FF tuning.
{% endhint %}

## Code Reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/TurretSubsystem.java" %}
