---
icon: sliders
---

# Setpoint Methods vs Command Helpers

YAMS Mechanism classes provide two ways to control mechanisms: **direct setpoint methods** inherited from `SmartMechanism`, and **command helper methods** that return `Command` objects. Understanding when to use each is key to writing clean, effective robot code.

The command helpers come from the mechanism classes in `yams.commands2.mechanisms` (Commands v2) or `yams.commands3.mechanisms` (Commands v3). Both have the same method names and arguments; only the returned `Command` and `Trigger` types differ (`org.wpilib.command2.Command` and `org.wpilib.command2.button.Trigger`, or `org.wpilib.command3.Command` and `org.wpilib.command3.Trigger`). The direct setpoint methods come from `yams.core.mechanisms.SmartMechanism` and are the same in both.

## Overview

| Approach | Example | Returns | Use Case |
|----------|---------|---------|----------|
| **Command Helpers** | `arm.run(Degrees.of(45))` | `Command` | Button bindings, autonomous sequences |
| **Direct Setpoint Methods** | `arm.setMechanismPositionSetpoint(Degrees.of(45))` | `void` | Custom commands, AdvantageKit IO, low-level control |

<figure><img src="../.gitbook/assets/setpoint-two-ways.png" alt="Two paths to the same SmartMotorController. A command helper such as arm.run returns a Command that requires the subsystem; a button binding or auto sequence schedules it, and the scheduler stops it when another command needs the mechanism. A direct setpoint method such as arm.setMechanismPositionSetpoint returns nothing and sets the target at once; it is called from your own code, such as a custom command, an IO class or a state machine, and nothing is scheduled."><figcaption><p>Both paths end at the same motor controller.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/setpoint-helper-vs-direct.gif" alt="Animation of the same arm moving to 45 degrees two ways. With the command helper arm.run, a command stays scheduled and owns the arm for as long as it runs. With the direct setpoint method there is one call that returns at once and nothing is scheduled. The arm moves and holds the same way in both."><figcaption><p>A helper gives you a <code>Command</code> that stays scheduled. A setpoint method is one call.</p></figcaption></figure>

## Command Helper Methods

Command helpers are methods on Mechanism classes that return `Command` objects. These are the **recommended approach** for most use cases.

### Why Use Command Helpers?

1. **Subsystem Requirements** - Commands automatically require their subsystem (or Commands v3 mechanism), preventing conflicts
2. **Lifecycle Management** - Commands handle starting/stopping the closed loop controller
3. **Composability** - Easy to chain with `.andThen()`, `.alongWith()`, etc. in Commands v2, or to `await` inside a coroutine in Commands v3
4. **Trigger Integration** - Work directly with button triggers like `controller.a().onTrue()`
5. **Naming** - Commands are automatically named for debugging

<figure><img src="../.gitbook/assets/mech-commands.png" alt="Three timelines starting when the command is scheduled. run(setpoint), also called setAngle or setHeight, runs until something else takes the mechanism and holds the setpoint the whole time. runTo(setpoint, tolerance) ends when the mechanism is within the tolerance, and then the default command takes over. set(dutycycle) and setVoltage(volts) are open loop with no PID, run until interrupted, and the closed loop controller comes back when they end."><figcaption><p>When each kind of command ends.</p></figcaption></figure>

### Available Command Helpers by Mechanism

#### Arm / Pivot

```java
// Run continuously to an angle (use for teleop)
arm.run(Degrees.of(45))
arm.run(() -> Degrees.of(joystick.getY() * 90))  // Supplier version
arm.setAngle(Degrees.of(45))         // Alias for run()

// Run to angle and end when reached (use in autonomous sequences)
arm.runTo(Degrees.of(45), Degrees.of(1))  // with tolerance

// Open loop control
arm.set(0.5)                         // Duty cycle [-1, 1]
arm.set(() -> joystick.getY())       // Duty cycle supplier
arm.setVoltage(Volts.of(6))          // Voltage
arm.setVoltage(() -> Volts.of(6))    // Voltage supplier
```

#### Elevator

```java
// Run continuously to a height (use for teleop)
elevator.run(Meters.of(1.0))
elevator.run(() -> Meters.of(joystick.getY()))
elevator.setHeight(Meters.of(1.0))   // Alias for run()

// Run to height and end when reached (use in autonomous sequences)
elevator.runTo(Meters.of(1.0), Centimeters.of(2))  // with tolerance

// Open loop control
elevator.set(0.5)
elevator.setVoltage(Volts.of(6))
```

