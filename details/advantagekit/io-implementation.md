# Step 2: Implement the IO class

With YAMS, you only need **one IO implementation** that works for both real hardware and simulation. The `SmartMotorController` automatically detects when it's running in simulation and uses physics simulation instead of real hardware.

## Arm IO Implementation (Works for Real and Sim!)

```java
package first.robot.subsystems.akit;

import static org.wpilib.units.Units.*;

import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.TalonFX;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.ArmFeedforward;
import org.wpilib.math.system.DCMotor;
import org.wpilib.command2.SubsystemBase;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.ArmConfig;
import yams.commands2.mechanisms.Arm;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.remote.TalonFXWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

/**
 * Single IO implementation for the arm - works in BOTH real and simulation!
 * Uses YAMS Arm mechanism class while pulling telemetry from the underlying SmartMotorController.
 */
public class ArmIOTalonFX implements ArmIO {

  private final Arm arm;
  private final SmartMotorController motor;

  public ArmIOTalonFX(SubsystemBase subsystem, int canId) {
    TalonFX talonFX = new TalonFX(canId, new CANBus(CANPort.CAN_S0));
    
    // Step 1: Create SmartMotorControllerConfig
    SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(subsystem)
        .withGearing(new MechanismGearing(GearBox.fromReductionStages(5, 4, 3)))
        .withClosedLoopController(5, 0, 0.1)
        .withSoftLimits(Rotations.of(-0.25), Rotations.of(0.25))
        .withFeedforward(new ArmFeedforward(0.1, 0.3, 0.5, 0.01))
        .withTrapezoidalProfile(RotationsPerSecond.of(1.0), RotationsPerSecondPerSecond.of(2.0))
        .withStartingPosition(Rotations.of(0));  // Starting position lives on the SmartMotorControllerConfig
    
    // Step 2: Create SmartMotorController (TalonFXWrapper)
    SmartMotorController smc = new TalonFXWrapper(talonFX, DCMotor.getKrakenX60(1), smcConfig);
    
    // Step 3: Create ArmConfig
    ArmConfig armConfig = new ArmConfig()
        .withLength(Inches.of(18))           // Arm length, used for simulation physics
        .withHardLimits(Rotations.of(-0.3), Rotations.of(0.3))  // Physical hard stops for sim
        .withTelemetry("Arm", TelemetryVerbosity.HIGH);
    
    // Step 4: Create Arm mechanism, simulation is handled automatically
    this.arm = new Arm(armConfig, smc);
    
    // Get reference to underlying SmartMotorController for telemetry
    this.motor = arm.getMotor();
  }

  @Override
  public void updateInputs(ArmIOInputs inputs) {
    // Pull telemetry data from the underlying SmartMotorController
    inputs.positionRotations = motor.getMechanismPosition().in(Rotations);
    inputs.velocityRotationsPerSec = motor.getMechanismVelocity().in(RotationsPerSecond);
    inputs.appliedVolts = motor.getVoltage().in(Volts);
    inputs.supplyCurrentAmps = motor.getSupplyCurrent().map(c -> c.in(Amps)).orElse(0.0);
    inputs.statorCurrentAmps = motor.getStatorCurrent().in(Amps);
    inputs.temperatureCelsius = motor.getTemperature().in(Celsius);
    inputs.targetPositionRotations = motor.getMechanismPositionSetpoint()
        .map(a -> a.in(Rotations)).orElse(0.0);
  }

  @Override
  public void setTargetAngle(double rotations) {
    // Use SmartMotorController's setPosition method
    motor.setPosition(Rotations.of(rotations));
  }

  @Override
  public void stop() {
    motor.setVoltage(Volts.of(0));
  }
  
  /** Access the Arm mechanism for command helpers like run() and runTo() */
  public Arm getArm() {
    return arm;
  }

  @Override
  public void simIterate() {
    arm.simIterate();
  }

  @Override
  public void updateTelemetry() {
    arm.updateTelemetry();
  }
}
```

