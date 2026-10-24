# How do I set an arm's target angle?

An `Arm`'s position setpoint is an `Angle` expressed in WPILib's typed unit system (`Angle`, `org.wpilib.units.measure.Angle`). Understanding the difference between `run()`, which tracks the setpoint continuously, and `runTo()`, which ends the command once the setpoint is reached within tolerance, shapes how you bind controls in `RobotContainer`.

<figure><img src="../.gitbook/assets/howto-arm-run-vs-runto.gif" alt="Animation of the same arm moving to 45 degrees with two commands, with a default command of run to 0 degrees. With run the command keeps running after the arm arrives, and the arm stays at 45 degrees. With runTo and a tolerance of 3 degrees the command ends once the arm has stayed inside the tolerance for 0.1 seconds, then the default command takes over and moves the arm back to 0 degrees."><figcaption><p><code>run()</code> keeps running until something interrupts it. <code>runTo()</code> ends at the setpoint, and the default command takes over.</p></figcaption></figure>

## Create `Command`s with our `Arm`

We use the `Arm` class as a interface to create commands!

{% tabs %}
{% tab title="Commands v2" %}
<pre class="language-java" data-title="ExampleSubsystem.java" data-line-numbers><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Degrees;
import static org.wpilib.units.Units.Feet;
import static org.wpilib.units.Units.Pounds;
<strong>import static org.wpilib.units.Units.Seconds;
</strong>
import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;

