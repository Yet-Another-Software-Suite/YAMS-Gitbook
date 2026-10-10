---
icon: database
---

# AdvantageKit Integration

YAMS integrates seamlessly with [AdvantageKit](https://docs.advantagekit.org/), Team 6328's logging and replay framework. This guide shows how to use YAMS mechanisms with AdvantageKit's IO layer pattern for deterministic log replay.

## What is AdvantageKit?

[AdvantageKit](https://docs.advantagekit.org/getting-started/what-is-advantagekit) is a logging framework that records **all inputs** to your robot code, enabling deterministic replay in simulation. Unlike traditional logging that captures specific values, AdvantageKit captures everything flowing into your code, allowing you to:

* Replay matches exactly as they happened
* Add new logged fields after the fact
* Test code changes against real match data
* Debug issues that only occurred on the real robot

{% hint style="info" %}
AdvantageKit is optional. YAMS works perfectly without it using its built-in telemetry system. Consider AdvantageKit if you want deterministic log replay capabilities.
{% endhint %}

## Architecture Overview

AdvantageKit uses an [IO interface pattern](https://docs.advantagekit.org/data-flow/recording-inputs/io-interfaces) to separate hardware interaction from business logic. Traditionally, this requires separate IO implementations for real hardware and simulation:

```
┌─────────────────────────────────────────────────────────────┐
│  Traditional AdvantageKit Pattern (without YAMS)            │
│                                                             │
│                      Subsystem                              │
│                          │                                  │
│                    IO Interface                             │
│                    /           \                            │
│              IO Real           IO Sim                       │
│         (Real hardware)    (Physics sim)                    │
│          - Manual motor     - DCMotorSim                    │
│          - Manual encoders  - Manual state                  │
└─────────────────────────────────────────────────────────────┘
```

## YAMS Simplification: One IO Class for Real and Sim

**With YAMS, you don't need separate Real and Sim IO implementations.** Every `SmartMotorController` wrapper (TalonFXWrapper, SparkWrapper, etc.) and every Mechanism class (Arm, Elevator, FlyWheel, SwerveModule, SwerveDrive, etc.) includes **passive simulation** that runs automatically when `Robot.isSimulation()` returns true.

```
┌─────────────────────────────────────────────────────────────┐
│  YAMS Pattern: Single IO Implementation                     │
│                                                             │
│                      Subsystem                              │
│                          │                                  │
│                    IO Interface                             │
│                          │                                  │
│                    IO (Single!)  ◄── Same class for both!   │
│                          │                                  │
│               SmartMotorController                          │
│                    or Mechanism                             │
│                          │                                  │
│          ┌───────────────┴───────────────┐                  │
│          │                               │                  │
│    Real Hardware              Passive Simulation            │
│    (when deployed)            (when in sim)                 │
│                                                             │
│    Automatically selected based on Robot.isSimulation()    │
└─────────────────────────────────────────────────────────────┘
```

This means:

* **One IO class** handles both real robot and simulation
* **No duplicate code** - configuration is written once
* **No SimWrapper needed** - real hardware wrappers simulate themselves
* **Physics are automatic** - based on your DCMotor, gearing, and mass/MOI configuration

{% hint style="success" %}
**Key Insight**: When you create a `TalonFXWrapper`, `TalonFXSWrapper`, `SparkWrapper`, or any YAMS Mechanism, the simulation physics run automatically in sim mode. You write your IO class once using real hardware objects, and YAMS handles the rest.
{% endhint %}

## Step 1: Define the IO Interface

Create an interface that defines what inputs your subsystem reads and what outputs it sends. Following AdvantageKit conventions, inputs are grouped in an `Inputs` class.

### Arm IO Interface

```java
package first.robot.subsystems.akit;

import org.littletonrobotics.junction.AutoLog;

public interface ArmIO {

  /**
   * Inputs that will be logged and replayed.
   * The @AutoLog annotation generates ArmIOInputsAutoLogged class.
   */
  @AutoLog
  public static class ArmIOInputs {
    public double positionRotations = 0.0;
    public double velocityRotationsPerSec = 0.0;
    public double appliedVolts = 0.0;
    public double supplyCurrentAmps = 0.0;
    public double statorCurrentAmps = 0.0;
    public double temperatureCelsius = 0.0;
    public double targetPositionRotations = 0.0;
  }

  /** Update the inputs from hardware. Called every loop cycle. */
  default void updateInputs(ArmIOInputs inputs) {}

  /** Set the target angle for the arm. */
  default void setTargetAngle(double rotations) {}

  /** Stop the arm motor. */
  default void stop() {}

  /** Update the sim telemetry */
  default void simIterate() {}

  /** Update the telemtry */
  default void updateTelemetry() {}
}
```

### Elevator IO Interface

```java
package first.robot.subsystems.akit;

import org.littletonrobotics.junction.AutoLog;

public interface ElevatorIO {

  @AutoLog
  public static class ElevatorIOInputs {
    public double positionMeters = 0.0;
    public double velocityMetersPerSec = 0.0;
    public double appliedVolts = 0.0;
    public double supplyCurrentAmps = 0.0;
    public double statorCurrentAmps = 0.0;
    public double temperatureCelsius = 0.0;
    public double targetPositionMeters = 0.0;
  }

  default void updateInputs(ElevatorIOInputs inputs) {}

  default void setTargetHeight(double meters) {}

  default void stop() {}

  /** Update the sim telemetry */
  default void simIterate() {}

  /** Update the telemtry */
  default void updateTelemetry() {}
}
```

### Shooter IO Interface

```java
package first.robot.subsystems.akit;

import org.littletonrobotics.junction.AutoLog;

public interface ShooterIO {

  @AutoLog
  public static class ShooterIOInputs {
    public double velocityRotationsPerSec = 0.0;
    public double appliedVolts = 0.0;
    public double supplyCurrentAmps = 0.0;
    public double statorCurrentAmps = 0.0;
    public double temperatureCelsius = 0.0;
    public double targetVelocityRotationsPerSec = 0.0;
  }

  default void updateInputs(ShooterIOInputs inputs) {}

  default void setTargetVelocity(double rotationsPerSec) {}

  default void stop() {}

  /** Update the sim telemetry */
  default void simIterate() {}

  /** Update the telemtry */
  default void updateTelemetry() {}
}
```

{% hint style="info" %}
The `@AutoLog` annotation from AdvantageKit generates a class (e.g., `ArmIOInputsAutoLogged`) that handles serialization for log replay. See [AdvantageKit AutoLog documentation](https://docs.advantagekit.org/data-flow/recording-inputs/io-interfaces#autolog) for details.

If the @AutoLog fails to generate the coresponding AutoLogged class, check your build.gradle. In the dependencies section you should have

### Build.gradle

```groovy
dependencies {
    annotationProcessor wpi.java.deps.wpilibAnnotations()
    def akitJson = new groovy.json.JsonSlurper().parseText(
        new File(projectDir.getAbsolutePath() + "/vendordeps/AdvantageKit.json").text)
    annotationProcessor "org.littletonrobotics.akit:akit-autolog:$akitJson.version"
    // ...
}
```

This is the setup used by the YAMS AdvantageKit examples. After changing it, clean the project with `gradlew clean` (Windows) or `./gradlew clean` (Linux/macOS) and then build. If the class still does not generate, try commenting out the `annotationProcessor wpi.java.deps.wpilibAnnotations()` line and clean again.
{% endhint %}

## Step 2: Implement the IO Class

With YAMS, you only need **one IO implementation** that works for both real hardware and simulation. The `SmartMotorController` automatically detects when it's running in simulation and uses physics simulation instead of real hardware.

### Arm IO Implementation (Works for Real and Sim!)

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

### Elevator IO Implementation (Works for Real and Sim!)

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

### Shooter (FlyWheel) IO Implementation (Works for Real and Sim!)

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

## Step 3: Create the Subsystem

The subsystem uses the IO interface and doesn't know whether it's running on real hardware or in simulation.

```java
package first.robot.subsystems.akit;

import static org.wpilib.units.Units.*;

import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import org.wpilib.units.measure.Angle;
import org.littletonrobotics.junction.Logger;

public class ArmSubsystem extends SubsystemBase {

  private final ArmIO io;
  private final ArmIOInputsAutoLogged inputs = new ArmIOInputsAutoLogged();

  public ArmSubsystem(ArmIO io) {
    this.io = io;
  }

  @Override
  public void periodic() {
    // Update the arms telemetry
    io.updateTelemetry();
    // Update and log inputs every cycle
    io.updateInputs(inputs);
    Logger.processInputs("Arm", inputs);
  }

  @Override
  public void simulationPeriodic() {
    io.simIterate();
  }

  /**
   * Command to move the arm to a target angle.
   * Uses run() for continuous control.
   */
  public Command setAngle(double rotations) {
    return run(() -> io.setTargetAngle(rotations))
        .withName("Arm.setAngle(" + rotations + ")");
  }

  /**
   * Command to move the arm to a target angle and finish when reached.
   * Uses runTo() pattern - be careful with default commands!
   */
  public Command goToAngle(double rotations) {
    return run(() -> io.setTargetAngle(rotations))
        .until(() -> isNear(Rotations.of(rotations), Rotations.of(0.01)))
        .withName("Arm.goToAngle(" + rotations + ")");
  }

  /** Returns true if the arm is within tolerance of a target position using WPILib's isNear(). */
  public boolean isNear(Angle target, Angle tolerance) {
    return Rotations.of(inputs.positionRotations).isNear(target, tolerance);
  }

  /** Returns the current arm position in rotations. */
  public double getPositionRotations() {
    return inputs.positionRotations;
  }

  /** Command to stop the arm. */
  public Command stop() {
    return runOnce(() -> io.stop()).withName("Arm.stop");
  }

}
```

{% hint style="warning" %}
See [run() vs runTo() Commands](run-vs-runto.md) for important information about when commands end and how that interacts with default commands.
{% endhint %}

## Step 4: Wire It Up in RobotContainer

With YAMS's passive simulation, you don't need conditional logic for real vs sim. The same IO class works for both!

```java
package first.robot;

import org.wpilib.command2.button.CommandNiDsXboxController;
import first.robot.subsystems.akit.*;

public class RobotContainer {

  private final ArmSubsystem arm;
  private final ElevatorSubsystem elevator;
  private final ShooterSubsystem shooter;

  private final CommandNiDsXboxController controller = new CommandNiDsXboxController(0);

  public RobotContainer() {
    // Same IO implementation works for BOTH real robot and simulation!
    // No need for if(RobotBase.isReal()) checks, YAMS handles it automatically
    arm = new ArmSubsystem(new ArmIOTalonFX(arm, 1));
    elevator = new ElevatorSubsystem(new ElevatorIOTalonFX(elevator, 2));
    shooter = new ShooterSubsystem(new ShooterIOTalonFX(shooter, 3));

    configureBindings();
  }

  private void configureBindings() {
    // Commands work identically on real robot and in simulation
    controller.a().whileTrue(arm.setAngle(0.25));   // rotations
    controller.b().whileTrue(arm.setAngle(-0.25));
    controller.x().whileTrue(elevator.setHeight(1.0)); // meters
    controller.y().whileTrue(shooter.setVelocity(80)); // rotations per second
  }
}
```

{% hint style="info" %}
**Compare to Traditional AdvantageKit**: Without YAMS, you'd need separate `ArmIOReal` and `ArmIOSim` classes with duplicate configuration. YAMS eliminates this duplication entirely.
{% endhint %}

## Log Replay Mode

For [log replay](https://docs.advantagekit.org/getting-started/what-is-advantagekit), AdvantageKit provides a "replay" mode where inputs are read from a log file instead of hardware. This is the **one case** where you need a separate IO implementation: an empty one that does nothing.

```java
package first.robot.subsystems.akit;

/**
 * Empty IO implementation for log replay.
 * AdvantageKit replays the logged inputs, so no hardware interaction needed.
 */
public class ArmIOReplay implements ArmIO {
  // All methods use default implementations (do nothing)
  // AdvantageKit will inject the logged values into inputs
}
```

In RobotContainer, only check for replay mode (the YAMS AdvantageKit examples define `Mode` and `currentMode` in `Robot`):

```java
public RobotContainer() {
  if (Robot.currentMode == Robot.Mode.REPLAY) {
    // Replay mode: empty IO, AdvantageKit injects logged values
    arm = new ArmSubsystem(new ArmIOReplay());
    elevator = new ElevatorSubsystem(new ElevatorIOReplay());
    shooter = new ShooterSubsystem(new ShooterIOReplay());
  } else {
    // Real AND Sim: same IO class works for both!
    arm = new ArmSubsystem(new ArmIOTalonFX(arm, 1));
    elevator = new ElevatorSubsystem(new ElevatorIOTalonFX(elevator, 2));
    shooter = new ShooterSubsystem(new ShooterIOTalonFX(shooter, 3));
  }
}
```

{% hint style="success" %}
**Summary**: With YAMS + AdvantageKit, you only need **2 IO classes** per mechanism:

1. **One real IO class** (e.g., `ArmIOTalonFX`): works for both real robot and simulation
2. **One empty replay IO class** (e.g., `ArmIOReplay`): for log replay only

Compare this to traditional AdvantageKit which requires **3 IO classes** (Real, Sim, Replay).
{% endhint %}

## Pattern Summary: Mechanism Classes with SmartMotorController Telemetry

The examples above demonstrate the recommended pattern for using YAMS with AdvantageKit:

1. **Create Mechanism classes** (`Arm`, `Elevator`, `FlyWheel`, etc.) for high-level control
2. **Get the underlying `SmartMotorController`** via `mechanism.getMotor()`
3. **Pull telemetry from `SmartMotorController`** for AdvantageKit logging
4. **Use Mechanism methods** like `setMechanismPositionSetpoint()` for control
5. **Optionally expose the Mechanism** via a getter for command helpers like `run()` and `runTo()`

This pattern gives you the best of both worlds:

* **Mechanism classes** provide convenient configuration and command helpers
* **SmartMotorController** provides detailed telemetry for AdvantageKit logging
* **Passive simulation** works automatically in both

{% hint style="info" %}
See the YAMS AdvantageKit examples for complete robot projects that log YAMS mechanisms with AdvantageKit: [Commands v2](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands2/advantage_kit) and [Commands v3](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands3/advantage_kit).
{% endhint %}

### Using WPILib's isNear() for Tolerance Checking

When checking if a mechanism is at its target, use the `isNear()` method from WPILib's unit classes (`Angle`, `Distance`, `AngularVelocity`):

```java
// In your subsystem, check if position is near target
Angle currentPosition = Rotations.of(inputs.positionRotations);
Angle targetPosition = Rotations.of(targetRotations);
Angle tolerance = Rotations.of(0.01);

boolean atTarget = currentPosition.isNear(targetPosition, tolerance);
```

This approach works with any unit type and handles conversions automatically.

To get a `Trigger` instead of a `boolean`, build it from the logged value so replay sees the same result. This is what the Commands v3 AdvantageKit example does in `ArmMechanism`:

```java
public Trigger near(Angle angle, Angle within) {
  return new Trigger(() -> getAngle().isNear(angle, within));
}
```

{% hint style="info" %}
YAMS mechanisms have their own `near(...)` Trigger factory (renamed from `isNear(...)`, the boolean check is still `isNear(...)`). Those read the live `SmartMotorController`, not your logged inputs, so prefer the logged value when the result must be replayed.
{% endhint %}

## Swerve Drive with AdvantageKit

YAMS swerve integrates seamlessly with AdvantageKit. Here's a complete example based on the official YAMS AdvantageKit swerve examples ([Commands v2 `SwerveSubsystem`](https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/advantage_kit/java/first/robot/subsystems/SwerveSubsystem.java), [Commands v3 `SwerveMechanism`](https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands3/advantage_kit/java/first/robot/mechanisms/SwerveMechanism.java)):

{% hint style="info" %}
WPILib 2027 renamed the swerve kinematics types: `ChassisSpeeds` is now `ChassisVelocities` and `SwerveModuleState` is now `SwerveModuleVelocity`. The YAMS swerve API uses the new names.
{% endhint %}

```java
package first.robot.subsystems;

import static org.wpilib.units.Units.*;

import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.CANcoder;
import com.ctre.phoenix6.hardware.Pigeon2;
import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;
import com.revrobotics.util.CANPorts;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.PIDController;
import org.wpilib.math.geometry.Pose2d;
import org.wpilib.math.geometry.Rotation2d;
import org.wpilib.math.geometry.Rotation3d;
import org.wpilib.math.geometry.Translation2d;
import org.wpilib.math.kinematics.ChassisVelocities;
import org.wpilib.math.kinematics.SwerveModulePosition;
import org.wpilib.math.kinematics.SwerveModuleVelocity;
import org.wpilib.math.system.DCMotor;
import org.wpilib.units.measure.Angle;
import org.wpilib.units.measure.AngularVelocity;
import org.wpilib.units.measure.LinearVelocity;
import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import java.util.function.DoubleSupplier;
import java.util.function.Supplier;
import org.littletonrobotics.junction.AutoLog;
import org.littletonrobotics.junction.Logger;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.commands2.config.SwerveDriveConfig;
import yams.commands2.swerve.SwerveDrive;
import yams.commands2.swerve.SwerveInputStream;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.SwerveModuleConfig;
import yams.core.mechanisms.swerve.SwerveModule;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.local.SparkWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class SwerveSubsystem extends SubsystemBase {

  private AngularVelocity maxAngularVelocity = DegreesPerSecond.of(720);
  private LinearVelocity maxLinearVelocity = MetersPerSecond.of(4);
  private Supplier<Angle> gyroAngleSupplier;
  private SwerveDriveConfig config;

  /** AdvantageKit inputs for the swerve drive */
  @AutoLog
  public static class SwerveInputs {
    public SwerveModulePosition[] positions = new SwerveModulePosition[4];
    public SwerveModuleVelocity[] states = new SwerveModuleVelocity[4];
    public Angle gyroRotation = Degrees.of(0);
    public ChassisVelocities robotRelativeSpeeds = new ChassisVelocities(0, 0, 0);
    public Pose2d estimatedPose = new Pose2d(0, 0, Rotation2d.fromDegrees(0));
  }

  private final SwerveInputsAutoLogged swerveInputs = new SwerveInputsAutoLogged();
  private final SwerveDrive drive;

  public SwerveModule createModule(SparkMax driveMotor, SparkMax azimuthMotor,
      CANcoder absoluteEncoder, String moduleName, Translation2d location) {

    MechanismGearing driveGearing = new MechanismGearing(GearBox.fromStages("12:1", "2:1"));
    MechanismGearing azimuthGearing = new MechanismGearing(GearBox.fromStages("21:1"));

    SmartMotorControllerConfig driveCfg = new SmartMotorControllerConfig(this)
        .withWheelDiameter(Inches.of(4))
        .withClosedLoopController(50, 0, 4)
        .withGearing(driveGearing)
        .withStatorCurrentLimit(Amps.of(40))
        .withTelemetry("driveMotor", TelemetryVerbosity.HIGH);

    SmartMotorControllerConfig azimuthCfg = new SmartMotorControllerConfig(this)
        .withClosedLoopController(50, 0, 4)
        .withContinuousWrapping(Radians.of(-Math.PI), Radians.of(Math.PI))
        .withGearing(azimuthGearing)
        .withStatorCurrentLimit(Amps.of(20))
        .withTelemetry("angleMotor", TelemetryVerbosity.HIGH);

    SmartMotorController driveSMC = new SparkWrapper(driveMotor, DCMotor.getNEO(1), driveCfg);
    SmartMotorController azimuthSMC = new SparkWrapper(azimuthMotor, DCMotor.getNEO(1), azimuthCfg);

    SwerveModuleConfig moduleConfig = new SwerveModuleConfig(driveSMC, azimuthSMC)
        .withAbsoluteEncoder(absoluteEncoder.getAbsolutePosition().asSupplier())
        .withTelemetry(moduleName, TelemetryVerbosity.HIGH)
        .withLocation(location)
        .withOptimization(true);

    return new SwerveModule(moduleConfig);
  }

  public SwerveInputStream getChassisSpeedsSupplier(DoubleSupplier x, DoubleSupplier y,
      DoubleSupplier rot) {
    return SwerveInputStream.of(drive, x, y, rot)
        .withMaximumAngularVelocity(maxAngularVelocity)
        .withMaximumLinearVelocity(maxLinearVelocity)
        .withDeadband(0.01)
        .withCubeRotationControllerAxis()
        .withCubeTranslationControllerAxis()
        .withAllianceRelativeControl();
  }

  public SwerveSubsystem() {
    Pigeon2 gyro = new Pigeon2(14, new CANBus(CANPort.CAN_S0));
    gyroAngleSupplier = gyro.getYaw().asSupplier();

    var fl = createModule(new SparkMax(CANPorts.fromBusId(1), 1, MotorType.kBrushless),
        new SparkMax(CANPorts.fromBusId(1), 2, MotorType.kBrushless),
        new CANcoder(3, new CANBus(CANPort.CAN_S0)),
        "frontleft", new Translation2d(Inches.of(12), Inches.of(12)));
    var fr = createModule(new SparkMax(CANPorts.fromBusId(1), 4, MotorType.kBrushless),
        new SparkMax(CANPorts.fromBusId(1), 5, MotorType.kBrushless),
        new CANcoder(6, new CANBus(CANPort.CAN_S0)),
        "frontright", new Translation2d(Inches.of(12), Inches.of(-12)));
    var bl = createModule(new SparkMax(CANPorts.fromBusId(1), 7, MotorType.kBrushless),
        new SparkMax(CANPorts.fromBusId(1), 8, MotorType.kBrushless),
        new CANcoder(9, new CANBus(CANPort.CAN_S0)),
        "backleft", new Translation2d(Inches.of(-12), Inches.of(12)));
    var br = createModule(new SparkMax(CANPorts.fromBusId(1), 10, MotorType.kBrushless),
        new SparkMax(CANPorts.fromBusId(1), 11, MotorType.kBrushless),
        new CANcoder(12, new CANBus(CANPort.CAN_S0)),
        "backright", new Translation2d(Inches.of(-12), Inches.of(-12)));

    config = new SwerveDriveConfig(this, fl, fr, bl, br)
        // The gyro supplier returns a Rotation3d built from the LOGGED yaw, so replay is deterministic
        .withGyro(() -> new Rotation3d(getGyroAngle()))
        .withStartingPose(swerveInputs.estimatedPose)
        .withTranslationController(new PIDController(1, 0, 0))
        .withRotationController(new PIDController(1, 0, 0));

    drive = new SwerveDrive(config);
  }

  private Rotation2d getGyroAngle() {
    return new Rotation2d(swerveInputs.gyroRotation);
  }

  private void updateInputs() {
    swerveInputs.estimatedPose = drive.getPose();
    swerveInputs.states = drive.getModuleStates();
    swerveInputs.positions = drive.getModulePositions();
    swerveInputs.robotRelativeSpeeds = drive.getRobotRelativeSpeed();
    swerveInputs.gyroRotation = gyroAngleSupplier.get();
  }

  /** Drive with field relative speeds (e.g. from a SwerveInputStream) using the logged gyro angle. */
  public Command setFieldRelativeChassisSpeeds(Supplier<ChassisVelocities> fieldRelativeSpeeds) {
    return run(() -> setRobotRelativeChassisSpeedsSetpoint(
        fieldRelativeSpeeds.get().toRobotRelative(getGyroAngle())))
        .withName("Set Field Relative Chassis Speeds Supplier");
  }

  private void setRobotRelativeChassisSpeedsSetpoint(ChassisVelocities speeds) {
    Logger.recordOutput("Swerve/DesiredChassisSpeeds", speeds);
    SwerveModuleVelocity[] states = drive.getStateFromRobotRelativeChassisSpeeds(speeds);
    Logger.recordOutput("Swerve/DesiredStates", states);
    drive.setSwerveModuleStates(states);
  }

  /** PID drive to a pose using the logged pose and gyro angle, so replay follows the same path. */
  public Command driveToPose(Pose2d pose) {
    return startRun(() -> {
      drive.resetTranslationPID();
      drive.resetRotationPID();
    }, () -> drive.setRobotRelativeChassisSpeeds(
        drive.driveToPoseSetpoint(pose, getPose(), getGyroAngle())))
        .withName("Drive to Pose");
  }

  /** Point every module at its corner (X pattern) with zero drive speed, logging the desired states. */
  public Command lock() {
    return run(() -> {
      SwerveModule[] modules = config.getModules();
      SwerveModuleVelocity[] desiredStates = new SwerveModuleVelocity[modules.length];
      for (int i = 0; i < modules.length; i++) {
        desiredStates[i] = new SwerveModuleVelocity(0,
            modules[i].getConfig().getLocation().orElseThrow().getAngle().orElse(Rotation2d.ZERO));
      }
      Logger.recordOutput("Swerve/DesiredStates", desiredStates);
      drive.setSwerveModuleStates(desiredStates);
    }).withName("Lock");
  }

  public Pose2d getPose() { return swerveInputs.estimatedPose; }
  public ChassisVelocities getRobotRelativeSpeeds() { return swerveInputs.robotRelativeSpeeds; }

  @Override
  public void periodic() {
    drive.updateTelemetry();  // Updates pose estimator, call BEFORE updateInputs
    updateInputs();
    Logger.processInputs("Swerve", swerveInputs);
  }

  @Override
  public void simulationPeriodic() {
    drive.simIterate();
  }
}
```

The `SwerveInputStream` output is field relative, so bind it through `setFieldRelativeChassisSpeeds`. `withTelemetry(...)` on the stream publishes its mode and configuration, and at `HIGH` adds a `Live Tuning` command (see [SwerveInputStream](swerve-input-stream.md)):

```java
// In RobotContainer
drive.setDefaultCommand(drive.setFieldRelativeChassisSpeeds(
    drive.getChassisSpeedsSupplier(() -> -xboxController.getLeftY(),
                                   () -> -xboxController.getLeftX(),
                                   () -> -xboxController.getRightX())
         .withTelemetry("Driver", TelemetryVerbosity.HIGH)));
```

{% hint style="warning" %}
`SwerveDrive.lockPose()` turns every module to its corner (the X pattern) with zero drive speed, but it is marked as not compatible with AdvantageKit. The YAMS AdvantageKit example builds the same X pattern itself in `lock()` and sends it with `setSwerveModuleStates()`, so the desired states are recorded with `Logger.recordOutput()` like every other drive command.
{% endhint %}

### Commands v3

The [Commands v3 AdvantageKit example](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands3/advantage_kit) uses the same inputs class and module setup, with these differences:

* The class implements `org.wpilib.command3.Mechanism` instead of extending `SubsystemBase`, and imports `yams.commands3.config.SmartMotorControllerConfig`, `yams.commands3.config.SwerveDriveConfig`, `yams.commands3.swerve.SwerveDrive` and `yams.commands3.swerve.SwerveInputStream`.
* Commands are coroutines (`while (true)` + `coroutine.yield()`).
* `periodic()` and `simulationPeriodic()` are plain methods that `Robot.robotPeriodic()` and `Robot.simulationPeriodic()` call before `scheduler.run()`.
* The teleop opmode creates the `SwerveInputStream` and hands it to the drive mechanism.

```java
// SwerveMechanism
private SwerveInputStream inputStream;

public Command driveInputStream() {
  return run(coroutine -> {
    while (true) {
      setFieldRelativeChassisSpeeds(inputStream.get());
      coroutine.yield();
    }
  }).named("Swerve Drive Input Stream");
}

public void setInputStream(SwerveInputStream inputStream) {
  // Stop publishing the old stream's telemetry before replacing it
  if (this.inputStream != null) {
    this.inputStream.getTelemetry().ifPresent(SwerveInputStreamTelemetry::close);
  }
  this.inputStream = inputStream;
}

public SwerveDrive getSwerveDrive() {
  return drive;
}
```

```java
// AngularVelocityTeleop, created from Robot.teleopInit()
robot.drive.setInputStream(SwerveInputStream.of(
        robot.drive.getSwerveDrive(),
        () -> -controller.getLeftY(),
        () -> -controller.getLeftX(),
        () -> -controller.getRightX())
    .withDeadband(0.01)
    .withCubeRotationControllerAxis()
    .withCubeTranslationControllerAxis()
    .withAllianceRelativeControl()
    .withTelemetry("Driver", TelemetryVerbosity.HIGH));
robot.drive.setDefaultCommand(robot.drive.driveInputStream());
```

### Key Points for Swerve with AdvantageKit

1. **Create an `@AutoLog` inputs class** with module states, positions, gyro, and pose
2. **Call `updateTelemetry()` BEFORE `updateInputs()`**: the pose estimator and internal state update in `updateTelemetry()`
3. **Feed the gyro from the logged yaw** (`withGyro(() -> new Rotation3d(getGyroAngle()))`) so odometry is deterministic in replay
4. **Log desired states and speeds** using `Logger.recordOutput()` for replay debugging
5. **Call `simIterate()` in `simulationPeriodic()`** for physics simulation

{% hint style="warning" %}
**Important: Replay Data Scope**

During AdvantageKit log replay, **only data inside the `@AutoLog` inputs classes is mutated**. YAMS's built-in telemetry (published via `withTelemetry()`) is NOT affected by replay: it will show live simulation values, not replayed values.

This means:

* **Inputs you define** (in `SwerveInputs`, `ArmInputs`, etc.): replayed correctly
* **YAMS telemetry** (from `.withTelemetry("Arm")`): shows live sim values during replay

To ensure accurate replay debugging, always pull the data you need into your `@AutoLog` inputs class rather than relying on YAMS telemetry during replay.
{% endhint %}

## Alternative Pattern: Direct Subsystem (No IO Layer)

For simpler projects, you can use YAMS mechanisms directly as shown in the official examples ([Commands v2 subsystems](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands2/advantage_kit/java/first/robot/subsystems), [Commands v3 mechanisms](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands3/advantage_kit/java/first/robot/mechanisms)):

{% tabs %}
{% tab title="Commands v2" %}
```java
package first.robot.subsystems;

import static org.wpilib.units.Units.*;

import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.TalonFX;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.ArmFeedforward;
import org.wpilib.math.system.DCMotor;
import org.wpilib.units.measure.Angle;
import org.wpilib.units.measure.AngularVelocity;
import org.wpilib.units.measure.Current;
import org.wpilib.units.measure.Voltage;
import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import org.littletonrobotics.junction.AutoLog;
import org.littletonrobotics.junction.Logger;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.commands2.mechanisms.Arm;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.ArmConfig;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.remote.TalonFXWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class ArmSubsystem extends SubsystemBase {

  @AutoLog
  public static class ArmInputs {
    public Angle pivotPosition = Degrees.of(0);
    public AngularVelocity pivotVelocity = DegreesPerSecond.of(0);
    public Voltage pivotAppliedVolts = Volts.of(0);
    public Current pivotCurrent = Amps.of(0);
  }

  private final ArmInputsAutoLogged armInputs = new ArmInputsAutoLogged();
  private final SmartMotorController armSMC;
  private final Arm arm;

  public ArmSubsystem() {
    SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
        .withControlMode(ControlMode.CLOSED_LOOP)
        .withClosedLoopController(18, 0, 0.2)
        .withTrapezoidalProfile(DegreesPerSecond.of(458), DegreesPerSecondPerSecond.of(688))
        .withFeedforward(new ArmFeedforward(-0.1, 1.2, 0, 0))
        .withGearing(new MechanismGearing(GearBox.fromReductionStages(12.5, 1)))
        .withZeroPower(MotorMode.BRAKE)
        .withStatorCurrentLimit(Amps.of(120))
        .withStartingPosition(Degrees.of(141));

    armSMC = new TalonFXWrapper(new TalonFX(40, new CANBus(CANPort.CAN_S0)), DCMotor.getFalcon500(1), smcConfig);

    ArmConfig armCfg = new ArmConfig()
        .withHardLimits(Degrees.of(-25), Degrees.of(141))
        .withLength(Feet.of(14.0 / 12))
        .withTelemetry("Arm", TelemetryVerbosity.HIGH);

    arm = new Arm(armCfg, armSMC);
  }

  public void updateInputs() {
    armInputs.pivotPosition = arm.getAngle();
    armInputs.pivotVelocity = armSMC.getMechanismVelocity();
    armInputs.pivotAppliedVolts = armSMC.getVoltage();
    armInputs.pivotCurrent = armSMC.getStatorCurrent();
  }

  public Command setAngle(Angle angle) {
    return arm.setAngle(angle);
  }

  @Override
  public void periodic() {
    updateInputs();
    Logger.processInputs("Arm", armInputs);
    arm.updateTelemetry();
  }

  @Override
  public void simulationPeriodic() {
    arm.simIterate();
  }
}
```
{% endtab %}

{% tab title="Commands v3" %}
In Commands v3 the class implements `Mechanism`, imports the `yams.commands3` bindings, and `Robot` calls `periodic()` and `simulationPeriodic()` itself. The inputs, configuration and `updateInputs()` are the same as the Commands v2 tab.

```java
package first.robot.mechanisms;

// Same static units, CTRE, WPILib math/units, AdvantageKit and yams.core imports as Commands v2, plus:
import org.wpilib.command3.Command;
import org.wpilib.command3.Mechanism;
import org.wpilib.command3.Trigger;
import yams.commands3.config.SmartMotorControllerConfig;
import yams.commands3.mechanisms.Arm;

public class ArmMechanism implements Mechanism {

  // ArmInputs, armInputs, smcConfig, armSMC, armCfg, arm and updateInputs() as in Commands v2

  public Command setAngle(Angle angle) {
    return arm.setAngle(angle);
  }

  /** Trigger built from the logged angle, so it replays correctly. */
  public Trigger near(Angle angle, Angle within) {
    return new Trigger(() -> armInputs.pivotPosition.isNear(angle, within));
  }

  public void periodic() {
    updateInputs();
    Logger.processInputs("Arm", armInputs);
    arm.updateTelemetry();
  }

  public void simulationPeriodic() {
    arm.simIterate();
  }
}
```

```java
// Robot.java (extends LoggedRobot)
@Override
public void robotPeriodic() {
  drive.periodic();
  arm.periodic();
  elevator.periodic();
  shooter.periodic();
  indexer.periodic();
  scheduler.run();
}

@Override
public void simulationPeriodic() {
  drive.simulationPeriodic();
  arm.simulationPeriodic();
  elevator.simulationPeriodic();
  shooter.simulationPeriodic();
  indexer.simulationPeriodic();
}
```
{% endtab %}
{% endtabs %}

This pattern is simpler and **does support log replay**, but with a key difference: during replay, the full YAMS simulation runs alongside the logged data. This means:

* **Pro**: You get full simulation behavior during replay, which can help debug physics-related issues
* **Con**: Replay requires more processing power since every `SmartMotorController` and Mechanism simulates its physics

Choose the IO layer pattern if you need lightweight replay; choose this direct pattern if you prefer simplicity and don't mind the extra processing during replay.

## Benefits of YAMS + AdvantageKit

| Feature                     | Benefit                                                   |
| --------------------------- | --------------------------------------------------------- |
| **No Duplicate IO Classes** | Single IO class works for real robot AND simulation       |
| **Deterministic Replay**    | Debug issues from real matches using log replay           |
| **Automatic Physics**       | YAMS uses DCMotor, gearing, and mass/MOI for accurate sim |
| **Less Boilerplate**        | 2 IO classes instead of 3 per mechanism                   |
| **Consistent Behavior**     | Same code paths execute in real and sim modes             |
| **Flexible Hardware**       | Swap vendors by changing one IO class                     |

## Further Reading

* [AdvantageKit Documentation](https://docs.advantagekit.org/)
* [What is AdvantageKit?](https://docs.advantagekit.org/getting-started/what-is-advantagekit)
* [IO Interfaces](https://docs.advantagekit.org/data-flow/recording-inputs/io-interfaces)
* [Template Projects](https://docs.advantagekit.org/getting-started/template-projects)
* [YAMS AdvantageKit Example (Commands v2)](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands2/advantage_kit)
* [YAMS AdvantageKit Example (Commands v3)](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands3/advantage_kit)
