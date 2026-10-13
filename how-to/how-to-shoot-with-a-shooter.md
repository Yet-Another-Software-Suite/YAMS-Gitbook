# How to shoot with a Shooter?

A flywheel shooter's effectiveness depends on reaching a stable velocity setpoint before the game piece enters the shooter. `FlyWheel` exposes a velocity trigger (`near`) so you can gate game piece injection on whether the wheel has spun up, rather than using a fixed time delay.

<figure><img src="../.gitbook/assets/howto-shooter-at-speed.gif" alt="Animation of a flywheel spinning up to 3000 RPM, with a band of 100 RPM around the setpoint. A feeder started by a fixed delay of 0.5 seconds feeds at 2252 RPM, before the wheel is ready. A feeder started by the near trigger feeds at 2900 RPM, once the wheel is inside the band."><figcaption><p>A fixed delay feeds whether or not the wheel is ready. The trigger waits for the wheel.</p></figcaption></figure>

## Create `Command`s with our `Shooter`

We use the `FlyWheel` class as a interface to create commands!

<pre class="language-java" data-title="ExampleSubsystem.java" data-line-numbers><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
<strong>import static org.wpilib.units.Units.Seconds;
</strong>
import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;

import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import org.wpilib.command2.button.Trigger;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import org.wpilib.math.system.DCMotor;
<strong>import org.wpilib.units.measure.AngularVelocity;
</strong>import yams.commands2.config.SmartMotorControllerConfig;
import yams.commands2.mechanisms.FlyWheel;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.mechanisms.config.FlyWheelConfig;
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
  .withFeedforward(new SimpleMotorFeedforward(0, 0, 0))
  .withSimFeedforward(new SimpleMotorFeedforward(0, 0, 0))
  // Telemetry name and verbosity level
  .withTelemetry("ShooterMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example gearbox(3,4) is the same as gearbox("3:1","4:1") which corresponds to the gearbox attached to your motor.
  .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
  // Motor properties to prevent over currenting.
  .withMotorInverted(false)
  .withZeroPower(MotorMode.COAST)
  .withStatorCurrentLimit(Amps.of(40))
  .withClosedLoopRampRate(Seconds.of(0.25))
  .withOpenLoopRampRate(Seconds.of(0.25));

  // Vendor motor controller object
  private SparkMax spark = new SparkMax(CANPort.CAN_S0, 4, MotorType.kBrushless);

  // Create our SmartMotorController from our Spark and config with the NEO.
  private SmartMotorController sparkSmartMotorController = new SparkWrapper(spark, DCMotor.getNEO(1), smcConfig);

 private final FlyWheelConfig shooterConfig = new FlyWheelConfig()
  // Diameter of the flywheel.
  .withDiameter(Inches.of(4))
  // Telemetry name and verbosity for the shooter.
  .withTelemetry("Shooter", TelemetryVerbosity.HIGH);

  // Shooter Mechanism
  private FlyWheel shooter = new FlyWheel(shooterConfig, sparkSmartMotorController);

<strong>  /**
</strong><strong>   * Gets the current velocity of the shooter.
</strong><strong>   *
</strong><strong>   * @return Shooter velocity.
</strong><strong>   */
</strong><strong>  public AngularVelocity getVelocity() {return shooter.getSpeed();}
</strong>
<strong>  /**
</strong><strong>   * Set the shooter velocity.
</strong><strong>   *
</strong><strong>   * @param speed Speed to set.
</strong><strong>   * @return {@link org.wpilib.command2.RunCommand}
</strong><strong>   */
</strong><strong>  public Command run(AngularVelocity speed) {return shooter.run(speed);}
</strong>
<strong>  /**
</strong><strong>   * Set the dutycycle of the shooter.
</strong><strong>   *
</strong><strong>   * @param dutyCycle DutyCycle to set.
</strong><strong>   * @return {@link org.wpilib.command2.RunCommand}
</strong><strong>   */
</strong><strong>  public Command set(double dutyCycle) {return shooter.set(dutyCycle);}
</strong>
<strong>  /**
</strong><strong>   * Trigger that is true while the shooter is within a tolerance of a velocity.
</strong><strong>   *
</strong><strong>   * @param speed Velocity to check.
</strong><strong>   * @param tolerance Allowed velocity error.
</strong><strong>   * @return {@link Trigger}
</strong><strong>   */
</strong><strong>  public Trigger atSpeed(AngularVelocity speed, AngularVelocity tolerance) {return shooter.near(speed, tolerance);}
</strong>
  /** Creates a new ExampleSubsystem. */
  public ExampleSubsystem() {}

  @Override
  public void periodic() {
    // This method will be called once per scheduler run
    shooter.updateTelemetry();
  }

  @Override
  public void simulationPeriodic() {
    // This method will be called once per scheduler run during simulation
    shooter.simIterate();
  }
}

</code></pre>

## Bind buttons to our `Shooter`

We bind buttons to use the `Commands` from our `Shooter`

<pre class="language-java" data-title="RobotContainer.java" data-line-numbers><code class="lang-java">// Copyright (c) FIRST and other WPILib contributors.
// Open Source Software; you can modify and/or share it under the terms of
// the WPILib BSD license file in the root directory of this project.

package first.robot;

import first.robot.subsystems.ExampleSubsystem;

<strong>import static org.wpilib.units.Units.RPM;
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

<strong>    // Set the default command to force the shooter rest.
</strong><strong>    m_exampleSubsystem.setDefaultCommand(m_exampleSubsystem.set(0));
</strong>  }

  /** Use this method to define your trigger->command mappings. */
  private void configureBindings() {
    
<strong>    // Schedule `run` when the Xbox controller's A or B button is pressed,
</strong><strong>    // cancelling on release.
</strong><strong>    m_driverController.a().whileTrue(m_exampleSubsystem.run(RPM.of(60)));
</strong><strong>    m_driverController.b().whileTrue(m_exampleSubsystem.run(RPM.of(300)));
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

To feed a game piece only once the wheel is up to speed, bind your feeder to the `atSpeed` trigger, for example `m_exampleSubsystem.atSpeed(RPM.of(300), RPM.of(10)).and(m_driverController.rightTrigger()).whileTrue(feeder.set(0.5));`. For a plain boolean check outside of a `Trigger`, `FlyWheel` also has `isNear(speed, tolerance)`.

{% hint style="info" %}
**Using Commands v3?** Import `yams.commands3.config.SmartMotorControllerConfig` and `yams.commands3.mechanisms.FlyWheel` instead, have your class implement `org.wpilib.command3.Mechanism` instead of extending `SubsystemBase`, and use `org.wpilib.command3.Command` and `org.wpilib.command3.Trigger`. The `FlyWheel` command and trigger factories (`run()`, `runTo()`, `set()`, `near()`) have the same names. Call `updateTelemetry()` and `simIterate()` from your robot's periodic method yourself, since a v3 `Mechanism` has no `periodic()` hook. See `examples/commands3/simple_robot-2027` in the YAMS repo for a complete robot.
{% endhint %}
