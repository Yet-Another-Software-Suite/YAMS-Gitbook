---
icon: elevator
---

# Elevators

An `Elevator` moves a carriage along a line: a lift, a telescoping climber, or a linear slide.

## How an elevator works

<figure><img src="../.gitbook/assets/mech-elevator-how.png" alt="A motor drives a gearbox, set by withGearing, which turns a drum or sprocket, set by withDrumRadius, which pulls a carriage up a rail. One drum rotation lifts the carriage one circumference, so the height is the drum rotations times the circumference and you work in meters."><figcaption><p>A rotating motor becomes a distance.</p></figcaption></figure>

The motor turns a drum or sprocket through a gearbox, and the drum pulls the carriage. One rotation of the drum moves the carriage one circumference, so once YAMS knows the [drum size](editor/config/gearing-and-units.md#withmechanismcircumference) you can work in meters and never think about rotations. [How to find your mechanism circumference?](../how-to/how-to-find-your-mechanism-circumference.md) covers measuring it.

A vertical elevator has to hold its carriage up, which is what the `kG` term of [`ElevatorFeedforward`](editor/config/feedforward.md#withfeedforward) is for.

<figure><img src="../.gitbook/assets/mech-flow.png" alt="A flow diagram. A command such as arm.run or arm.runTo gives a setpoint to the mechanism, which gives it to the SmartMotorController. The controller runs the profile, PID, feedforward and limits and sends volts to the motor, and reads back the encoder position and velocity. updateTelemetry is called every loop to publish values and draw the mechanism. simIterate is called every loop in simulation to step the physics."><figcaption><p>The same chain runs every mechanism.</p></figcaption></figure>

A YAMS `Elevator` is a thin layer over one [`SmartMotorController`](editor/README.md). The `Elevator` knows what the mechanism is, turns your commands into setpoints, and runs its simulation. The motor controller does the closed loop control. Keep one mechanism per subsystem, so live tuning and command requirements stay simple.

## Creating an elevator

The code is the same for Commands v2 and Commands v3 except for the package you import from and the class that owns the mechanism.

{% tabs %}
{% tab title="Commands v2" %}
```java
import yams.commands2.config.SmartMotorControllerConfig;
import yams.commands2.mechanisms.Elevator;

public class ElevatorSubsystem extends SubsystemBase
{
  private final SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
      .withDrumRadius(Inches.of(0.25), 22) // chain pitch and sprocket teeth
      .withClosedLoopController(4, 0, 0)
      .withFeedforward(new ElevatorFeedforward(0, 0.4, 3.5))
      .withTrapezoidalProfile(MetersPerSecond.of(1.5), MetersPerSecondPerSecond.of(3))
      .withSoftLimits(Meters.of(0), Meters.of(1.25))
      .withStartingPosition(Meters.of(0))
      .withStatorCurrentLimit(Amps.of(40))
      .withZeroPower(MotorMode.BRAKE)
      .withTelemetry("ElevatorMotor", TelemetryVerbosity.HIGH);
  private final SmartMotorController motor = new SparkWrapper(spark, DCMotor.getNEO(1), motorConfig);
  private final ElevatorConfig config = new ElevatorConfig()
      .withCarriageWeight(Pounds.of(16))
      .withHardLimits(Meters.of(0), Meters.of(1.3))
      .withTelemetry("Elevator", TelemetryVerbosity.HIGH);
  private final Elevator elevator = new Elevator(config, motor);

  @Override
  public void periodic()
  {
    elevator.updateTelemetry(); // every loop
  }

  @Override
  public void simulationPeriodic()
  {
    elevator.simIterate(); // every loop, in simulation
  }
}
```

`this` is the `Subsystem`. The scheduler calls `periodic()` and `simulationPeriodic()` for you.
{% endtab %}

{% tab title="Commands v3" %}
```java
import yams.commands3.config.SmartMotorControllerConfig;
import yams.commands3.mechanisms.Elevator;

public class ElevatorMechanism implements Mechanism
{
  private final SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
      .withDrumRadius(Inches.of(0.25), 22) // chain pitch and sprocket teeth
      .withClosedLoopController(4, 0, 0)
      .withFeedforward(new ElevatorFeedforward(0, 0.4, 3.5))
      .withTrapezoidalProfile(MetersPerSecond.of(1.5), MetersPerSecondPerSecond.of(3))
      .withSoftLimits(Meters.of(0), Meters.of(1.25))
      .withStartingPosition(Meters.of(0))
      .withStatorCurrentLimit(Amps.of(40))
      .withZeroPower(MotorMode.BRAKE)
      .withTelemetry("ElevatorMotor", TelemetryVerbosity.HIGH);
  private final SmartMotorController motor = new SparkWrapper(spark, DCMotor.getNEO(1), motorConfig);
  private final ElevatorConfig config = new ElevatorConfig()
      .withCarriageWeight(Pounds.of(16))
      .withHardLimits(Meters.of(0), Meters.of(1.3))
      .withTelemetry("Elevator", TelemetryVerbosity.HIGH);
  private final Elevator elevator = new Elevator(config, motor);

  public void periodic()
  {
    elevator.updateTelemetry(); // every loop
  }

  public void simulationPeriodic()
  {
    elevator.simIterate(); // every loop, in simulation
  }
}
```

`this` is the Commands v3 `Mechanism`. A `Mechanism` has no periodic hook, so call `periodic()` and `simulationPeriodic()` yourself from `Robot.robotPeriodic()`.
{% endtab %}
{% endtabs %}

The options in `motorConfig` are explained in [SmartMotorControllerConfig](editor/config/README.md), and the options in `config` in [ElevatorConfig](mechanism-configs/elevator-config.md).

## Commands

Every command below exists with the same name in Commands v2 and Commands v3.

<figure><img src="../.gitbook/assets/mech-commands.png" alt="Three timelines starting when the command is scheduled. run(setpoint), also called setAngle or setHeight, runs until something else takes the mechanism and holds the setpoint the whole time. runTo(setpoint, tolerance) ends when the mechanism is within the tolerance, and then the default command takes over. set(dutycycle) and setVoltage(volts) are open loop with no PID, run until interrupted, and the closed loop controller comes back when they end."><figcaption><p>When each kind of command ends.</p></figcaption></figure>

| Command | What it does | Ends |
| --- | --- | --- |
| `setHeight(Distance)`, `run(Distance)` | Go to the height and hold it. The two are the same. | Never, until interrupted. |
| `runTo(Distance, Distance tolerance)` | Go to the height. | When the elevator is within the tolerance. |
| `set(double)` | Output a duty cycle, with the closed loop off. | Never, until interrupted. |
| `setVoltage(Voltage)` | Output a voltage, with the closed loop off. | Never, until interrupted. |

`setHeight`, `run` and `runTo` also take a `Supplier<Distance>`, and `set` and `setVoltage` a supplier too, for a setpoint that changes while the command runs.

{% tabs %}
{% tab title="Commands v2" %}
```java
// In the subsystem
public Command hold(Distance height)
{
  return elevator.setHeight(height); // never ends: a good default command
}

public Command moveTo(Distance height)
{
  return elevator.runTo(height, Centimeters.of(2)); // ends when it gets there
}

// In RobotContainer
elevator.setDefaultCommand(elevator.hold(Meters.of(0)));
Command score = elevator.moveTo(Meters.of(1.2)).andThen(intake.eject());
```
{% endtab %}

{% tab title="Commands v3" %}
```java
// In the mechanism
public Command hold(Distance height)
{
  return run(coroutine -> coroutine.await(elevator.setHeight(height)))
      .withPriority(Command.LOWEST_PRIORITY) // so any other command can take over
      .named("Elevator Hold");
}

public Command moveTo(Distance height)
{
  return elevator.runTo(height, Centimeters.of(2)); // ends when it gets there
}

// In Robot
elevator.setDefaultCommand(elevator.hold(Meters.of(0)));
Command score = Command.noRequirements(coroutine -> {
  coroutine.await(elevator.moveTo(Meters.of(1.2)));
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
elevator.setMeasurementPositionSetpoint(Meters.of(0.8));
```

These do not require the subsystem or mechanism, so nothing stops two callers fighting over the setpoint. Prefer the commands unless you need these. [Setpoint Methods vs Command Helpers](setpoint-methods.md) compares the two in detail.

## Asking where it is

<figure><img src="../.gitbook/assets/mech-triggers.png" alt="A number line from minus 30 to 120 degrees with an arm at 72 degrees. near 80 within 10 is true. between 0 and 60 is false. gte 45 is true. lte 45 is false. max and min, at the ends of the travel, are false. Each is a Trigger, and isNear, isBetween, isGte, isLte, isAtMax and isAtMin return the same answers as booleans."><figcaption><p>The same questions as Triggers or as booleans.</p></figcaption></figure>

| As a `Trigger` | As a boolean | True when |
| --- | --- | --- |
| `near(height, within)` | `isNear(height, within)` | The elevator is within `within` of `height`. |
| `between(start, end)` | `isBetween(start, end)` | The elevator is between the two heights. |
| `gte(height)`, `lte(height)` | `isGte(height)`, `isLte(height)` | The elevator is at or above, or at or below, the height. |
| `max()`, `min()` | `isAtMax()`, `isAtMin()` | The elevator is at its soft limit, or its hard limit if no soft limit is set. |

`getHeight()` returns the current height and `getVelocity()` its speed. The graphic above shows an arm, and an elevator works the same way with heights.

```java
elevator.near(Meters.of(1.2), Centimeters.of(2)).onTrue(intake.eject());
if (elevator.isAtMin()) { /* fully down */ }
```

## Which kind of elevator do you have?

<figure><img src="../.gitbook/assets/mech-elevator-kinds.png" alt="Five kinds of elevator and what each needs. Drum, rope or cable on a spool: withDrumRadius with the radius. Chain or belt, a sprocket or pulley: withDrumRadius with the pitch and teeth. Rack and pinion, a gear on a toothed bar: withMechanismCircumference with the pinion pitch diameter times pi. Cascading, each stage lifts the next: withCascadingElevatorStages. Continuous, one rope through all stages: nothing extra."><figcaption><p>Five kinds of elevator and what each changes in the config.</p></figcaption></figure>

Two questions decide the config: how the motor moves the elevator, and how the stages are rigged. Answer both.

### How the motor moves it

{% tabs %}
{% tab title="Drum" %}
A rope, cable or strap winds onto a drum. Give the drum's radius, measured to the middle of the rope where it sits on the drum.

```java
.withGearing(new MechanismGearing(GearBox.fromReductionStages(5, 4)))
.withDrumRadius(Inches.of(0.75))
```

If the rope winds over itself, the radius grows as the elevator rises and the height reads short near the top. Use a drum wide enough for one layer, or measure the travel and work the circumference back from it.
{% endtab %}

{% tab title="Chain driven" %}
A sprocket drives a chain that carries the carriage. The chain moves one pitch for every tooth, so give the chain pitch and the sprocket's tooth count and YAMS works out the rest.

```java
.withGearing(new MechanismGearing(GearBox.fromReductionStages(5, 4)))
.withDrumRadius(Inches.of(0.25), 22) // #25 chain is 0.25 in pitch, #35 is 0.375 in; 22 teeth
```

A timing belt on a pulley works the same way: give the belt pitch and the pulley's tooth count.
{% endtab %}

{% tab title="Rack and pinion" %}
A pinion gear on the motor runs along a toothed rack. One turn of the pinion moves the carriage the pinion's pitch circumference.

```java
.withGearing(new MechanismGearing(GearBox.fromReductionStages(5)))
.withMechanismCircumference(Inches.of(1.5).times(Math.PI)) // pitch diameter x pi
```

Use the pitch diameter from the gear's datasheet, not the outside diameter. For a metric gear the pitch diameter is the module times the tooth count.

Most rack and pinion mechanisms are level linear slides. Tell the simulation there is no gravity to hold against, and expect to need [separate simulation gains](editor/simulation-only-pid-+-feedforward.md):

```java
new ElevatorConfig()
    .withAngle(Degrees.of(0))  // drawn level
    .withHorizontalElevator()  // no gravity in simulation
```

A level slide needs no `kG`, so its `ElevatorFeedforward` has `kG` of 0.
{% endtab %}
{% endtabs %}

### How the stages are rigged

{% tabs %}
{% tab title="One stage" %}
The carriage rides directly on the fixed frame. There is nothing more to set.
{% endtab %}

{% tab title="Cascading" %}
Each stage lifts the next one, so the carriage moves further than the part the motor pulls: twice as far with two moving stages, three times with three. Tell YAMS how many, and it divides the gearing to match.

```java
.withGearing(new MechanismGearing(GearBox.fromReductionStages(5, 4)))
.withDrumRadius(Inches.of(0.25), 22)
.withCascadingElevatorStages(2)
```

Call it after `withGearing`. See [`withCascadingElevatorStages`](editor/config/gearing-and-units.md#withcascadingelevatorstages). Every height you give and read is then the carriage's height.
{% endtab %}

{% tab title="Continuous" %}
One rope or belt runs through every stage. The stages may rise in any order, but the carriage always rises by the length of rope pulled, so a continuous elevator is configured like a single stage: nothing extra to set.

```java
.withGearing(new MechanismGearing(GearBox.fromReductionStages(5, 4)))
.withDrumRadius(Inches.of(0.75))
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Whatever the kind, check it the same way: command a move of a known distance and measure how far the carriage really went. If it is off by a constant factor, the circumference or the stage count is wrong.
{% endhint %}

### Elevators with a constant force spring

A spring that helps lift the carriage changes what `kG` should be. See [Elevators with a constant force spring](live-tuning/README.md#elevators-with-a-constant-force-spring).

## Hard limits and soft limits

Hard limits are where the physical stops are, and the simulated elevator cannot pass them. Soft limits are where the motor controller keeps the real elevator. Set both, with the soft limits a little inside: see [`withHardLimits`](mechanism-configs/shared-options.md#withhardlimits).

## Configuring an elevator

Every `ElevatorConfig` option is on the [ElevatorConfig](mechanism-configs/elevator-config.md) page under this one, with a graphic and an example for each. Options shared by every mechanism are in [Shared options](mechanism-configs/shared-options.md).

An `Elevator` needs its carriage weight, hard limits, a drum size and a starting position before it can be created in simulation: see [what simulation needs](mechanism-configs/README.md#what-simulation-needs).

## Code Reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/ElevatorSubsystem.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands3/simple_robot-2027/src/main/java/first/robot/mechanisms/ElevatorMechanism.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/ExponentiallyProfiledElevatorSubsystem.java" %}