## Elevator IO Implementation (Works for Real and Sim!)

```java
package first.robot.subsystems.akit;

import static org.wpilib.units.Units.*;

import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.TalonFX;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.ElevatorFeedforward;
import org.wpilib.math.system.DCMotor;
import org.wpilib.command2.SubsystemBase;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.ElevatorConfig;
import yams.commands2.mechanisms.Elevator;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.remote.TalonFXWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

/**
 * Single IO implementation - uses YAMS Elevator mechanism with SmartMotorController telemetry.
 */
public class ElevatorIOTalonFX implements ElevatorIO {

  private final Elevator elevator;
  private final SmartMotorController motor;

  public ElevatorIOTalonFX(SubsystemBase subsystem, int canId) {
    TalonFX talonFX = new TalonFX(canId, new CANBus(CANPort.CAN_S0));
    
    // Step 1: Create SmartMotorControllerConfig
    SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(subsystem)
        .withGearing(new MechanismGearing(GearBox.fromReductionStages(5, 4)))
        .withDrumRadius(Inches.of(0.75))  // Drum radius
        .withStartingPosition(Meters.of(0.5))  // Starting height, used for simulation
        .withClosedLoopController(10, 0, 0.5)
        .withSoftLimits(Meters.of(0.02), Meters.of(1.2))
        .withFeedforward(new ElevatorFeedforward(0.1, 0.2, 0.5, 0.01))
        .withTrapezoidalProfile(MetersPerSecond.of(1.0), MetersPerSecondPerSecond.of(2.0));
    
    // Step 2: Create SmartMotorController (TalonFXWrapper)
    SmartMotorController smc = new TalonFXWrapper(talonFX, DCMotor.getKrakenX60(1), smcConfig);
    
    // Step 3: Create ElevatorConfig
    ElevatorConfig elevatorConfig = new ElevatorConfig()
        .withCarriageWeight(Pounds.of(10))                 // Carriage mass - used for simulation physics
        .withHardLimits(Meters.of(0), Meters.of(1.5))  // Physical hard stops for sim
        .withTelemetry("Elevator", TelemetryVerbosity.HIGH);
    
    // Step 4: Create Elevator mechanism, simulation is handled automatically
    this.elevator = new Elevator(elevatorConfig, smc);
    
    // Get reference to underlying SmartMotorController for telemetry
    this.motor = elevator.getMotor();
  }

  @Override
  public void updateInputs(ElevatorIOInputs inputs) {
    // Pull telemetry data from the underlying SmartMotorController
    inputs.positionMeters = motor.getMeasurementPosition().in(Meters);
    inputs.velocityMetersPerSec = motor.getMeasurementVelocity().in(MetersPerSecond);
    inputs.appliedVolts = motor.getVoltage().in(Volts);
    inputs.supplyCurrentAmps = motor.getSupplyCurrent().map(c -> c.in(Amps)).orElse(0.0);
    inputs.statorCurrentAmps = motor.getStatorCurrent().in(Amps);
    inputs.temperatureCelsius = motor.getTemperature().in(Celsius);
    // The setpoint is stored as a mechanism angle; convert it to a height with the config
    inputs.targetPositionMeters = motor.getMechanismPositionSetpoint()
        .map(a -> motor.getConfig().convertFromMechanism(a).in(Meters)).orElse(0.0);
  }

  @Override
  public void setTargetHeight(double meters) {
    // Use SmartMotorController's setPosition method with Distance
    motor.setPosition(Meters.of(meters));
  }

  @Override
  public void stop() {
    motor.setVoltage(Volts.of(0));
  }
  
  /** Access the Elevator mechanism for command helpers like run() and runTo() */
  public Elevator getElevator() {
    return elevator;
  }

  @Override
  public void simIterate() {
    elevator.simIterate();
  }

  @Override
  public void updateTelemetry() {
    elevator.updateTelemetry();
  }
}
```

