---
description: >-
  A game piece makes a mechanism heavier, and can move its center of mass, which
  changes the feedforward and feedback gains it needs.
---

# How do I control a mechanism that carries a game piece?

Gains tuned on an empty mechanism are tuned for that mechanism. Pick up a game piece and the elevator is heavier, and the arm is heavier and has its center of mass further out. An empty arm that holds its angle perfectly can sag below it with a game piece, or overshoot and oscillate when it moves.

How much changes depends on the mechanism: how heavy the game piece is next to the mechanism, and where it is held.

## What changes, and what doesn't

| Gain                  | Depends on                                                           | With a game piece                                                      |
| --------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `kS`                  | Friction in the gearbox and bearings                                 | Barely changes                                                         |
| `kV`                  | The motor and the gearing                                            | **Does not change.** It is the voltage the motor needs to spin at a speed |
| `kG`                  | The weight, and for an arm how far out the center of mass is         | **Goes up**, often a lot                                               |
| `kA`                  | The mass, or for an arm the moment of inertia                        | Goes up                                                                |
| `kP`, `kD`            | The mass or inertia they have to move                                | The same gains move a heavier mechanism more slowly, with more overshoot |
| Profile limits        | How fast the motors can accelerate the mechanism                     | A heavier mechanism may not keep up with the profile                   |

`kG` is usually the one that matters. It holds the mechanism up, so a `kG` that is too small makes the mechanism sit below its setpoint, with `kP` pushing against the difference.

## How much it changes, by mechanism

### Elevator

`kG` is in proportion to the weight the elevator lifts:

```
kG with the game piece = kG empty × (carriage weight + game piece weight) / carriage weight
```

A 1 kg game piece on a 4 kg carriage needs 25% more `kG`. A spring or counterweight that holds up part of the carriage takes a fixed amount off `kG`, so the change is the same in volts but a larger share of what is left.

A **continuous** or **cascade** elevator's stages are lifted in turn as it rises, so `kG` steps up as each stage is picked up, with or without a game piece. If the elevator sags at some heights and not others, this is a likely reason.

### Arm

`kG` is in proportion to the arm's weight times how far its center of mass is from the pivot, and the game piece adds its own:

```
kG with the game piece = kG empty × (arm weight × arm's center of mass distance
                                     + game piece weight × game piece distance)
                                   / (arm weight × arm's center of mass distance)
```

A game piece at the end of an arm adds far more than its weight suggests. A 1 kg game piece 0.6 m out, on a 3 kg arm whose center of mass is 0.3 m out, needs 67% more `kG`.

A game piece held off to one side of the arm, rather than along it, also moves the angle at which gravity pulls hardest. `ArmFeedforward` assumes the center of mass is on the line from the pivot, at the angle the arm reads. If the game piece is far off that line, the arm holds well at some angles and sags at others. A larger `kG` helps most where the arm is close to level.

### Arm on an elevator, or a telescoping arm

The elevator lifts the arm and the game piece, so the elevator's `kG` goes up with the game piece, but does not depend on the arm's angle.

A **telescoping arm**, or an arm with a wrist, moves its center of mass as it extends or bends. Its `kG` changes all the time, not only with the game piece.

### Shooter flywheel

A game piece passes through the flywheel rather than staying on it. Its `kV` does not change. The flywheel slows as it shoots, and `kP` and the flywheel's own inertia bring it back up. See [How do I shoot with a shooter?](how-to-shoot-with-a-shooter.md).

## Measure the gains with the game piece

Measure the loaded gains on the robot, as you measured the empty ones, with [live tuning](../details/live-tuning/):

1. Load a game piece, and move the mechanism to where gravity pulls hardest: an arm level, or an elevator at any height. Then let go, so that only the motor holds it from here on.
2. Raise the voltage until the mechanism just starts to move up, and note it. Lower it until the mechanism just starts to move down, and note that.
3. `kG` is halfway between the two voltages. `kS` is half the difference between them.
4. With that `kG`, check the mechanism reaches its setpoints with the game piece, and raise `kP` (and `kD`) if it is slow.

The formulas above give a starting point, and tell you whether a game piece changes the gains enough to matter. A change of a few percent may not be worth a second set of gains.

## Switch gains when the mechanism picks up a game piece

