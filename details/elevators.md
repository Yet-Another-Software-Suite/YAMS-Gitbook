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

A YAMS `Elevator` is a thin layer over one [`SmartMotorController`](editor/). The `Elevator` knows what the mechanism is, turns your commands into setpoints, and runs its simulation. The motor controller does the closed loop control. Keep one mechanism per subsystem, so live tuning and command requirements stay simple.

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

The options in `motorConfig` are explained in [SmartMotorControllerConfig](editor/config/), and the options in `config` in [ElevatorConfig](mechanism-configs/elevator-config.md).

## Commands

Every command below exists with the same name in Commands v2 and Commands v3.

<figure><img src="../.gitbook/assets/mech-commands.png" alt="Three timelines starting when the command is scheduled. run(setpoint), also called setAngle or setHeight, runs until something else takes the mechanism and holds the setpoint the whole time. runTo(setpoint, tolerance) ends when the mechanism is within the tolerance, and then the default command takes over. set(dutycycle) and setVoltage(volts) are open loop with no PID, run until interrupted, and the closed loop controller comes back when they end."><figcaption><p>When each kind of command ends.</p></figcaption></figure>

| Command                                | What it does                                        | Ends                                       |
| -------------------------------------- | --------------------------------------------------- | ------------------------------------------ |
| `setHeight(Distance)`, `run(Distance)` | Go to the height and hold it. The two are the same. | Never, until interrupted.                  |
| `runTo(Distance, Distance tolerance)`  | Go to the height.                                   | When the elevator is within the tolerance. |
| `set(double)`                          | Output a duty cycle, with the closed loop off.      | Never, until interrupted.                  |
| `setVoltage(Voltage)`                  | Output a voltage, with the closed loop off.         | Never, until interrupted.                  |

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