## Shooter (FlyWheel) IO Implementation (Works for Real and Sim!)

```java
package first.robot.subsystems.akit;

import static org.wpilib.units.Units.*;

import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.TalonFX;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import org.wpilib.math.system.DCMotor;
import org.wpilib.command2.SubsystemBase;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.FlyWheelConfig;
import yams.commands2.mechanisms.FlyWheel;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.remote.TalonFXWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

/**
 * Single IO implementation - uses YAMS FlyWheel mechanism with SmartMotorController telemetry.
 */
public class ShooterIOTalonFX implements ShooterIO {

  private final FlyWheel flywheel;
  private final SmartMotorController motor;

  public ShooterIOTalonFX(SubsystemBase subsystem, int canId) {
    TalonFX talonFX = new TalonFX(canId, new CANBus(CANPort.CAN_S0));
    
    // Step 1: Create SmartMotorControllerConfig
    SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(subsystem)
        .withGearing(new MechanismGearing(GearBox.fromReductionStages(1)))  // Direct drive
        .withClosedLoopController(0.5, 0, 0)
        .withSoftLimits(RPM.of(0), RPM.of(6000))  // Velocity soft limits
        .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0.01));
    
    // Step 2: Create SmartMotorController (TalonFXWrapper)
    SmartMotorController smc = new TalonFXWrapper(talonFX, DCMotor.getKrakenX60(1), smcConfig);
    
    // Step 3: Create FlyWheelConfig
    FlyWheelConfig flywheelConfig = new FlyWheelConfig()
        .withDiameter(Inches.of(4))              // Flywheel diameter
        .withTelemetry("Shooter", TelemetryVerbosity.HIGH);
    
    // Step 4: Create FlyWheel mechanism, simulation is handled automatically
    this.flywheel = new FlyWheel(flywheelConfig, smc);
    
    // Get reference to underlying SmartMotorController for telemetry
    this.motor = flywheel.getMotor();
  }

  @Override
  public void updateInputs(ShooterIOInputs inputs) {
    // Pull telemetry data from the underlying SmartMotorController
    inputs.velocityRotationsPerSec = motor.getMechanismVelocity().in(RotationsPerSecond);
    inputs.appliedVolts = motor.getVoltage().in(Volts);
    inputs.supplyCurrentAmps = motor.getSupplyCurrent().map(c -> c.in(Amps)).orElse(0.0);
    inputs.statorCurrentAmps = motor.getStatorCurrent().in(Amps);
    inputs.temperatureCelsius = motor.getTemperature().in(Celsius);
    inputs.targetVelocityRotationsPerSec = motor.getMechanismSetpointVelocity()
        .map(v -> v.in(RotationsPerSecond)).orElse(0.0);
  }

  @Override
  public void setTargetVelocity(double rotationsPerSec) {
    // Use SmartMotorController's setVelocity method
    motor.setVelocity(RotationsPerSecond.of(rotationsPerSec));
  }

  @Override
  public void stop() {
    motor.setVoltage(Volts.of(0));
  }
  
  /** Access the FlyWheel mechanism for command helpers like run() and runTo() */
  public FlyWheel getFlyWheel() {
    return flywheel;
  }

  @Override
  public void simIterate() {
    flywheel.simIterate();
  }

  @Override
  public void updateTelemetry() {
    flywheel.updateTelemetry();
  }
}
```

{% hint style="success" %}
**Why This Works**: YAMS's passive simulation is built into every wrapper. When `Robot.isSimulation()` is true, the `TalonFXWrapper` (and all other wrappers) automatically uses the `DCMotor`, gearing, and mass/MOI you provided to simulate physics. You get accurate simulation without writing any simulation-specific code!
{% endhint %}

Next: [Steps 3 and 4: The subsystem and RobotContainer](subsystem-and-robotcontainer.md)