A motor controller holds up to four [slots](../details/editor/config/closed-loop-control.md#slots) of gains, each with its own [feedforward](../details/editor/config/feedforward.md#slots). Put the empty gains in one slot and the loaded gains in another, and switch slots when the mechanism picks up or lets go of a game piece. Switching is quick: it only changes which slot the motor controller uses.

{% tabs %}
{% tab title="Commands v2" %}
{% code title="ArmSubsystem.java (fragment)" %}
```java
import org.wpilib.command2.Commands;
import org.wpilib.command2.button.Trigger;
import org.wpilib.math.controller.ArmFeedforward;
import yams.core.motorcontrollers.SmartMotorController.ClosedLoopControllerSlot;

private final SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
    // Empty.
    .withClosedLoopController(40, 0, 0, ClosedLoopControllerSlot.SLOT_0)
    .withFeedforward(new ArmFeedforward(0.1, 0.35, 1.2), ClosedLoopControllerSlot.SLOT_0)
    // Holding a game piece: more kG to hold it up, and more kP to move the extra weight.
    .withClosedLoopController(55, 0, 0, ClosedLoopControllerSlot.SLOT_1)
    .withFeedforward(new ArmFeedforward(0.1, 0.60, 1.2), ClosedLoopControllerSlot.SLOT_1)
    // ... the rest of the config
    ;

public ArmSubsystem() {
  // Switch gains as the intake picks up and lets go of a game piece. The commands require
  // nothing, so they do not interrupt the command moving the arm.
  new Trigger(this::hasGamePiece)
      .onTrue(Commands.runOnce(() -> motor.setClosedLoopSlot(ClosedLoopControllerSlot.SLOT_1)))
      .onFalse(Commands.runOnce(() -> motor.setClosedLoopSlot(ClosedLoopControllerSlot.SLOT_0)));
}
```
{% endcode %}
{% endtab %}

{% tab title="Commands v3" %}
{% code title="ArmMechanism.java (fragment)" %}
```java
import org.wpilib.command3.Command;
import org.wpilib.command3.Trigger;
import org.wpilib.math.controller.ArmFeedforward;
import yams.core.motorcontrollers.SmartMotorController.ClosedLoopControllerSlot;

private final SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
    // Empty.
    .withClosedLoopController(40, 0, 0, ClosedLoopControllerSlot.SLOT_0)
    .withFeedforward(new ArmFeedforward(0.1, 0.35, 1.2), ClosedLoopControllerSlot.SLOT_0)
    // Holding a game piece: more kG to hold it up, and more kP to move the extra weight.
    .withClosedLoopController(55, 0, 0, ClosedLoopControllerSlot.SLOT_1)
    .withFeedforward(new ArmFeedforward(0.1, 0.60, 1.2), ClosedLoopControllerSlot.SLOT_1)
    // ... the rest of the config
    ;

public ArmMechanism() {
  // Switch gains as the intake picks up and lets go of a game piece. The commands require
  // nothing, so they do not interrupt the command moving the arm.
  new Trigger(this::hasGamePiece)
      .onTrue(Command.noRequirements(coroutine -> motor.setClosedLoopSlot(ClosedLoopControllerSlot.SLOT_1))
          .named("Arm Loaded Gains"))
      .onFalse(Command.noRequirements(coroutine -> motor.setClosedLoopSlot(ClosedLoopControllerSlot.SLOT_0))
          .named("Arm Empty Gains"));
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

The gains in this code are **placeholders** to show the shape of the change; use the ones you measured above.

`hasGamePiece()` is whatever tells you the mechanism has a game piece: a beam break, a distance sensor, or the intake's current, as in [How do I detect a game piece from motor current?](how-do-i-sense-activity-based-off-of-the-current.md). The slot takes effect with the next setpoint, and a mechanism's commands send a setpoint every loop.

[Loosely coupled followers](../details/editor/config/followers.md#withlooselycoupledfollowers) are switched to the same slot. Give each follower the same slots in its own config.

## A mechanism whose gains change as it moves

A telescoping arm, an arm with a wrist, or an elevator picking up its stages needs a different `kG` as it moves, not only with a game piece. Two ways to handle it:

* **Slots for ranges.** Up to four sets of gains, such as retracted, extended, and each with a game piece. Switch with triggers on the position, as above. This is the simplest, and the gains in each slot can be tuned on their own.
* **Change `kG` as it moves**, with `motor.setKg(kG)`, worked out from the extension or angle. Only do this when `kG` has changed enough to matter, not every loop: on a Talon or a SPARK, `setKg` writes the motor controller's configuration, which takes time. On a Nova, or any mechanism whose [closed loop runs on the SystemCore](../details/editor/config/closed-loop-control.md#where-the-loop-runs), it only changes a number.

## Common pitfalls

### Only `kP` was raised

A heavier mechanism that sags looks like it needs more `kP`. Raising `kP` pushes it closer to the setpoint, but it still sits below it, and a `kP` large enough to hide the sag makes the empty mechanism oscillate. Raise `kG` first, and `kP` only if the mechanism is still slow.

### The slot switches on a flickering sensor

A sensor that flickers as a game piece passes switches the gains back and forth. Debounce the trigger, such as `new Trigger(this::hasGamePiece).debounce(0.1)`, so it only switches once the sensor has settled.

### Slot 3 on a Talon

A TalonFX and a TalonFXS have three slots, `SLOT_0` to `SLOT_2`. Selecting `SLOT_3` throws an exception.

### The profile is too fast for the loaded mechanism

The motion profile's limits are the same in every slot. A profile that the empty mechanism keeps up with can be too fast with a game piece, so the mechanism lags behind it and overshoots at the end. Set the profile's limits for the loaded mechanism.

### The simulation is always empty

The simulation uses the mass and moment of inertia in the config, so it never carries the game piece. Gains for the loaded mechanism can be checked in simulation by setting the config's mass to the loaded mass, but tune them on the robot.

## Check it

Graph the arm's angle and setpoint, pick up a game piece, and send the arm to the same setpoint with and without it. With the right slot selected, the arm should settle on the setpoint both times, rather than sagging below it when loaded. Watch the motor's `closedloop/slot` value on the dashboard change from 0 to 1 as it picks the game piece up.

## Next

Tune the empty mechanism first, if you have not: [Step 5: Tune it](../tutorials/arm/tune.md).
