# How to set an Elevator's target Height?

`Elevator` setpoints are expressed as `Distance` values (meters, inches, etc.). The motion profile limits how fast the elevator accelerates to the target, so commanding a large height change won't slam the carriage into the top hard stop if your profile parameters are tuned correctly.

## Create and configure our `Elevator`

Our `Elevator` will easily configure the `SmartMotorController` and create a simple and intuitive interface.

{% tabs %}
{% tab title="Commands v2" %}
<pre class="language-java" data-title="ExampleSubsystem.java" data-line-numbers><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.Pounds;
import static org.wpilib.units.Units.Seconds;

import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;

import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.ElevatorFeedforward;
import org.wpilib.math.system.DCMotor;
import yams.commands2.config.SmartMotorControllerConfig;
<strong>import yams.commands2.mechanisms.Elevator;
</strong>import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
<strong>import yams.core.mechanisms.config.ElevatorConfig;
</strong>import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.local.SparkWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class ExampleSubsystem extends SubsystemBase {

  private SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Drum radius is required for elevators. Chain-driven: specify chain pitch and tooth count.
  .withDrumRadius(Inches.of(0.25), 22)
  // Feedback Constants (PID Constants)
  .withClosedLoopController(4, 0, 0)
  .withSimClosedLoopController(4, 0, 0)
  // Feedforward Constants
  .withFeedforward(new ElevatorFeedforward(0, 0, 0))
  .withSimFeedforward(new ElevatorFeedforward(0, 0, 0))
  // Telemetry name and verbosity level
  .withTelemetry("ElevatorMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example gearbox(3,4) is the same as gearbox("3:1","4:1") which corresponds to the gearbox attached to your motor.
  .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
  // Starting height of the carriage, required for simulation.
  .withStartingPosition(Meters.of(0))
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
{% endtab %}

{% tab title="Commands v3" %}
{% code title="ExampleMechanism.java" lineNumbers="true" %}
```java
// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.mechanisms;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.Pounds;
import static org.wpilib.units.Units.Seconds;

import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;

import org.wpilib.command3.Command;
import org.wpilib.command3.Mechanism;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.ElevatorFeedforward;
import org.wpilib.math.system.DCMotor;
import yams.commands3.config.SmartMotorControllerConfig;
import yams.commands3.mechanisms.Elevator;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.ElevatorConfig;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.local.SparkWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class ExampleMechanism implements Mechanism {

  private SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Drum radius is required for elevators. Chain-driven: specify chain pitch and tooth count.
  .withDrumRadius(Inches.of(0.25), 22)
  // Feedback Constants (PID Constants)
  .withClosedLoopController(4, 0, 0)
  .withSimClosedLoopController(4, 0, 0)
  // Feedforward Constants
  .withFeedforward(new ElevatorFeedforward(0, 0, 0))
  .withSimFeedforward(new ElevatorFeedforward(0, 0, 0))
  // Telemetry name and verbosity level
  .withTelemetry("ElevatorMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example gearbox(3,4) is the same as gearbox("3:1","4:1") which corresponds to the gearbox attached to your motor.
  .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
  // Starting height of the carriage, required for simulation.
  .withStartingPosition(Meters.of(0))
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

  /** Creates a new ExampleMechanism. */
  public ExampleMechanism() {}

  /**
   * Example command factory method.
   *
   * @return a command
   */
  public Command exampleMethodCommand() {
    // Inline construction of command goes here.
    // Mechanism::run requires `this` mechanism, and the command ends when the lambda returns.
    return run(
        coroutine -> {
          /* one-time action goes here */
        }).named("Example Method");
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

  public void periodic() {
    // A Commands v3 Mechanism has no periodic hook: call this from Robot.robotPeriodic()
    elevator.updateTelemetry();
  }

  public void simulationPeriodic() {
    // A Commands v3 Mechanism has no periodic hook: Robot.simulationInit() runs this in its own Notifier
    elevator.simIterate();
  }
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

## Create `Command`s with our `Elevator`

We use the `Elevator` class as a interface to create commands!

{% tabs %}
{% tab title="Commands v2" %}
<pre class="language-java" data-title="ExampleSubsystem.java" data-line-numbers><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.MetersPerSecond;
import static org.wpilib.units.Units.MetersPerSecondPerSecond;
import static org.wpilib.units.Units.Pounds;
import static org.wpilib.units.Units.Seconds;

import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;

import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.ElevatorFeedforward;
import org.wpilib.math.system.DCMotor;
<strong>import org.wpilib.units.measure.Distance;
</strong>import yams.commands2.config.SmartMotorControllerConfig;
import yams.commands2.mechanisms.Elevator;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.ElevatorConfig;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.local.SparkWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class ExampleSubsystem extends SubsystemBase {

  private SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Drum radius is required for elevators. Chain-driven: specify chain pitch and tooth count.
  .withDrumRadius(Inches.of(0.25), 22)
  // Feedback Constants (PID Constants)
  .withClosedLoopController(4, 0, 0)
  .withSimClosedLoopController(4, 0, 0)
  // Motion profile limits (max velocity and acceleration)
  .withTrapezoidalProfile(MetersPerSecond.of(0.5), MetersPerSecondPerSecond.of(0.5))
  // Feedforward Constants
  .withFeedforward(new ElevatorFeedforward(0, 0, 0))
  .withSimFeedforward(new ElevatorFeedforward(0, 0, 0))
  // Telemetry name and verbosity level
  .withTelemetry("ElevatorMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example gearbox(3,4) is the same as gearbox("3:1","4:1") which corresponds to the gearbox attached to your motor.
  .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
  // Starting height of the carriage, required for simulation.
  .withStartingPosition(Meters.of(0))
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
</strong><strong>   * Set the height of the elevator.
</strong><strong>   * @param height Distance to go to.
</strong><strong>   */
</strong><strong>  public Command run(Distance height) { return elevator.run(height);}
</strong>
<strong>  /**
</strong><strong>   * Move the elevator up and down.
</strong><strong>   * @param dutycycle [-1, 1] speed to set the elevator too.
</strong><strong>   */
</strong><strong>  public Command set(double dutycycle) { return elevator.set(dutycycle);}
</strong>
  /** Creates a new ExampleSubsystem. */
  public ExampleSubsystem() {}

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
{% endtab %}

{% tab title="Commands v3" %}
{% code title="ExampleMechanism.java" lineNumbers="true" %}
```java
// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.mechanisms;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.MetersPerSecond;
import static org.wpilib.units.Units.MetersPerSecondPerSecond;
import static org.wpilib.units.Units.Pounds;
import static org.wpilib.units.Units.Seconds;

import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;

import org.wpilib.command3.Command;
import org.wpilib.command3.Mechanism;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.ElevatorFeedforward;
import org.wpilib.math.system.DCMotor;
import org.wpilib.units.measure.Distance;
import yams.commands3.config.SmartMotorControllerConfig;
import yams.commands3.mechanisms.Elevator;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.ElevatorConfig;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.local.SparkWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class ExampleMechanism implements Mechanism {

  private SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Drum radius is required for elevators. Chain-driven: specify chain pitch and tooth count.
  .withDrumRadius(Inches.of(0.25), 22)
  // Feedback Constants (PID Constants)
  .withClosedLoopController(4, 0, 0)
  .withSimClosedLoopController(4, 0, 0)
  // Motion profile limits (max velocity and acceleration)
  .withTrapezoidalProfile(MetersPerSecond.of(0.5), MetersPerSecondPerSecond.of(0.5))
  // Feedforward Constants
  .withFeedforward(new ElevatorFeedforward(0, 0, 0))
  .withSimFeedforward(new ElevatorFeedforward(0, 0, 0))
  // Telemetry name and verbosity level
  .withTelemetry("ElevatorMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example gearbox(3,4) is the same as gearbox("3:1","4:1") which corresponds to the gearbox attached to your motor.
  .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
  // Starting height of the carriage, required for simulation.
  .withStartingPosition(Meters.of(0))
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

  /**
   * Set the height of the elevator.
   * @param height Distance to go to.
   */
  public Command run(Distance height) { return elevator.run(height);}

  /**
   * Move the elevator up and down.
   * @param dutycycle [-1, 1] speed to set the elevator too.
   */
  public Command set(double dutycycle) { return elevator.set(dutycycle);}

  /**
   * Hold the elevator at a height at the lowest priority, so any other command can take over.
   * Meant for the default command.
   * @param height Height to hold.
   */
  public Command hold(Distance height) {
    return run(coroutine -> coroutine.await(elevator.run(height)))
        .withPriority(Command.LOWEST_PRIORITY)
        .named("Elevator Hold");
  }

  /** Creates a new ExampleMechanism. */
  public ExampleMechanism() {}

  public void periodic() {
    // A Commands v3 Mechanism has no periodic hook: call this from Robot.robotPeriodic()
    elevator.updateTelemetry();
  }

  public void simulationPeriodic() {
    // A Commands v3 Mechanism has no periodic hook: Robot.simulationInit() runs this in its own Notifier
    elevator.simIterate();
  }
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

<figure><img src="../.gitbook/assets/mech-commands.png" alt="Three timelines starting when the command is scheduled. run(setpoint), also called setAngle or setHeight, runs until something else takes the mechanism and holds the setpoint the whole time. runTo(setpoint, tolerance) ends when the mechanism is within the tolerance, and then the default command takes over. set(dutycycle) and setVoltage(volts) are open loop with no PID, run until interrupted, and the closed loop controller comes back when they end."><figcaption><p>When each kind of command ends.</p></figcaption></figure>

## Bind buttons to our `Elevator`

We bind buttons to use the `Commands` from our `Elevator`

{% tabs %}
{% tab title="Commands v2" %}
<pre class="language-java" data-title="RobotContainer.java" data-line-numbers><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot;

import first.robot.subsystems.ExampleSubsystem;

<strong>import static org.wpilib.units.Units.Meters;
</strong>
import org.wpilib.command2.Command;
import org.wpilib.command2.Commands;
import org.wpilib.command2.button.CommandNiDsXboxController;

/**
 * This class is where the bulk of the robot should be declared. Since Command-based is a
 * "declarative" paradigm, very little robot logic should actually be handled in the {@link Robot}
 * periodic methods (other than the scheduler calls). Instead, the structure of the robot (including
 * subsystems, commands, and trigger mappings) should be declared here.
 */
public class RobotContainer {
  // The robot's subsystems and commands are defined here...
  private final ExampleSubsystem m_exampleSubsystem = new ExampleSubsystem();

  private final CommandNiDsXboxController m_driverController = new CommandNiDsXboxController(0);

  /** The container for the robot. Contains subsystems, OI devices, and commands. */
  public RobotContainer() {
    // Configure the trigger bindings
    configureBindings();

<strong>    // Set the default command to force the elevator to go to 0.
</strong><strong>    m_exampleSubsystem.setDefaultCommand(m_exampleSubsystem.run(Meters.of(0)));
</strong>  }

  /** Use this method to define your trigger->command mappings. */
  private void configureBindings() {
    
<strong>    // Schedule `run` when the Xbox controller's A or B button is pressed,
</strong><strong>    // cancelling on release.
</strong><strong>    m_driverController.a().whileTrue(m_exampleSubsystem.run(Meters.of(0.5)));
</strong><strong>    m_driverController.b().whileTrue(m_exampleSubsystem.run(Meters.of(1)));
</strong>
<strong>    // Schedule `set` when the Xbox controller's X or Y button is pressed,
</strong><strong>    // cancelling on release.
</strong><strong>    m_driverController.x().whileTrue(m_exampleSubsystem.set(0.3));
</strong><strong>    m_driverController.y().whileTrue(m_exampleSubsystem.set(-0.3));
</strong>
  }

  /**
   * Use this to pass the autonomous command to the main {@link Robot} class.
   *
   * @return the command to run in autonomous
   */
  public Command getAutonomousCommand() {
    return Commands.print("No autonomous command configured");
  }
}

</code></pre>
{% endtab %}

{% tab title="Commands v3" %}
{% code title="Robot.java" lineNumbers="true" %}
```java
// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot;

import first.robot.mechanisms.ExampleMechanism;

import static org.wpilib.units.Units.Meters;

import java.util.ArrayList;
import java.util.List;
import org.wpilib.command3.Scheduler;
import org.wpilib.command3.button.CommandNiDsXboxController;
import org.wpilib.framework.OpModeRobot;
import org.wpilib.system.Notifier;

/**
 * With Commands v3 there is no RobotContainer. The robot's mechanisms and controllers are declared
 * here, and the button bindings live in the opmodes.
 */
public class Robot extends OpModeRobot {
  // The robot's mechanisms are defined here...
  public final ExampleMechanism exampleMechanism = new ExampleMechanism();

  public final CommandNiDsXboxController driverController = new CommandNiDsXboxController(0);

  private final Scheduler scheduler = Scheduler.getDefault();
  // Each mechanism's simulation Notifier, kept so that it is not garbage collected.
  private final List<Notifier> simulationNotifiers = new ArrayList<>();

  public Robot() {
    // Set the default command to force the elevator to go to 0.
    exampleMechanism.setDefaultCommand(exampleMechanism.hold(Meters.of(0)));
  }

  @Override
  public void robotPeriodic() {
    // A Mechanism has no periodic hook, so update it before the scheduler runs.
    exampleMechanism.periodic();
    scheduler.run();
  }

  @Override
  public void simulationInit() {
    // Runs once, when the simulation starts.
    // Run each mechanism's simulation in its own Notifier, every robot loop.
    startSimulation("Example Mechanism", exampleMechanism::simulationPeriodic);
  }

  /** Runs a mechanism's simulation in its own Notifier, every robot loop period. */
  private void startSimulation(String name, Runnable simulationPeriodic) {
    Notifier notifier = new Notifier(simulationPeriodic);
    notifier.setName(name + " Simulation");
    notifier.startPeriodic(getPeriod());
    simulationNotifiers.add(notifier);
  }
}
```
{% endcode %}

{% code title="ExampleTeleop.java" lineNumbers="true" %}
```java
// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.opmodes;

import static org.wpilib.units.Units.Meters;

import first.robot.Robot;
import org.wpilib.opmode.OpMode;
import org.wpilib.opmode.Teleop;

/** Bindings made in an opmode's constructor only exist while that opmode runs. */
@Teleop
public class ExampleTeleop implements OpMode {
  public ExampleTeleop(Robot robot) {
    var controller = robot.driverController;

    // Schedule `run` when the Xbox controller's A or B button is pressed,
    // cancelling on release.
    controller.a().whileTrue(robot.exampleMechanism.run(Meters.of(0.5)));
    controller.b().whileTrue(robot.exampleMechanism.run(Meters.of(1)));

    // Schedule `set` when the Xbox controller's X or Y button is pressed,
    // cancelling on release.
    controller.x().whileTrue(robot.exampleMechanism.set(0.3));
    controller.y().whileTrue(robot.exampleMechanism.set(-0.3));
  }
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

## Simulate our Elevator

We can use our `Elevator` in simulation, with the exact same code that will control the real robot!

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

Connect an Xbox controller to your system and drag and drop from **System Joysticks** to **Joystick\[0]**

Open up the Simulated mechanism with **NetworkTables -> SmartDashboard -> Arm -> mechanism**

<figure><img src="../.gitbook/assets/Robot Simulation 8_30_2025 7_06_08 PM.png" alt=""><figcaption></figcaption></figure>

Resize **Elevator/mechanism** to your liking

Press **Teleoperated** in **Robot State** then you can use your controller like its controlling the real robot!

<figure><img src="../.gitbook/assets/Robot Simulation 2025-08-30 18-37-59.gif" alt=""><figcaption></figcaption></figure>

Congratulations on successfully programming your Elevator!! :tada::tada:

{% hint style="info" %}
**Using Commands v3?** Use the Commands v3 tabs above. A v3 `Mechanism` has no `periodic()` hook, so `Robot.robotPeriodic()` calls `updateTelemetry()` and `simIterate()` through the mechanism, and the button bindings live in an opmode instead of a `RobotContainer`. See `examples/commands3/simple_robot-2027` in the YAMS repo for a complete robot.
{% endhint %}
