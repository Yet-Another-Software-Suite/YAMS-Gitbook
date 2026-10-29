---
icon: cubes
---

# Steps 3 and 4: The subsystem and RobotContainer

## Step 3: Create the subsystem

The subsystem uses the IO interface and doesn't know whether it's running on real hardware or in simulation. It builds its own IO class: the IO class needs the subsystem for its `SmartMotorControllerConfig`, and the subsystem does not exist yet while `RobotContainer` is still creating it. So the constructor takes a factory, a function that builds the IO class from the subsystem, such as `ArmIOTalonFX::new`.

```java
package first.robot.subsystems.akit;

import static org.wpilib.units.Units.*;

import java.util.function.Function;
import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import org.wpilib.units.measure.Angle;
import org.littletonrobotics.junction.Logger;

public class ArmSubsystem extends SubsystemBase {

  private final ArmIO io;
  private final ArmIOInputsAutoLogged inputs = new ArmIOInputsAutoLogged();

  /**
   * @param ioFactory Builds the IO class for this subsystem, for example ArmIOTalonFX::new.
   *                  The subsystem passes itself in, because the IO class's
   *                  SmartMotorControllerConfig needs the subsystem.
   */
  public ArmSubsystem(Function<SubsystemBase, ArmIO> ioFactory) {
    this.io = ioFactory.apply(this);
  }

  @Override
  public void periodic() {
    // Update the arm's telemetry
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
   * Command to move the arm to a target angle and hold it.
   * This command does not end on its own.
   */
  public Command setAngle(Angle angle) {
    return run(() -> io.setTargetAngle(angle))
        .withName("Arm.setAngle(" + angle + ")");
  }

  /**
   * Command to move the arm to a target angle and finish when reached.
   * Be careful with default commands, see the hint below.
   */
  public Command goToAngle(Angle angle, Angle tolerance) {
    return run(() -> io.setTargetAngle(angle))
        .until(() -> isNear(angle, tolerance))
        .withName("Arm.goToAngle(" + angle + ")");
  }

  /** Returns true if the logged arm position is within tolerance of a target, using WPILib's isNear(). */
  public boolean isNear(Angle target, Angle tolerance) {
    return getAngle().isNear(target, tolerance);
  }

  /** The logged arm position, so it replays correctly. */
  public Angle getAngle() {
    return Rotations.of(inputs.positionRotations);
  }

  /** Command to stop the arm. */
  public Command stop() {
    return runOnce(() -> io.stop()).withName("Arm.stop");
  }

}
```

{% hint style="warning" %}
See [When mechanism commands end](../run-vs-runto.md) for important information about when commands end and how that interacts with default commands.
{% endhint %}

## Step 4: Wire it up in RobotContainer

With YAMS's passive simulation, you don't need conditional logic for real vs sim. The same IO class works for both.

```java
package first.robot;

import static org.wpilib.units.Units.Degrees;

import org.wpilib.command2.button.CommandNiDsXboxController;
import first.robot.subsystems.akit.ArmIOTalonFX;
import first.robot.subsystems.akit.ArmSubsystem;

public class RobotContainer {

  // Same IO class for the real robot and simulation. YAMS handles the difference.
  private final ArmSubsystem arm = new ArmSubsystem(ArmIOTalonFX::new);

  private final CommandNiDsXboxController controller = new CommandNiDsXboxController(0);

  public RobotContainer() {
    configureBindings();
  }

  private void configureBindings() {
    // Commands work identically on the real robot and in simulation
    controller.a().whileTrue(arm.setAngle(Degrees.of(45)));
    controller.b().whileTrue(arm.setAngle(Degrees.of(-45)));
  }
}
```

The elevator and shooter follow the same pattern: an `ElevatorSubsystem` and a `ShooterSubsystem` that take `Function<SubsystemBase, ElevatorIO>` and `Function<SubsystemBase, ShooterIO>`, log their inputs in `periodic()`, and have command methods that take a `Distance` and an `AngularVelocity`. Create them with `new ElevatorSubsystem(ElevatorIOTalonFX::new)` and `new ShooterSubsystem(ShooterIOTalonFX::new)`.

**Check:** run the code in simulation and hold A. The arm swings to 45 degrees on the simulation's `Mechanism2d`, and in AdvantageScope `AdvantageKit/Arm/positionRotations` rises to about `0.125`.

**If it did not work:**

* **`ArmIOInputsAutoLogged` cannot be found:** the AutoLog code generator is not set up. See [Step 0](setup.md).
* **Nothing is logged under `Arm`:** `Robot` does not call `Logger.start()`, or it is not a `LoggedRobot`. See [Step 0](setup.md).
* **The arm does not move:** check the hard limits in `ArmIOTalonFX` allow 45 degrees (the example's are ±0.3 rotations, about ±108 degrees).

{% hint style="info" %}
**Compare to Traditional AdvantageKit**: Without YAMS, you'd need separate `ArmIOReal` and `ArmIOSim` classes with duplicate configuration. YAMS eliminates this duplication entirely.
{% endhint %}

Next: [Log replay](log-replay.md)