| Function                                   | What it does                                                                                                                                         |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `setMechanismPositionSetpoint(Angle)`      | Go to this angle with closed loop control.                                                                                                           |
| `setMeasurementPositionSetpoint(Distance)` | Go to this distance. Needs a [circumference](editor/config/gearing-and-units.md#withmechanismcircumference).                                         |
| `setVoltageSetpoint(Voltage)`              | Output this voltage. Stops the closed loop.                                                                                                          |
| `setDutyCycleSetpoint(double)`             | Output this duty cycle, from -1 to 1. Stops the closed loop.                                                                                         |
| `getMechanismSetpoint()`                   | The current position setpoint as an angle, if there is one.                                                                                          |
| `getMeasurementSetpoint()`                 | The same setpoint as a distance. Empty if there is no setpoint or no [circumference](editor/config/gearing-and-units.md#withmechanismcircumference). |

```java
elevator.setMeasurementPositionSetpoint(Meters.of(0.8));
```

These do not require the subsystem or mechanism, so nothing stops two callers fighting over the setpoint. Prefer the commands unless you need these. [Setpoint Methods vs Command Helpers](setpoint-methods.md) compares the two in detail.

## Asking where it is

<figure><img src="../.gitbook/assets/mech-triggers-elevator.png" alt="A number line from 0 to 1.5 meters with an elevator at 1.18 meters. near 1.2 meters within 5 centimeters is true. between 0.25 and 0.75 meters is false. gte 0.5 meters is true. lte 0.5 meters is false. max and min, at the top and bottom of the travel, are false. Each is a Trigger, and the is functions such as isNear give the same answers as booleans."><figcaption><p>The same questions as Triggers or as booleans.</p></figcaption></figure>

| As a `Trigger`               | As a boolean                     | True when                                                                     |
| ---------------------------- | -------------------------------- | ----------------------------------------------------------------------------- |
| `near(height, within)`       | `isNear(height, within)`         | The elevator is within `within` of `height`.                                  |
| `between(start, end)`        | `isBetween(start, end)`          | The elevator is between the two heights.                                      |
| `gte(height)`, `lte(height)` | `isGte(height)`, `isLte(height)` | The elevator is at or above, or at or below, the height.                      |
| `max()`, `min()`             | `isAtMax()`, `isAtMin()`         | The elevator is at its soft limit, or its hard limit if no soft limit is set. |

`getHeight()` returns the current height and `getVelocity()` its speed.

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

To show each stage on an AdvantageScope robot model, also give the `ElevatorConfig` the stages with [`withStages`](mechanism-configs/elevator-config.md#withstages), and publish `elevator.getCascadePose3d()`: every stage moves at once, each by its share of the carriage's height.
{% endtab %}

{% tab title="Continuous" %}
One rope or belt runs through every stage. The stages may rise in any order, but the carriage always rises by the length of rope pulled, so a continuous elevator is configured like a single stage: nothing extra to set.

```java
.withGearing(new MechanismGearing(GearBox.fromReductionStages(5, 4)))
.withDrumRadius(Inches.of(0.75))
```

To show each stage on an AdvantageScope robot model, give the `ElevatorConfig` the stages with [`withStages`](mechanism-configs/elevator-config.md#withstages), and publish `elevator.getContinuousPose3d()`. It shows the carriage rising the whole of its travel first, then the stage below it, and so on. If your stages rise in another order, the carriage is still drawn at its height, but the stages below it are not.
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Whatever the kind, check it the same way: command a move of a known distance and measure how far the carriage really went. If it is off by a constant factor, the circumference or the stage count is wrong.
{% endhint %}

### Elevators with a constant force spring

A spring that helps lift the carriage changes what `kG` should be. See [Elevators with a constant force spring](live-tuning/#elevators-with-a-constant-force-spring).

## Hard limits and soft limits

Hard limits are where the physical stops are, and the simulated elevator cannot pass them. Soft limits are where the motor controller keeps the real elevator. Set both, with the soft limits a little inside: see [`withHardLimits`](mechanism-configs/shared-options.md#withhardlimits).

## Configuring and tuning an elevator, step by step

Do the configuration steps first, in simulation, then the tuning steps on the real mechanism with [live tuning](live-tuning/). Tuning something that is configured wrong wastes the time.

{% hint style="danger" %}
Like all tuning, this can be dangerous. Keep clear of the mechanism, start with low current limits and small moves, and practice in simulation first.
{% endhint %}

### Configure it

{% stepper %}
{% step %}
#### Set the gearing

Give the reduction from the motor to the drum, sprocket or pinion with [`withGearing`](editor/config/gearing-and-units.md#withgearing).
{% endstep %}

{% step %}
#### Set how the motor moves it

Give the drum radius, the chain pitch and teeth, or the pinion's pitch circumference: see [how the motor moves it](elevators.md#how-the-motor-moves-it) below.
{% endstep %}

{% step %}
#### Set how the stages are rigged

Add `withCascadingElevatorStages` for a cascading elevator and nothing for a single stage or continuous one: see [how the stages are rigged](elevators.md#how-the-stages-are-rigged).
{% endstep %}

{% step %}
#### Make up positive

Apply a small positive voltage. If the carriage goes down, flip [`withMotorInverted`](editor/config/motor-basics.md#withmotorinverted).
{% endstep %}

{% step %}
#### Check the distance

Command a move of a known distance, such as 0.5 m, and measure the carriage with a tape. If it is off by a constant factor, fix the circumference or the stage count before going any further.
{% endstep %}

{% step %}
#### Set where it starts

Decide what height 0 means, usually the carriage fully down, and set [`withStartingPosition`](editor/config/encoders.md#withstartingposition) to where the carriage is when the robot turns on.
{% endstep %}

{% step %}
#### Set the limits

Put [`withHardLimits`](mechanism-configs/shared-options.md#withhardlimits) at the stops and [`withSoftLimits`](editor/config/limits-and-protection.md#withsoftlimits) a centimeter or two inside them.
{% endstep %}

{% step %}
#### Describe it to the simulator

Set [`withCarriageWeight`](mechanism-configs/elevator-config.md#withcarriageweight), and `withHorizontalElevator()` for a level slide.
{% endstep %}
{% endstepper %}

### Tune it

{% stepper %}
{% step %}
#### Find kG and kS from two voltages

Set kP, kI, kD and kS to 0, so kG is the only voltage reaching the motor. Raise it until the carriage just creeps up, and write it down. Lower it until the carriage just creeps down, and write that down. Then `kG = (up + down) / 2` and `kS = (up - down) / 2`.
{% endstep %}

{% step %}
#### Check it holds

With only the feedforward, the carriage should stay where you leave it at the bottom, the middle and the top.
{% endstep %}

{% step %}
#### Raise kP, then add kD

Raise kP until it reaches the setpoint quickly and stops cleanly. Add a little kD if it overshoots or bounces. Without a profile a new setpoint is a sudden jump, so tune with small moves, a few degrees or centimeters at a time.
{% endstep %}

{% step %}
#### Add the motion profile last

Once the elevator holds and reaches small setpoints cleanly, set a maximum velocity and acceleration it can really reach, and raise them a step at a time.
{% endstep %}

{% step %}
#### Find kV with the profile

kV only acts while the setpoint is moving, so it is tuned with the profile. Run a long move and raise kV until `measurement/velocity` matches the setpoint velocity in the middle of it.
{% endstep %}

{% step %}
#### Check it loaded

Repeat the hold check with a game piece on the carriage.
{% endstep %}
{% endstepper %}

[Tuning order](live-tuning/#tuning-order) has the same steps with the NetworkTables keys to watch.

### What changes for each kind

{% tabs %}
{% tab title="Rack and pinion" %}
A level rack and pinion slide has no weight to hold, so it tunes differently:

* **kG is 0.** Skip the two voltage test for kG. Find kS on its own: the voltage at which the slide just starts to move, in either direction.
* **kS matters more.** A slide's resistance is nearly all friction, so a good kS does most of the work at low speed.
* **Expect backlash.** The gap between the pinion and the rack means the carriage can move a little with the motor still. Do not raise kP to chase it, or the slide will buzz back and forth across the gap. Use a `runTo` tolerance wider than the backlash.
* **Simulate it level.** Use `withHorizontalElevator()`, and give it [its own simulation gains](editor/simulation-only-pid-+-feedforward.md), because the simulation has no friction or backlash.

A rack and pinion that lifts vertically is tuned like any other elevator, with kG.
{% endtab %}

{% tab title="Cascading" %}
Everything is measured at the carriage, which moves several times further than the stage the motor pulls:

* **Check the stage count first.** The distance check in the configuration steps is what catches a wrong `withCascadingElevatorStages`.
* **Expect a larger kG.** The motor lifts the carriage through a mechanical disadvantage, and lifts the moving stages as well, so it needs more voltage to hold than a single stage with the same carriage.
* **Profile limits are the carriage's.** A maximum velocity of 1.5 m/s is the carriage moving at 1.5 m/s. The first stage moves at a fraction of that.
* **Watch the current.** Because of the disadvantage, a cascading elevator draws more current for the same load. If it cannot reach the top, see [when tuning does not help](live-tuning/#when-tuning-does-not-help).
{% endtab %}

{% tab title="Continuous" %}
A continuous elevator is configured like a single stage, but its stages rise in whatever order friction decides:

* **Check kG at several heights.** The weight being lifted changes as each stage starts to move. If the carriage holds at the bottom but drifts higher up, use the kG that holds best in the range you use most and let kP hold the rest.
* **Expect a bump** as each stage picks up or sets down. A little kD smooths it.
{% endtab %}

{% tab title="With a constant force spring" %}
A constant force spring does part of the motor's job, so kG is whatever is left over. It can be small, zero, or negative when the spring is stronger than the carriage.

* **Measure kG, do not calculate it.** Use the two voltage test in the tuning steps. With a spring, the voltage that just moves the carriage down is often negative.
* **Check the hold at the bottom, middle and top.** The springs are not perfectly constant.
* **Give it simulation gains.** The simulation knows nothing about the spring, so the real gains will not hold the simulated carriage. Set [simulation only gains](editor/simulation-only-pid-+-feedforward.md) for it.
* **The spring does not change kV, kP or kD.**

[Elevators with a constant force spring](live-tuning/#elevators-with-a-constant-force-spring) walks through the measurement with graphics.
{% endtab %}
{% endtabs %}

## ElevatorConfig options

Every `ElevatorConfig` option is on the [ElevatorConfig](mechanism-configs/elevator-config.md) page under this one, with a graphic and an example for each. Options shared by every mechanism are in [Shared options](mechanism-configs/shared-options.md).

An `Elevator` needs its carriage weight, hard limits, a drum size and a starting position before it can be created in simulation: see [what simulation needs](mechanism-configs/#what-simulation-needs).

## Code reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/ElevatorSubsystem.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands3/simple_robot-2027/src/main/java/first/robot/mechanisms/ElevatorMechanism.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/ExponentiallyProfiledElevatorSubsystem.java" %}
