# Log replay

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

The examples in this tutorial demonstrate the recommended pattern for using YAMS with AdvantageKit:

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

Next: [Swerve drive with AdvantageKit](swerve-drive.md)