#### FlyWheel

```java
// Run continuously at a velocity (use for teleop)
flywheel.run(RPM.of(5000))
flywheel.run(() -> RPM.of(targetRPM))

// Run to velocity and end when reached (use for spin-up sequences)
flywheel.runTo(RPM.of(5000), RPM.of(100))  // with tolerance

// Linear velocity (requires circumference configured)
flywheel.run(MetersPerSecond.of(20))
flywheel.runTo(MetersPerSecond.of(20), MetersPerSecond.of(0.5))

// Open loop control
flywheel.set(0.8)
flywheel.setVoltage(Volts.of(10))
```

### Example: Button Bindings

{% tabs %}
{% tab title="Commands v2" %}
```java
// Teleop controls - use run() for continuous control
controller.a().whileTrue(arm.run(Degrees.of(45)));
controller.b().whileTrue(arm.run(Degrees.of(-30)));
controller.x().whileTrue(elevator.run(Meters.of(1.0)));
controller.y().whileTrue(flywheel.run(RPM.of(5000)));

// Autonomous-style - use runTo() when you need to wait for completion
controller.leftBumper().onTrue(
    arm.runTo(Degrees.of(90), Degrees.of(2))
        .andThen(shooter.runTo(RPM.of(5000), RPM.of(50)))
        .andThen(intake.feedNote())
);
```
{% endtab %}

{% tab title="Commands v3" %}
```java
// Teleop controls - use run() for continuous control
controller.a().whileTrue(arm.run(Degrees.of(45)));
controller.b().whileTrue(arm.run(Degrees.of(-30)));
controller.x().whileTrue(elevator.run(Meters.of(1.0)));
controller.y().whileTrue(flywheel.run(RPM.of(5000)));

// Autonomous-style - await each runTo() inside a coroutine instead of chaining
controller.leftBumper().onTrue(Command.noRequirements(coroutine -> {
  coroutine.await(arm.runTo(Degrees.of(90), Degrees.of(2)));
  coroutine.await(shooter.runTo(RPM.of(5000), RPM.of(50)));
  coroutine.await(intake.feedNote());
}).named("Aim And Shoot"));
```
{% endtab %}
{% endtabs %}

## Direct Setpoint Methods

Direct setpoint methods are inherited from `SmartMechanism` and set the motor controller's target immediately without returning a Command.

### SmartMechanism Setpoint Methods

All Mechanism classes inherit these methods from `SmartMechanism`:

```java
// Position setpoints
void setMechanismPositionSetpoint(Angle angle)      // Rotational position (Arm, Pivot)
void setMeasurementPositionSetpoint(Distance dist)  // Linear position (Elevator)

// Velocity setpoints
void setMechanismVelocitySetpoint(AngularVelocity vel)  // Rotational velocity
void setMeasurementVelocitySetpoint(LinearVelocity vel) // Linear velocity (Elevator, FlyWheel)

// Open loop setpoints
void setVoltageSetpoint(Voltage volts)              // Voltage control
void setDutyCycleSetpoint(double dutyCycle)         // Duty cycle [-1, 1]
```

### When to Use Direct Setpoint Methods

1. **AdvantageKit IO Classes** - When implementing IO interfaces that don't use Commands
2. **Custom Command Logic** - When building complex commands with custom execute() methods
3. **State Machine Control** - When a state machine directly controls mechanism state
4. **Performance Critical** - Avoiding Command overhead in tight loops (rare)

### Example: AdvantageKit IO

```java
public class ArmIOYAMS implements ArmIO {
  private final Arm arm;

  @Override
  public void setTargetAngle(double rotations) {
    // Use direct setpoint method - no Command needed
    arm.setMechanismPositionSetpoint(Rotations.of(rotations));
  }
}
```

### Example: Custom Command

{% tabs %}
{% tab title="Commands v2" %}
```java
public class ComplexArmCommand extends Command {
  private final Arm arm;
  private final Supplier<Angle> targetSupplier;

  @Override
  public void execute() {
    Angle target = targetSupplier.get();
    // Apply custom logic before setting
    if (arm.getAngle().lt(Degrees.of(0))) {
      target = target.plus(Degrees.of(5)); // Offset when below horizontal
    }
    arm.setMechanismPositionSetpoint(target);
  }
}
```
{% endtab %}

{% tab title="Commands v3" %}
Commands v3 has no command classes with `execute()`. Write the same logic as a `while (true)` coroutine on your mechanism:

