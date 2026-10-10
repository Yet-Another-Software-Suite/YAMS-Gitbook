---
icon: play
---

# run() vs runTo() Commands

Understanding when to use `run()` versus `runTo()` is critical for building reliable robot control. Choosing the wrong one leads to unexpected behavior, especially when combined with default commands.

The examples on this page use Commands v2 (`yams.commands2.mechanisms.*`). The Commands v3 mechanisms (`yams.commands3.mechanisms.*`) have the same `run()` and `runTo()` methods and they behave the same way; see [Commands v3](#commands-v3) for the differences in how you compose and bind them.

## Quick Reference

| Method | Command Ends? | Motor Stops? | Best For |
|--------|--------------|--------------|----------|
| `run(setpoint)` | Never (runs until interrupted) | No | Continuous control, teleop |
| `runTo(setpoint)` | When goal reached | **No** | Autonomous sequences, one-shot movements |

{% hint style="danger" %}
**Critical**: Neither `run()` nor `runTo()` stops the motor when the command ends. The `SmartMotorController` continues sending the last request. This is by design for smooth control, but can cause unexpected behavior if not understood.
{% endhint %}

## run() - Continuous Control

Use `run()` when you expect the command to run **indefinitely** until interrupted by another command.

```java
/**
 * Set the height of the elevator. Command runs forever until interrupted.
 * @param height Target height
 * @return A Command that never ends on its own
 */
public Command setHeight(Distance height) { 
  return elevator.run(height);
}
```

### When to Use run()

- **Teleop button bindings** with `whileTrue()` - command runs while button is held
- **Default commands** - these should run forever by definition
- **Holding position** - when you want to actively maintain a setpoint
- **Velocity control** - flywheels that spin until told otherwise

### Example: Teleop Elevator Control

```java
// RobotContainer.java
public RobotContainer() {
  // Default command: always try to go to home position
  m_elevator.setDefaultCommand(m_elevator.setHeight(Meters.of(0)));
}

private void configureBindings() {
  // While A is held, go to 0.5m. When released, default command takes over.
  m_driverController.a().whileTrue(m_elevator.setHeight(Meters.of(0.5)));
  
  // While B is held, go to 1.0m. When released, default command takes over.
  m_driverController.b().whileTrue(m_elevator.setHeight(Meters.of(1.0)));
}
```

**What happens:**
1. Robot starts, default command runs, elevator goes to 0m
2. Driver presses A, `setHeight(0.5m)` interrupts default, elevator goes to 0.5m
3. Driver releases A, command is cancelled, default command resumes, elevator returns to 0m

This works correctly because `run()` never ends on its own - it's always interrupted by something else.

## runTo() - Goal-Based Control

Use `runTo()` when you want the command to **end automatically** when the mechanism reaches its goal.

```java
/**
 * Set the height of the elevator. Command ends when goal is reached.
 * @param height Target height
 * @param tolerance Distance tolerance for completion
 * @return A Command that ends when at setpoint
 */
public Command setHeightAndStop(Distance height, Distance tolerance) { 
  return elevator.runTo(height, tolerance);
}
```

### When to Use runTo()

- **Autonomous routines** - move to position, then do the next thing
- **Sequential commands** - `moveArm.andThen(shoot).andThen(moveArmBack)`
- **One-shot movements** - press button once to go to a preset

{% hint style="warning" %}
**Important**: `runTo()` requires a tolerance parameter to determine when the goal is reached. Choose a tolerance appropriate for your mechanism (e.g., `Centimeters.of(2)` for an elevator, `Degrees.of(1)` for an arm). The command ends once the mechanism has stayed within the tolerance for 0.1 seconds (a rising-edge debounce on the `near()` trigger).
{% endhint %}

### Example: Autonomous Sequence

```java
// Autonomous command sequence
public Command autoScoreHigh() {
  return Commands.sequence(
    // Move elevator to scoring position (ends when reached)
    m_elevator.setHeightAndStop(Meters.of(1.2), Centimeters.of(2)),
    // Extend arm (ends when reached)
    m_arm.setAngleAndStop(Degrees.of(45), Degrees.of(1)),
    // Score the game piece
    m_intake.outtake().withTimeout(0.5),
    // Retract arm (ends when reached)
    m_arm.setAngleAndStop(Degrees.of(0), Degrees.of(1)),
    // Lower elevator (ends when reached)
    m_elevator.setHeightAndStop(Meters.of(0), Centimeters.of(2))
  );
}
```

## The Default Command Pitfall

{% hint style="danger" %}
**This is the most common mistake with `runTo()`!**
{% endhint %}

When `runTo()` finishes (because the goal was reached), the command scheduler looks for another command to run. If you have a **default command** set, it will immediately take over.

### The Problem

```java
// RobotContainer.java - PROBLEMATIC CODE
public RobotContainer() {
  // Default command: go to home position
  m_elevator.setDefaultCommand(m_elevator.setHeight(Meters.of(0)));
}

private void configureBindings() {
  // Press A once to go to 1 meter
  m_driverController.a().onTrue(m_elevator.setHeightAndStop(Meters.of(1.0), Centimeters.of(2)));
}
```

**What the programmer expects:**
1. Press A
2. Elevator goes to 1 meter
3. Elevator stays at 1 meter

**What actually happens:**
1. Press A
2. `setHeightAndStop(1.0m)` starts running
3. Elevator moves toward 1 meter
4. Elevator reaches 1 meter, `runTo()` command ends
5. **Default command immediately starts**, commanding 0 meters
6. Elevator goes back down to 0 meters

### Solution 1: Use run() Instead

If you want the mechanism to stay at the commanded position, use `run()`:

```java
// Press A once to go to 1 meter and STAY there
m_driverController.a().onTrue(m_elevator.setHeight(Meters.of(1.0)));
```

Now the `run()` command never ends on its own, so the default command never gets a chance to run. The elevator will stay at 1 meter until another command is scheduled.

### Solution 2: No Default Command

If you don't need a default command, don't set one:

```java
public RobotContainer() {
  // No default command - mechanism holds last setpoint
  // m_elevator.setDefaultCommand(...); // Removed!
}

private void configureBindings() {
  // These work correctly now
  m_driverController.a().onTrue(m_elevator.setHeightAndStop(Meters.of(0.5), Centimeters.of(2)));
  m_driverController.b().onTrue(m_elevator.setHeightAndStop(Meters.of(1.0), Centimeters.of(2)));
}
```

Since there's no default command, when `runTo()` ends, nothing else runs. The `SmartMotorController` keeps sending the last setpoint, holding position.

### Solution 3: Use whileTrue() for Momentary Control

If you want the mechanism to return to a home position when not commanded, use `whileTrue()`:

```java
public RobotContainer() {
  m_elevator.setDefaultCommand(m_elevator.setHeight(Meters.of(0)));
}

private void configureBindings() {
  // HOLD A to go to 1 meter, RELEASE to return home
  m_driverController.a().whileTrue(m_elevator.setHeight(Meters.of(1.0)));
}
```

This makes the intent clear: hold the button to hold the position.

### Solution 4: Use Triggers for State-Based Control

Following [command-based best practices](https://bovlb.github.io/frc-tips/commands/best-practices.html), use triggers to control behavior:

```java
public class ElevatorSubsystem extends SubsystemBase {
  // Expose state as triggers
  public final Trigger isHigh = new Trigger(() -> elevator.getPosition().gt(Meters.of(0.8)));
  public final Trigger isLow = new Trigger(() -> elevator.getPosition().lt(Meters.of(0.2)));
  
  // Command factories
  public Command goHigh() { return elevator.run(Meters.of(1.0)); }
  public Command goLow() { return elevator.run(Meters.of(0)); }
}

// RobotContainer.java
private void configureBindings() {
  // Button toggles between high and low
  m_driverController.a()
    .and(m_elevator.isLow)
    .onTrue(m_elevator.goHigh());
    
  m_driverController.a()
    .and(m_elevator.isHigh)
    .onTrue(m_elevator.goLow());
}
```

## Understanding Motor Behavior

Both `run()` and `runTo()` set the `SmartMotorController`'s setpoint. When the command ends:

- The **command** stops running
- The **SmartMotorController** does NOT stop
- The motor continues driving toward the last setpoint

This is intentional! It allows for smooth transitions between commands without the mechanism "going limp" between commands.

### If You Need to Stop the Motor

If you actually want to stop motor output when a command ends, use `finallyDo()`:

```java
public Command setHeightAndStopMotor(Distance height, Distance tolerance) {
  return elevator.runTo(height, tolerance)
    .finallyDo(() -> elevator.setDutyCycleSetpoint(0)); // Stops the closed loop and outputs 0
}
```

Or create a dedicated stop command:

```java
public Command stop() {
  return runOnce(() -> elevator.setDutyCycleSetpoint(0)); // Open loop 0
}
```

## Autonomous with runTo()

`runTo()` shines in autonomous routines where you need sequential operations:

```java
public Command autoRoutine() {
  return Commands.sequence(
    // Step 1: Position the arm (runTo ends when reached)
    m_arm.setAngleAndStop(Degrees.of(45), Degrees.of(1)),
    
    // Steps 2 and 3: Spin up the shooter and wait until it is at speed.
    // runTo() ends at speed, and the shooter keeps spinning because the motor is not stopped.
    m_shooter.runTo(RPM.of(4000), RPM.of(100)),
    
    // Step 4: Feed the game piece
    m_feeder.feedForward().withTimeout(0.5),
    
    // Step 5: Stop shooter and return arm
    Commands.parallel(
      m_shooter.setVoltage(Volts.of(0)),  // Stop the shooter
      m_arm.setAngleAndStop(Degrees.of(0), Degrees.of(1))
    )
  );
}
```

## Commands v3

With the Commands v3 vendordep, import the mechanisms from `yams.commands3.mechanisms` and `Command`/`Trigger` from `org.wpilib.command3`. The method names are the same, but they are built as coroutines:

| Method | Commands v3 behavior |
|--------|----------------------|
| `run(setpoint)`, `setAngle()`, `setHeight()` | A `while (true)` loop that sets the setpoint and calls `coroutine.yield()`. Never ends on its own. |
| `runTo(setpoint, tolerance)` | Sets the setpoint once, then waits with `coroutine.waitUntil(near(setpoint, tolerance).debounce(...))`. Ends when the goal is reached. |
| `set(dutycycle)`, `setVoltage(volts)` | A `while (true)` loop that stops the closed loop controller, and restarts it when the command is canceled. |

The motor still keeps its last request when a command ends, and the default command pitfall still applies: when `runTo()` ends, the mechanism's default command takes over.

### Default commands

Write default commands as coroutines at the lowest priority, so any other command on the mechanism can take over:

```java
public class ElevatorMechanism implements Mechanism {
  // SmartMotorControllerConfig(this), SmartMotorController and Elevator elevator ...

  public Command hold(Distance height) {
    return run(coroutine -> coroutine.await(elevator.setHeight(height)))
        .withPriority(Command.LOWEST_PRIORITY)
        .named("Elevator Hold " + height.in(Meters) + " m");
  }

  public Command moveTo(Distance height) {
    return elevator.runTo(height, Centimeters.of(2));
  }
}

// Robot.java
elevator.setDefaultCommand(elevator.hold(Meters.of(0)));
```

### Sequences

Instead of `Commands.sequence(...)` and `andThen(...)`, await each `runTo()` inside a coroutine. `Command.noRequirements(...)` keeps the sequence itself from owning any mechanism; each mechanism is only owned while its own command runs:

```java
public Command autoScoreHigh() {
  return Command.noRequirements(coroutine -> {
    // Each moveTo() ends when its mechanism reaches the goal
    coroutine.awaitAll(elevator.moveTo(Meters.of(1.2)), arm.moveTo(Degrees.of(45)));
    // Score the game piece for 0.5 seconds
    coroutine.awaitAny(intake.outtake(), Command.waitFor(Seconds.of(0.5)).named("Outtake Time"));
    coroutine.awaitAll(arm.moveTo(Degrees.of(0)), elevator.moveTo(Meters.of(0)));
  }).named("Auto Score High");
}
```

### Avoiding the default command pitfall

Besides the Commands v2 solutions above, Commands v3 lets you set a default command from inside a running command. It only lasts until that command ends, then the global default comes back:

```java
public Command scoreHigh() {
  return Command.noRequirements(coroutine -> {
    // After moveTo() ends, hold the scoring height instead of returning to the global default
    elevator.setDefaultCommand(elevator.hold(Meters.of(1.2)));
    coroutine.await(elevator.moveTo(Meters.of(1.2)));
    coroutine.wait(Seconds.of(1));
  }).named("Score High");
}
```

### Stopping the motor

Instead of a `finallyDo()` decorator, await `runTo()` and then stop the motor in the same coroutine:

```java
public Command setHeightAndStopMotor(Distance height, Distance tolerance) {
  return run(coroutine -> {
    coroutine.await(elevator.runTo(height, tolerance));
    elevator.setDutyCycleSetpoint(0); // Stops the closed loop and outputs 0
  }).named("Elevator To Height Then Stop");
}
```

## Common Pitfalls Summary

| Pitfall | Symptom | Solution |
|---------|---------|----------|
| `runTo()` with default command | Mechanism returns to default position after reaching goal | Use `run()` or remove default command |
| Using `onTrue()` with `run()` expecting it to end | Command runs forever, can't do anything else | Use `runTo()` for finite commands |
| Assuming motor stops when command ends | Mechanism drifts or holds unexpected position | Understand that `SmartMotorController` continues |
| Tolerance smaller than the steady-state error | `runTo()` never ends because goal is never "reached" | Pass a larger tolerance to `runTo()` or tune the controller |
| Tolerance too tight | `runTo()` ends and restarts repeatedly (oscillation) | Increase tolerance or add debounce |

## Decision Flowchart

```
Do you want the command to end automatically?
├── NO → Use run()
│   └── Combine with whileTrue() for teleop
│   └── Use as default command
└── YES → Use runTo()
    └── Is there a default command?
        ├── YES → Will it conflict with your goal?
        │   ├── YES → Remove default or use run() instead
        │   └── NO → runTo() is fine
        └── NO → runTo() is fine
```

## Best Practices

1. **Default commands should use `run()`** - They run forever by definition (in Commands v3, also give them `Command.LOWEST_PRIORITY`)
2. **Teleop with `whileTrue()` should use `run()`** - Natural pairing
3. **Autonomous sequences should use `runTo()`** - Need to know when steps complete
4. **Be explicit about intent** - Name methods clearly (`setHeight` vs `setHeightAndStop`)
5. **Consider state-based control** - Use triggers instead of complex command logic
6. **Test both paths** - Test what happens when commands end, not just when they run

## Further Reading

- [Command-Based Best Practices](https://bovlb.github.io/frc-tips/commands/best-practices.html) - Comprehensive guide on command-based architecture
- [WPILib Command Scheduler](https://docs.wpilib.org/en/stable/docs/software/commandbased/command-scheduler.html) - How commands are scheduled
- [Default Commands](https://docs.wpilib.org/en/stable/docs/software/commandbased/subsystems.html#default-commands) - Official documentation on default commands
