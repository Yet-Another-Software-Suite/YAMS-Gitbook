---
icon: rotate
---

# Turrets/Wrists

Turrets and wrists are both a `Pivot` in YAMS: a mechanism that turns to an angle and is not pulled on by gravity.

## How a pivot works

<figure><img src="../.gitbook/assets/mech-pivot-vs-arm.png" alt="Two panels. An arm swings up and down, gravity pulls on it, it needs kG to hold and uses ArmFeedforward. A pivot, seen from above, turns about a vertical axis or is balanced, needs no kG and uses SimpleMotorFeedforward."><figcaption><p>A pivot is an arm that gravity does not pull on.</p></figcaption></figure>

A pivot is controlled exactly like an arm, with one difference: there is no gravity term. A turret turns about a vertical axis, and a balanced wrist has its weight centered on its shaft, so neither needs voltage just to stay where it is. That makes a pivot simpler to tune, and it is why a pivot uses [`SimpleMotorFeedforward`](editor/config/feedforward.md#withfeedforward) where an arm uses `ArmFeedforward`.

If the mechanism droops when the power is off, gravity does matter: use an [Arm](arms.md).

<figure><img src="../.gitbook/assets/mech-flow.png" alt="A flow diagram. A command such as arm.run or arm.runTo gives a setpoint to the mechanism, which gives it to the SmartMotorController. The controller runs the profile, PID, feedforward and limits and sends volts to the motor, and reads back the encoder position and velocity. updateTelemetry is called every loop to publish values and draw the mechanism. simIterate is called every loop in simulation to step the physics."><figcaption><p>The same chain runs every mechanism.</p></figcaption></figure>

A YAMS `Pivot` is a thin layer over one [`SmartMotorController`](editor/README.md). The `Pivot` knows what the mechanism is, turns your commands into setpoints, and runs its simulation. The motor controller does the closed loop control. Keep one mechanism per subsystem, so live tuning and command requirements stay simple.

## Creating a pivot

The code is the same for Commands v2 and Commands v3 except for the package you import from and the class that owns the mechanism.

{% tabs %}
{% tab title="Commands v2" %}
```java
import yams.commands2.config.SmartMotorControllerConfig;
import yams.commands2.mechanisms.Pivot;

public class PivotSubsystem extends SubsystemBase
{
  private final SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
      .withClosedLoopController(4, 0, 0)
      .withTrapezoidalProfile(DegreesPerSecond.of(180), DegreesPerSecondPerSecond.of(90))
      .withSoftLimits(Degrees.of(-170), Degrees.of(170))
      .withStartingPosition(Degrees.of(0))
      .withMomentOfInertia(Meters.of(0.25), Pounds.of(4))
      .withStatorCurrentLimit(Amps.of(40))
      .withZeroPower(MotorMode.BRAKE)
      .withTelemetry("TurretMotor", TelemetryVerbosity.HIGH);
  private final SmartMotorController motor = new SparkWrapper(spark, DCMotor.getNEO(1), motorConfig);
  private final PivotConfig config = new PivotConfig()
      .withHardLimits(Degrees.of(-180), Degrees.of(180))
      .withTelemetry("Turret", TelemetryVerbosity.HIGH);
  private final Pivot pivot = new Pivot(config, motor);

  @Override
  public void periodic()
  {
    pivot.updateTelemetry(); // every loop
  }

  @Override
  public void simulationPeriodic()
  {
    pivot.simIterate(); // every loop, in simulation
  }
}
```

`this` is the `Subsystem`. The scheduler calls `periodic()` and `simulationPeriodic()` for you.
{% endtab %}

{% tab title="Commands v3" %}
```java
import yams.commands3.config.SmartMotorControllerConfig;
import yams.commands3.mechanisms.Pivot;

public class PivotMechanism implements Mechanism
{
  private final SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
      .withClosedLoopController(4, 0, 0)
      .withTrapezoidalProfile(DegreesPerSecond.of(180), DegreesPerSecondPerSecond.of(90))
      .withSoftLimits(Degrees.of(-170), Degrees.of(170))
      .withStartingPosition(Degrees.of(0))
      .withMomentOfInertia(Meters.of(0.25), Pounds.of(4))
      .withStatorCurrentLimit(Amps.of(40))
      .withZeroPower(MotorMode.BRAKE)
      .withTelemetry("TurretMotor", TelemetryVerbosity.HIGH);
  private final SmartMotorController motor = new SparkWrapper(spark, DCMotor.getNEO(1), motorConfig);
  private final PivotConfig config = new PivotConfig()
      .withHardLimits(Degrees.of(-180), Degrees.of(180))
      .withTelemetry("Turret", TelemetryVerbosity.HIGH);
  private final Pivot pivot = new Pivot(config, motor);

  public void periodic()
  {
    pivot.updateTelemetry(); // every loop
  }

  public void simulationPeriodic()
  {
    pivot.simIterate(); // every loop, in simulation
  }
}
```

`this` is the Commands v3 `Mechanism`. A `Mechanism` has no periodic hook, so call `periodic()` and `simulationPeriodic()` yourself from `Robot.robotPeriodic()`.
{% endtab %}
{% endtabs %}

The options in `motorConfig` are explained in [SmartMotorControllerConfig](editor/config/README.md), and the options in `config` in [PivotConfig](mechanism-configs/pivot-config.md).

## Commands

Every command below exists with the same name in Commands v2 and Commands v3.

<figure><img src="../.gitbook/assets/mech-commands.png" alt="Three timelines starting when the command is scheduled. run(setpoint), also called setAngle or setHeight, runs until something else takes the mechanism and holds the setpoint the whole time. runTo(setpoint, tolerance) ends when the mechanism is within the tolerance, and then the default command takes over. set(dutycycle) and setVoltage(volts) are open loop with no PID, run until interrupted, and the closed loop controller comes back when they end."><figcaption><p>When each kind of command ends.</p></figcaption></figure>

| Command | What it does | Ends |
| --- | --- | --- |
| `setAngle(Angle)`, `run(Angle)` | Go to the angle and hold it. The two are the same. | Never, until interrupted. |
| `runTo(Angle, Angle tolerance)` | Go to the angle. | When the pivot is within the tolerance. |
| `set(double)` | Output a duty cycle, with the closed loop off. | Never, until interrupted. |
| `setVoltage(Voltage)` | Output a voltage, with the closed loop off. | Never, until interrupted. |

`setAngle`, `run` and `runTo` also take a `Supplier<Angle>`, which is how a turret tracks a moving target: give it a supplier that returns the angle to the target.

{% tabs %}
{% tab title="Commands v2" %}
```java
// In the subsystem
public Command hold(Angle angle)
{
  return pivot.setAngle(angle); // never ends: a good default command
}

public Command moveTo(Angle angle)
{
  return pivot.runTo(angle, Degrees.of(1)); // ends when it gets there
}

// In RobotContainer
turret.setDefaultCommand(turret.hold(Degrees.of(0)));
Command score = turret.moveTo(Degrees.of(90)).andThen(intake.eject());
```
{% endtab %}

{% tab title="Commands v3" %}
```java
// In the mechanism
public Command hold(Angle angle)
{
  return run(coroutine -> coroutine.await(pivot.setAngle(angle)))
      .withPriority(Command.LOWEST_PRIORITY) // so any other command can take over
      .named("Pivot Hold");
}

public Command moveTo(Angle angle)
{
  return pivot.runTo(angle, Degrees.of(1)); // ends when it gets there
}

// In Robot
turret.setDefaultCommand(turret.hold(Degrees.of(0)));
Command score = Command.noRequirements(coroutine -> {
  coroutine.await(turret.moveTo(Degrees.of(90)));
  coroutine.await(intake.eject());
}).named("Score");
```
{% endtab %}
{% endtabs %}

When a command ends the motor keeps its last request, so a mechanism that reached its goal stays there until another command, usually the default one, gives it something else. [run() vs runTo() Commands](run-vs-runto.md) covers the pitfalls, including a default command undoing a `runTo()`.

## Setpoint functions

The commands above are built on plain functions that set the setpoint and return at once. Use them when you write your own command, or when something other than a command decides the setpoint, such as an AdvantageKit IO layer.

| Function | What it does |
| --- | --- |
| `setMechanismPositionSetpoint(Angle)` | Go to this angle with closed loop control. |
| `setMeasurementPositionSetpoint(Distance)` | Go to this distance. Needs a [circumference](editor/config/gearing-and-units.md#withmechanismcircumference). |
| `setVoltageSetpoint(Voltage)` | Output this voltage. Stops the closed loop. |
| `setDutyCycleSetpoint(double)` | Output this duty cycle, from -1 to 1. Stops the closed loop. |
| `getMechanismSetpoint()` | The current position setpoint, if there is one. |

```java
pivot.setMechanismPositionSetpoint(Degrees.of(30));
```

These do not require the subsystem or mechanism, so nothing stops two callers fighting over the setpoint. Prefer the commands unless you need these. [Setpoint Methods vs Command Helpers](setpoint-methods.md) compares the two in detail.

## Asking where it is

<figure><img src="../.gitbook/assets/mech-triggers.png" alt="A number line from minus 30 to 120 degrees with an arm at 72 degrees. near 80 within 10 is true. between 0 and 60 is false. gte 45 is true. lte 45 is false. max and min, at the ends of the travel, are false. Each is a Trigger, and isNear, isBetween, isGte, isLte, isAtMax and isAtMin return the same answers as booleans."><figcaption><p>The same questions as Triggers or as booleans.</p></figcaption></figure>

| As a `Trigger` | As a boolean | True when |
| --- | --- | --- |
| `near(angle, within)` | `isNear(angle, within)` | The pivot is within `within` of `angle`. |
| `between(start, end)` | `isBetween(start, end)` | The pivot is between the two angles. |
| `gte(angle)`, `lte(angle)` | `isGte(angle)`, `isLte(angle)` | The pivot is at or above, or at or below, the angle. |
| `max()`, `min()` | `isAtMax()`, `isAtMin()` | The pivot is at its soft limit, or its hard limit if no soft limit is set. |

`getAngle()` returns the pivot's current angle.

```java
turret.near(Degrees.of(90), Degrees.of(1)).whileTrue(shooter.fire());
```

## Turrets that spin all the way round

| The turret | What to set |
| --- | --- |
| Can spin forever (a slip ring, or nothing to wind up) | [`withContinuousWrapping`](editor/config/closed-loop-control.md#withcontinuouswrapping) on the motor config, and no soft limits, so it takes the short way round. |
| Is limited by its wiring | [`withSoftLimits`](editor/config/limits-and-protection.md#withsoftlimits), and hard limits a little wider. It may have to unwind the long way. |
| Is geared so its encoders turn more than once over its travel | [EasyCRT](easycrt.md), which works out the turret's true angle from two encoders. |

## Moment of inertia

How heavy the pivot is to turn is set on the motor config with [`withMomentOfInertia`](editor/config/simulation.md#withmomentofinertia). It only matters in simulation, where it also decides the current the pivot draws from the [battery simulation](../understanding/battery-simulation.md).

## Configuring a pivot

Every `PivotConfig` option is on the [PivotConfig](mechanism-configs/pivot-config.md) page under this one, with a graphic and an example for each. Options shared by every mechanism are in [Shared options](mechanism-configs/shared-options.md).

A `Pivot` needs its hard limits and a starting position before it can be created in simulation: see [what simulation needs](mechanism-configs/README.md#what-simulation-needs).

## Code Reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/TurretSubsystem.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands3/simple_robot-2027/src/main/java/first/robot/mechanisms/TurretMechanism.java" %}