import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.ArmFeedforward;
import org.wpilib.math.system.DCMotor;
<strong>import org.wpilib.units.measure.Angle;
</strong><strong>import org.wpilib.units.measure.Voltage;
</strong>import yams.commands2.config.SmartMotorControllerConfig;
import yams.commands2.mechanisms.Arm;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.ArmConfig;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.local.SparkWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class ExampleSubsystem extends SubsystemBase {

  private SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Feedback Constants (PID Constants)
  .withClosedLoopController(50, 0, 0)
  .withSimClosedLoopController(50, 0, 0)
  // Feedforward Constants
  .withFeedforward(new ArmFeedforward(0, 0, 0))
  .withSimFeedforward(new ArmFeedforward(0, 0, 0))
  // Telemetry name and verbosity level
  .withTelemetry("ArmMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example gearbox(3,4) is the same as gearbox("3:1","4:1") which corresponds to the gearbox attached to your motor.
  .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
  // Soft limit is applied to the SmartMotorControllers PID
  .withSoftLimits(Degrees.of(-20), Degrees.of(10))
  // Starting position is where your arm starts
  .withStartingPosition(Degrees.of(-5))
  // Length and mass of your arm, used to estimate the moment of inertia for sim.
  .withMomentOfInertia(Feet.of(3), Pounds.of(8))
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

  private ArmConfig armCfg = new ArmConfig()
  // Hard limit is applied to the simulation.
  .withHardLimits(Degrees.of(-30), Degrees.of(40))
  // Length of your arm for sim.
  .withLength(Feet.of(3))
  // Telemetry name and verbosity for the arm.
  .withTelemetry("Arm", TelemetryVerbosity.HIGH);

  // Arm Mechanism
  private Arm arm = new Arm(armCfg, sparkSmartMotorController);

<strong>  /**
</strong><strong>   * Run the arm to an angle, does not stop when reached.
</strong><strong>   * @param angle Angle to go to.
</strong><strong>   */
</strong><strong>  public Command run(Angle angle) { return arm.run(angle);}
</strong>  
<strong>  /**
</strong><strong>   * Run the arm to an angle, stops when reached.
</strong><strong>   * @param angle Angle to go to.
</strong><strong>   * @param tolerance Angle tolerance for completion.
</strong><strong>   */
</strong><strong>  public Command runTo(Angle angle, Angle tolerance) { return arm.runTo(angle, tolerance);}
</strong>  
<strong>  /**
</strong><strong>   * Set the angle of the arm.
</strong><strong>   * @param angle Angle to go to.
</strong><strong>   */
</strong><strong>  public void setAngleSetpoint(Angle angle) { arm.setMechanismPositionSetpoint(angle); }
</strong>
<strong>  /**
</strong><strong>   * Move the arm up and down.
</strong><strong>   * @param dutycycle [-1, 1] speed to set the arm too.
</strong><strong>   */
</strong><strong>  public Command set(double dutycycle) { return arm.set(dutycycle);}
</strong>
<strong>  /**
</strong><strong>   * Set the voltage of the arm, useful for finding its feedforward gains.
</strong><strong>   * @param volts Voltage to apply.
</strong><strong>   */
</strong><strong>  public Command setVoltage(Voltage volts) { return arm.setVoltage(volts);}
</strong>
  /** Creates a new ExampleSubsystem. */
  public ExampleSubsystem() {}

  @Override
  public void periodic() {
    // This method will be called once per scheduler run
    arm.updateTelemetry();
  }

  @Override
  public void simulationPeriodic() {
    // This method will be called once per scheduler run during simulation
    arm.simIterate();
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
import static org.wpilib.units.Units.Degrees;
import static org.wpilib.units.Units.Feet;
import static org.wpilib.units.Units.Pounds;
import static org.wpilib.units.Units.Seconds;

import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;

import org.wpilib.command3.Command;
import org.wpilib.command3.Mechanism;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.ArmFeedforward;
import org.wpilib.math.system.DCMotor;
import org.wpilib.units.measure.Angle;
import org.wpilib.units.measure.Voltage;
import yams.commands3.config.SmartMotorControllerConfig;
import yams.commands3.mechanisms.Arm;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.ArmConfig;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.local.SparkWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class ExampleMechanism implements Mechanism {

  private SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Feedback Constants (PID Constants)
  .withClosedLoopController(50, 0, 0)
  .withSimClosedLoopController(50, 0, 0)
  // Feedforward Constants
  .withFeedforward(new ArmFeedforward(0, 0, 0))
  .withSimFeedforward(new ArmFeedforward(0, 0, 0))
  // Telemetry name and verbosity level
  .withTelemetry("ArmMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example gearbox(3,4) is the same as gearbox("3:1","4:1") which corresponds to the gearbox attached to your motor.
  .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
  // Soft limit is applied to the SmartMotorControllers PID
  .withSoftLimits(Degrees.of(-20), Degrees.of(10))
  // Starting position is where your arm starts
  .withStartingPosition(Degrees.of(-5))
  // Length and mass of your arm, used to estimate the moment of inertia for sim.
  .withMomentOfInertia(Feet.of(3), Pounds.of(8))
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

  private ArmConfig armCfg = new ArmConfig()
  // Hard limit is applied to the simulation.
  .withHardLimits(Degrees.of(-30), Degrees.of(40))
  // Length of your arm for sim.
  .withLength(Feet.of(3))
  // Telemetry name and verbosity for the arm.
  .withTelemetry("Arm", TelemetryVerbosity.HIGH);

  // Arm Mechanism
  private Arm arm = new Arm(armCfg, sparkSmartMotorController);

  /**
   * Run the arm to an angle, does not stop when reached.
   * @param angle Angle to go to.
   */
  public Command run(Angle angle) { return arm.run(angle);}
  
  /**
   * Run the arm to an angle, stops when reached.
   * @param angle Angle to go to.
   * @param tolerance Angle tolerance for completion.
   */
  public Command runTo(Angle angle, Angle tolerance) { return arm.runTo(angle, tolerance);}
  
  /**
   * Set the angle of the arm.
   * @param angle Angle to go to.
   */
  public void setAngleSetpoint(Angle angle) { arm.setMechanismPositionSetpoint(angle); }

  /**
   * Move the arm up and down.
   * @param dutycycle [-1, 1] speed to set the arm too.
   */
  public Command set(double dutycycle) { return arm.set(dutycycle);}

  /**
   * Set the voltage of the arm, useful for finding its feedforward gains.
   * @param volts Voltage to apply.
   */
  public Command setVoltage(Voltage volts) { return arm.setVoltage(volts);}

  /**
   * Hold the arm at an angle at the lowest priority, so any other command can take over.
   * Meant for the default command.
   * @param angle Angle to hold.
   */
  public Command hold(Angle angle) {
    return run(coroutine -> coroutine.await(arm.run(angle)))
        .withPriority(Command.LOWEST_PRIORITY)
        .named("Arm Hold");
  }

  /** Creates a new ExampleMechanism. */
  public ExampleMechanism() {}

  public void periodic() {
    // A Commands v3 Mechanism has no periodic hook: call this from Robot.robotPeriodic()
    arm.updateTelemetry();
  }

  public void simulationPeriodic() {
    // A Commands v3 Mechanism has no periodic hook: Robot.simulationInit() runs this in its own Notifier
    arm.simIterate();
  }
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

## Bind buttons to our `Arm`

We bind buttons to use the `Commands` from our `Arm`

{% tabs %}
{% tab title="Commands v2" %}
<pre class="language-java" data-title="RobotContainer.java" data-line-numbers><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot;

import first.robot.subsystems.ExampleSubsystem;

<strong>import static org.wpilib.units.Units.Degrees;
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

<strong>    // Set the default command to force the arm to go to 0.
</strong><strong>    m_exampleSubsystem.setDefaultCommand(m_exampleSubsystem.run(Degrees.of(0)));
</strong>  }

  /** Use this method to define your trigger->command mappings. */
  private void configureBindings() {
    
<strong>    // Schedule `run` when the Xbox controller's A or B button is pressed,
</strong><strong>    // cancelling on release.
</strong><strong>    m_driverController.a().whileTrue(m_exampleSubsystem.run(Degrees.of(-5)));
</strong><strong>    m_driverController.b().whileTrue(m_exampleSubsystem.run(Degrees.of(15)));
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

import static org.wpilib.units.Units.Degrees;

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
    // Set the default command to force the arm to go to 0.
    exampleMechanism.setDefaultCommand(exampleMechanism.hold(Degrees.of(0)));
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

import static org.wpilib.units.Units.Degrees;

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
    controller.a().whileTrue(robot.exampleMechanism.run(Degrees.of(-5)));
    controller.b().whileTrue(robot.exampleMechanism.run(Degrees.of(15)));

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

{% hint style="info" %}
**Using Commands v3?** Use the Commands v3 tabs above. A v3 `Mechanism` has no `periodic()` hook, so `Robot.robotPeriodic()` calls `updateTelemetry()` and `simIterate()` through the mechanism, and the button bindings live in an opmode instead of a `RobotContainer`. See `examples/commands3/simple_robot-2027` in the YAMS repo for a complete robot.
{% endhint %}