```java
public Command complexArmCommand(Supplier<Angle> targetSupplier) {
  return run(coroutine -> {
    while (true) {
      Angle target = targetSupplier.get();
      // Apply custom logic before setting
      if (arm.getAngle().lt(Degrees.of(0))) {
        target = target.plus(Degrees.of(5)); // Offset when below horizontal
      }
      arm.setMechanismPositionSetpoint(target);
      coroutine.yield();
    }
  }).named("Complex Arm Command");
}
```
{% endtab %}
{% endtabs %}

### Example: State machine (Commands v3)

A state machine keeps one variable for what the mechanism should be doing, and one command that reads it every loop. Buttons and other commands only ask for a state. The direct setpoint methods fit this well, because the state machine is the only thing that ever commands the motor.

```java
public class ArmMechanism implements Mechanism {
  /** Every state the arm can be in, and the angle it holds there. */
  public enum ArmState {
    STOWED(Degrees.of(0)),
    INTAKING(Degrees.of(-20)),
    SCORING(Degrees.of(80));

    public final Angle angle;

    ArmState(Angle angle) {
      this.angle = angle;
    }
  }

  private static final Angle TOLERANCE = Degrees.of(2);

  private final Arm arm;
  private final BooleanSupplier hasGamePiece;
  private ArmState state = ArmState.STOWED;

  public ArmMechanism(BooleanSupplier hasGamePiece) {
    this.hasGamePiece = hasGamePiece;
    // createArm() stands for the usual setup: the SmartMotorControllerConfig, the SmartMotorController and the Arm.
    arm = createArm();
  }

  /** Ask for a state. A plain call: it does not schedule or interrupt anything. */
  public void request(ArmState next) {
    state = next;
  }

  /** The same request as a command, for button bindings. It requires nothing, so it never interrupts the state machine. */
  public Command requestCommand(ArmState next) {
    return Command.noRequirements(coroutine -> request(next)).named("Arm request " + next);
  }

  public ArmState getState() {
    return state;
  }

  /** True once the arm has reached the angle of its current state. */
  public boolean atState() {
    return arm.isNear(state.angle, TOLERANCE);
  }

  /**
   * The one command that owns the arm. Each loop it applies the transitions, then sends the
   * current state's angle with a direct setpoint method. Use it as the default command.
   */
  public Command runStateMachine() {
    return run(coroutine -> {
      while (true) {
        switch (state) {
          // Once a game piece is in, stow by itself.
          case INTAKING -> {
            if (hasGamePiece.getAsBoolean()) {
              state = ArmState.STOWED;
            }
          }
          // Once the game piece has left, stow by itself.
          case SCORING -> {
            if (atState() && !hasGamePiece.getAsBoolean()) {
              state = ArmState.STOWED;
            }
          }
          case STOWED -> {
            // Nothing to do: wait for a request.
          }
        }
        arm.setMechanismPositionSetpoint(state.angle);
        coroutine.yield();
      }
    }).named("Arm State Machine");
  }

  public void periodic() {
    arm.updateTelemetry();
  }

  public void simulationPeriodic() {
    arm.simIterate();
  }
}
```

Make the state machine the default command, and have the bindings request states:

```java
// In Robot: the state machine is the arm's default command, so it always runs.
arm.setDefaultCommand(arm.runStateMachine());

// In an opmode: buttons only ask for a state.
controller.a().onTrue(arm.requestCommand(ArmState.INTAKING));
controller.b().onTrue(arm.requestCommand(ArmState.SCORING));
controller.x().onTrue(arm.requestCommand(ArmState.STOWED));
```

How it fits together:

* **One owner.** `runStateMachine()` is the only command that requires the arm, so nothing can interrupt it by accident.
* **Requests are plain calls.** `request(...)` just changes a variable. `requestCommand(...)` wraps it in a command that requires nothing, so pressing a button never interrupts the state machine.
* **Transitions live in one place.** The `switch` decides when the arm leaves a state by itself, such as stowing once a game piece is in.
* **`setMechanismPositionSetpoint(...)` runs every loop** with the current state's angle, inside a `while (true)` loop that ends each pass with `coroutine.yield()`.

### Starting and stopping the SystemCore closed loop

