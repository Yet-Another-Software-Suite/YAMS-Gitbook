# Steps 3 and 4: The subsystem and RobotContainer

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
See [run() vs runTo() Commands](../run-vs-runto.md) for important information about when commands end and how that interacts with default commands.
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

Next: [Log replay](log-replay.md)
