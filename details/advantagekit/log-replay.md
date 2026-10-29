# Log replay

## Log replay mode

For [log replay](https://docs.advantagekit.org/getting-started/what-is-advantagekit), AdvantageKit provides a "replay" mode where inputs are read from a log file instead of hardware. This is the **one case** where you need a separate IO implementation: an empty one that does nothing.

<figure><img src="../../.gitbook/assets/akit-loop-live-and-replay.gif" alt="Animation of one loop of the robot shown twice. On the real robot and in simulation, the IO class reads the YAMS arm into the inputs, the inputs are written to the log, the subsystem logic runs on them, and its setpoint goes back through the IO class to the YAMS arm. In log replay the inputs come from the log file, the same subsystem logic runs on them, and the outputs go to an empty replay IO class that drops them."><figcaption><p>In replay the inputs come from the log, and the empty IO class drops the outputs.</p></figcaption></figure>

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

In RobotContainer, only check for replay mode. `Robot.currentMode` comes from [Step 0](setup.md):

```java
package first.robot;

import first.robot.subsystems.akit.ArmIOReplay;
import first.robot.subsystems.akit.ArmIOTalonFX;
import first.robot.subsystems.akit.ArmSubsystem;

public class RobotContainer {

  // Replay mode: the empty IO class, AdvantageKit fills the inputs from the log.
  // Real AND sim: the same IO class works for both.
  private final ArmSubsystem arm = new ArmSubsystem(
      Robot.currentMode == Robot.Mode.REPLAY ? subsystem -> new ArmIOReplay() : ArmIOTalonFX::new);

  // The elevator and shooter follow the same pattern with ElevatorIOReplay and ShooterIOReplay.
}
```

To replay a log, set `simMode` in `Robot` to `Mode.REPLAY` and run the simulation. `LogFileUtil.findReplayLog()` uses the log open in AdvantageScope, or asks you for one. See [How To: Traditional Replay](https://docs.advantagekit.org/getting-started/traditional-replay) in the AdvantageKit docs.

**Check:** a new log file ending in `_sim` appears next to the one you replayed. Open both in AdvantageScope: `Arm/positionRotations` is the same in both.

{% hint style="success" %}
**Summary**: With YAMS + AdvantageKit, you only need **2 IO classes** per mechanism:

1. **One real IO class** (e.g., `ArmIOTalonFX`): works for both real robot and simulation
2. **One empty replay IO class** (e.g., `ArmIOReplay`): for log replay only

Compare this to traditional AdvantageKit which requires **3 IO classes** (Real, Sim, Replay).
{% endhint %}

## Pattern summary: mechanism classes with SmartMotorController telemetry

The examples in this tutorial demonstrate the recommended pattern for using YAMS with AdvantageKit:

1. **Create Mechanism classes** (`Arm`, `Elevator`, `FlyWheel`, etc.) for high-level control
2. **Get the underlying `SmartMotorController`** via `mechanism.getMotor()`
3. **Pull telemetry from `SmartMotorController`** for AdvantageKit logging
4. **Send setpoints with the `SmartMotorController`**, such as `motor.setPosition(angle)` or `motor.setVelocity(velocity)`, as the IO classes in [Step 2](io-implementation.md) do
5. **Optionally expose the Mechanism** via a getter for command helpers like `run()` and `runTo()`

This pattern gives you the best of both worlds:

* **Mechanism classes** provide convenient configuration and command helpers
* **SmartMotorController** provides detailed telemetry for AdvantageKit logging
* **Passive simulation** works automatically in both

{% hint style="info" %}
See the YAMS AdvantageKit examples for complete robot projects that log YAMS mechanisms with AdvantageKit: [Commands v2](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands2/advantage_kit) and [Commands v3](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands3/advantage_kit).
{% endhint %}

### Using WPILib's isNear() for tolerance checking

When checking if a mechanism is at its target, use the `isNear()` method from WPILib's unit classes (`Angle`, `Distance`, `AngularVelocity`):

```java
// In your subsystem, check if the logged position is near the target
Angle currentPosition = Rotations.of(inputs.positionRotations);
Angle targetPosition = Degrees.of(45);
Angle tolerance = Degrees.of(2);

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