Some closed loops run on the motor controller and some run on the SystemCore. [Where the loop runs](editor/config/closed-loop-control.md#where-the-loop-runs) lists which is which. A loop on the SystemCore is a piece of YAMS code that runs every control period and writes an output to the motor, and it keeps doing that until it is stopped. So with the direct setpoint methods, **you** have to start and stop it to match the kind of control you want:

| You want | The SystemCore closed loop must be | Why |
| --- | --- | --- |
| Position or velocity control (`setMechanismPositionSetpoint`, `setMechanismVelocitySetpoint` and the measurement versions) | Running | A stopped loop does nothing with the setpoint, so the mechanism does not move to it |
| Voltage or duty cycle control (`setVoltageSetpoint`, `setDutyCycleSetpoint`) | Stopped | A running loop writes its own output on its next run and overrides yours |

```java
arm.getMotorController().startClosedLoopController(); // before position or velocity control
arm.getMotorController().stopClosedLoopController();  // before voltage or duty cycle control
```

{% hint style="info" %}
The [command helpers](#command-helper-methods) already do this. `set(...)` and `setVoltage(...)` stop the closed loop while they run and start it again when they end. You only need to do it yourself when you use the direct setpoint methods.
{% endhint %}

The pattern is always the same: put the loop in the state your command needs when the command starts, and put it back when the command ends, however it ends.

{% tabs %}
{% tab title="Commands v2" %}
Use `startRun` to set the loop up once and then run every loop, and `finallyDo` to put it back whether the command finishes or is interrupted.

```java
/** Drive the arm by voltage, for example from a joystick. */
public Command manual(Supplier<Voltage> volts) {
  return startRun(
          () -> arm.getMotorController().stopClosedLoopController(),   // once, at the start
          () -> arm.setVoltageSetpoint(volts.get()))                    // every loop
      .finallyDo(() -> arm.getMotorController().startClosedLoopController()) // at the end, always
      .withName("Arm Manual");
}

/** Hold an angle that can change while the command runs. */
public Command track(Supplier<Angle> angle) {
  return startRun(
          () -> arm.getMotorController().startClosedLoopController(),  // once, at the start
          () -> arm.setMechanismPositionSetpoint(angle.get()))          // every loop
      .withName("Arm Track");
}
```
{% endtab %}

{% tab title="Commands v3" %}
Set the loop up before the `while (true)` loop, and use `whenCanceled` to put it back.

```java
/** Drive the arm by voltage, for example from a joystick. */
public Command manual(Supplier<Voltage> volts) {
  return run(coroutine -> {
    arm.getMotorController().stopClosedLoopController();                // once, at the start
    while (true) {
      arm.setVoltageSetpoint(volts.get());                              // every loop
      coroutine.yield();
    }
  }).whenCanceled(() -> arm.getMotorController().startClosedLoopController()) // at the end
    .named("Arm Manual");
}

/** Hold an angle that can change while the command runs. */
public Command track(Supplier<Angle> angle) {
  return run(coroutine -> {
    arm.getMotorController().startClosedLoopController();               // once, at the start
    while (true) {
      arm.setMechanismPositionSetpoint(angle.get());                    // every loop
      coroutine.yield();
    }
  }).named("Arm Track");
}
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
**Starting the loop resets it.** `startClosedLoopController()` resets the PID controller and starts the motion profile again from where the mechanism is now. Call it once when a command starts, as above, and not every loop, or the profile never gets going.
{% endhint %}

{% hint style="success" %}
**The same code is safe on every motor controller.** When the closed loop runs on the motor controller itself, there is no SystemCore loop, and both calls do nothing. So you can write commands this way without knowing where the loop runs, and they keep working if you change motor controller or profile later.
{% endhint %}

## Setpoint Method Support Matrix

Every Mechanism class inherits all of these methods, but not all of them make sense for every mechanism. "No" below means the method does not apply to that mechanism.

### Position Setpoint Methods

| Method | Arm | Pivot | Elevator | FlyWheel |
|--------|-----|-------|----------|----------|
| `setMechanismPositionSetpoint(Angle)` | Yes | Yes | Yes | No |
| `setMeasurementPositionSetpoint(Distance)` | No | No | Yes | No |

### Velocity Setpoint Methods

| Method | Arm | Pivot | Elevator | FlyWheel |
|--------|-----|-------|----------|----------|
| `setMechanismVelocitySetpoint(AngularVelocity)` | Yes | Yes | Yes | Yes |
| `setMeasurementVelocitySetpoint(LinearVelocity)` | No | No | Yes | Yes* |

*FlyWheel requires circumference to be configured for linear velocity.

### Open Loop Setpoint Methods

| Method | Arm | Pivot | Elevator | FlyWheel |
|--------|-----|-------|----------|----------|
| `setVoltageSetpoint(Voltage)` | Yes | Yes | Yes | Yes |
| `setDutyCycleSetpoint(double)` | Yes | Yes | Yes | Yes |

{% hint style="warning" %}
**Linear Methods**: The `Distance` and `LinearVelocity` methods convert through the mechanism circumference in the `SmartMotorControllerConfig`. Calling one without a circumference configured (like `setMeasurementPositionSetpoint(Distance)` on an `Arm`) throws a `SmartMotorControllerConfigurationException`, and `FlyWheel.setMeasurementVelocitySetpoint(LinearVelocity)` throws a `FlyWheelConfigurationException` if the FlyWheel diameter is not set. Check the matrix above before using direct setpoint methods.
{% endhint %}

## Checking Position with near() and isNear()

To check if a mechanism is near a target, use `near()`, which returns a `Trigger`:

```java
// Arm / Pivot - uses Angle.isNear() internally
Trigger nearTarget = arm.near(Degrees.of(45), Degrees.of(2));

// Elevator - uses Distance.isNear() internally  
Trigger nearHeight = elevator.near(Meters.of(1.0), Centimeters.of(2));

// FlyWheel - uses AngularVelocity.isNear() internally
Trigger atSpeed = flywheel.near(RPM.of(5000), RPM.of(100));
```

These methods use WPILib's built-in `isNear()` function on the `Angle`, `Distance`, and `AngularVelocity` measure classes for accurate tolerance checking.

{% hint style="info" %}
The `Trigger` factory used to be called `isNear()` and was renamed to `near()` to match the other Trigger factories (`gte()`, `lte()`, `between()`, `max()`, `min()`). `isNear()` with the same arguments still exists and returns a plain `boolean`, for use outside a `Command` or `Trigger`.
{% endhint %}

<figure><img src="../.gitbook/assets/mech-triggers.png" alt="A number line from minus 30 to 120 degrees with a mechanism at 72 degrees. near 80 within 10 is true. between 0 and 60 is false. gte 45 is true. lte 45 is false. max and min, at the ends of the travel, are false. Each is a Trigger, and the is functions such as isNear give the same answers as booleans."><figcaption><p>The same questions as Triggers or as booleans.</p></figcaption></figure>

### Using near() in Commands

{% tabs %}
{% tab title="Commands v2" %}
```java
// Wait until arm is near target
Commands.waitUntil(arm.near(Degrees.of(45), Degrees.of(1)))

// Conditional logic based on position
arm.near(Degrees.of(90), Degrees.of(5)).onTrue(ledSubsystem.setGreen());

// In a command's isFinished(), use the boolean isNear()
@Override
public boolean isFinished() {
    return arm.isNear(targetAngle, tolerance);
}
```
{% endtab %}

{% tab title="Commands v3" %}
```java
// Wait until arm is near target, inside a coroutine
coroutine.waitUntil(arm.near(Degrees.of(45), Degrees.of(1)));

// Conditional logic based on position
arm.near(Degrees.of(90), Degrees.of(5)).onTrue(leds.setGreen());

// Plain boolean check, e.g. in an if statement inside a coroutine
if (arm.isNear(targetAngle, tolerance)) {
  // ...
}
```
{% endtab %}
{% endtabs %}

## Decision Guide

<figure><img src="../.gitbook/assets/setpoint-decision.png" alt="A decision diagram. If the mechanism is controlled by a button binding or an autonomous sequence, use a command helper such as arm.run, arm.runTo or arm.setAngle. If it is controlled by a custom command, an IO class or a state machine, use a direct setpoint method such as arm.setMechanismPositionSetpoint, elevator.setMeasurementPositionSetpoint or arm.setVoltageSetpoint."><figcaption><p>Start from what is making the call.</p></figcaption></figure>

## Best Practices

1. **Prefer Command Helpers** - They handle subsystem requirements and closed loop lifecycle
2. **Use run() for Teleop** - Continuous control that responds to interruption
3. **Use runTo() for Sequences** - When you need to wait for the mechanism to reach a target
4. **Use Direct Methods for IO** - When implementing AdvantageKit IO or custom commands
5. **Always Check Support** - Verify the method is supported before using direct setpoint methods
6. **Use Triggers for State** - Use `near()`, `max()`, `min()` triggers instead of polling

## See Also

- [run() vs runTo() Commands](run-vs-runto.md) - Important differences between continuous and goal-based commands
- [AdvantageKit Integration](advantagekit-integration.md) - Using YAMS with AdvantageKit's IO pattern
- [Live Tuning](live-tuning/README.md) - Units for all tunable parameters
