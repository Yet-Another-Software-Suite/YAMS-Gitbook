---
icon: rotate
---

# Turrets/Wrists

Turrets and wrists are both a `Pivot` in YAMS: a mechanism that turns to an angle. A `Pivot` is different from an `Arm` because it does not account for gravity.

## How a pivot works

<figure><img src="../.gitbook/assets/mech-pivot-vs-arm.png" alt="Two panels. An Arm accounts for gravity: it swings up and down, needs kG to hold and uses ArmFeedforward. A Pivot does not account for gravity: it is for a vertical axis or a balanced wrist, has no kG and uses SimpleMotorFeedforward."><figcaption><p>An Arm accounts for gravity. A Pivot does not.</p></figcaption></figure>

A pivot is controlled exactly like an arm, with one difference: a `Pivot` does not account for gravity. It has no `kG`, its simulation applies no gravity, and it uses [`SimpleMotorFeedforward`](editor/config/feedforward.md#withfeedforward) where an arm uses `ArmFeedforward`.

That is the right model for a turret, which turns about a vertical axis, and for a wrist that is balanced about its shaft: neither needs voltage just to stay where it is. It also makes a pivot simpler to tune.

If your mechanism needs voltage to hold itself up, it needs gravity accounted for: use an [Arm](arms.md).

<figure><img src="../.gitbook/assets/mech-flow.png" alt="A flow diagram. A command such as arm.run or arm.runTo gives a setpoint to the mechanism, which gives it to the SmartMotorController. The controller runs the profile, PID, feedforward and limits and sends volts to the motor, and reads back the encoder position and velocity. updateTelemetry is called every loop to publish values and draw the mechanism. simIterate is called every loop in simulation to step the physics."><figcaption><p>The same chain runs every mechanism.</p></figcaption></figure>

A YAMS `Pivot` is a thin layer over one [`SmartMotorController`](editor/). The `Pivot` knows what the mechanism is, turns your commands into setpoints, and runs its simulation. The motor controller does the closed loop control. Keep one mechanism per subsystem, so live tuning and command requirements stay simple.

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

The options in `motorConfig` are explained in [SmartMotorControllerConfig](editor/config/), and the options in `config` in [PivotConfig](mechanism-configs/pivot-config.md).

## Commands

Every command below exists with the same name in Commands v2 and Commands v3.

<figure><img src="../.gitbook/assets/mech-commands.png" alt="Three timelines starting when the command is scheduled. run(setpoint), also called setAngle or setHeight, runs until something else takes the mechanism and holds the setpoint the whole time. runTo(setpoint, tolerance) ends when the mechanism is within the tolerance, and then the default command takes over. set(dutycycle) and setVoltage(volts) are open loop with no PID, run until interrupted, and the closed loop controller comes back when they end."><figcaption><p>When each kind of command ends.</p></figcaption></figure>

| Command                         | What it does                                       | Ends                                    |
| ------------------------------- | -------------------------------------------------- | --------------------------------------- |
| `setAngle(Angle)`, `run(Angle)` | Go to the angle and hold it. The two are the same. | Never, until interrupted.               |
| `runTo(Angle, Angle tolerance)` | Go to the angle.                                   | When the pivot is within the tolerance. |
| `set(double)`                   | Output a duty cycle, with the closed loop off.     | Never, until interrupted.               |
| `setVoltage(Voltage)`           | Output a voltage, with the closed loop off.        | Never, until interrupted.               |

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

| Function                                   | What it does                                                                                                                                         |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `setMechanismPositionSetpoint(Angle)`      | Go to this angle with closed loop control.                                                                                                           |
| `setMeasurementPositionSetpoint(Distance)` | Go to this distance. Needs a [circumference](editor/config/gearing-and-units.md#withmechanismcircumference).                                         |
| `setVoltageSetpoint(Voltage)`              | Output this voltage. Stops the closed loop.                                                                                                          |
| `setDutyCycleSetpoint(double)`             | Output this duty cycle, from -1 to 1. Stops the closed loop.                                                                                         |
| `getMechanismSetpoint()`                   | The current position setpoint as an angle, if there is one.                                                                                          |
| `getMeasurementSetpoint()`                 | The same setpoint as a distance. Empty if there is no setpoint or no [circumference](editor/config/gearing-and-units.md#withmechanismcircumference). |

```java
pivot.setMechanismPositionSetpoint(Degrees.of(30));
```

These do not require the subsystem or mechanism, so nothing stops two callers fighting over the setpoint. Prefer the commands unless you need these. [Setpoint Methods vs Command Helpers](setpoint-methods.md) compares the two in detail.

## Asking where it is

<figure><img src="../.gitbook/assets/mech-triggers.png" alt="A number line from minus 30 to 120 degrees with a mechanism at 72 degrees. near 80 within 10 is true. between 0 and 60 is false. gte 45 is true. lte 45 is false. max and min, at the ends of the travel, are false. Each is a Trigger, and the is functions such as isNear give the same answers as booleans."><figcaption><p>The same questions as Triggers or as booleans.</p></figcaption></figure>

| As a `Trigger`             | As a boolean                   | True when                                                                  |
| -------------------------- | ------------------------------ | -------------------------------------------------------------------------- |
| `near(angle, within)`      | `isNear(angle, within)`        | The pivot is within `within` of `angle`.                                   |
| `between(start, end)`      | `isBetween(start, end)`        | The pivot is between the two angles.                                       |
| `gte(angle)`, `lte(angle)` | `isGte(angle)`, `isLte(angle)` | The pivot is at or above, or at or below, the angle.                       |
| `max()`, `min()`           | `isAtMax()`, `isAtMin()`       | The pivot is at its soft limit, or its hard limit if no soft limit is set. |

`getAngle()` returns the pivot's current angle.

```java
turret.near(Degrees.of(90), Degrees.of(1)).whileTrue(shooter.fire());
```

## Turret or wrist?

<figure><img src="../.gitbook/assets/mech-pivot-kinds.png" alt="Two panels. A turret, seen from above, turns on top of the robot with its angle measured from the robot&#x27;s front; it either wraps or is limited by its wiring, tracks a target with a Supplier of Angle, and can use EasyCRT for its true angle. A wrist turns on the end of an arm with its angle measured from the arm it is mounted on; it always has soft limits, moves together with the arm in a sequence, and should be kept balanced."><figcaption><p>A turret and a wrist are both a Pivot.</p></figcaption></figure>

{% tabs %}
{% tab title="Turret" %}
A turret turns on top of the robot. Its angle is measured from the robot's front, so 0° usually means pointing forward.

**Configure it** for how far it can turn:

| The turret                                            | What to set                                                                                                                                     |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Can spin forever (a slip ring, or nothing to wind up) | [`withContinuousWrapping`](editor/config/closed-loop-control.md#withcontinuouswrapping) and no soft limits, so it takes the short way round.    |
| Is limited by its wiring                              | [`withSoftLimits`](editor/config/limits-and-protection.md#withsoftlimits), with hard limits a little wider. It may have to unwind the long way. |

```java
// A turret limited by its wiring to one and a bit turns
SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(10, 4)))
    .withClosedLoopController(20, 0, 0.5)
    .withTrapezoidalProfile(DegreesPerSecond.of(360), DegreesPerSecondPerSecond.of(720))
    .withSoftLimits(Degrees.of(-200), Degrees.of(200))
    .withStartingPosition(Degrees.of(0))
    .withMomentOfInertia(Meters.of(0.25), Pounds.of(8));
PivotConfig config = new PivotConfig()
    .withHardLimits(Degrees.of(-220), Degrees.of(220))
    .withTelemetry("Turret", TelemetryVerbosity.HIGH);
```

**Use it** to track something by giving `setAngle` a supplier. To aim at a point on the field, the turret's angle is the direction to the target minus the robot's heading:

```java
public Command aimAt(Supplier<Angle> fieldAngleToTarget, Supplier<Angle> robotHeading)
{
  return pivot.setAngle(() -> fieldAngleToTarget.get().minus(robotHeading.get()));
}
```

A turret geared so its encoders turn more than once over its travel cannot tell where it is from one encoder. [EasyCRT](easycrt.md) works out its true angle from two.
{% endtab %}

{% tab title="Wrist" %}
A wrist turns on the end of another mechanism, usually an arm. Its angle is measured from the arm it is mounted on, not from the ground.

**Configure it** with soft limits always, because a wrist has parts to hit at both ends of its travel, and a small moment of inertia.

```java
SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(5, 5)))
    .withClosedLoopController(12, 0, 0.2)
    .withTrapezoidalProfile(DegreesPerSecond.of(360), DegreesPerSecondPerSecond.of(720))
    .withSoftLimits(Degrees.of(-90), Degrees.of(90))
    .withStartingPosition(Degrees.of(0))
    .withMomentOfInertia(Inches.of(6), Pounds.of(2));
PivotConfig config = new PivotConfig()
    .withHardLimits(Degrees.of(-100), Degrees.of(100))
    .withTelemetry("Wrist", TelemetryVerbosity.HIGH);
```

**Use it** together with the arm it rides on. Keep the wrist in its own subsystem, and move the two from a command that owns neither, so each keeps its own default command.

A `Pivot` does not account for gravity, so it suits a wrist that is balanced about its shaft. A heavy, unbalanced wrist is pulled on by gravity by an amount that depends on the arm's angle as well as its own, which neither `Pivot` nor `Arm` models. Balance the wrist if you can. If you cannot, expect to need more `kP`, and check it holds at every arm angle.
{% endtab %}
{% endtabs %}

## Moment of inertia

How heavy the pivot is to turn is set on the motor config with [`withMomentOfInertia`](editor/config/simulation.md#withmomentofinertia). It only matters in simulation, where it also decides the current the pivot draws from the [battery simulation](../understanding/battery-simulation.md).

## Configuring and tuning a pivot, step by step

Do the configuration steps first, in simulation, then the tuning steps on the real mechanism with [live tuning](live-tuning/). Tuning something that is configured wrong wastes the time.

{% hint style="danger" %}
Like all tuning, this can be dangerous. Keep clear of the mechanism, start with low current limits and small moves, and practice in simulation first.
{% endhint %}

### Configure it

{% stepper %}
{% step %}
#### Set the gearing

Give the reduction from the motor to the pivot with [`withGearing`](editor/config/gearing-and-units.md#withgearing).
{% endstep %}

{% step %}
#### Choose the positive direction

For a turret, counter-clockwise seen from above is positive, to match the robot's heading. Apply a small positive voltage and flip [`withMotorInverted`](editor/config/motor-basics.md#withmotorinverted) if it turns the other way.
{% endstep %}

{% step %}
#### Choose where zero is

A turret's zero is usually pointing at the robot's front. A wrist's zero is usually in line with its arm. Set an absolute encoder's [zero offset](editor/config/encoders.md#withexternalencoderzerooffset), or a [starting position](editor/config/encoders.md#withstartingposition).
{% endstep %}

{% step %}
#### Wrap it or limit it

A turret that can spin forever gets [`withContinuousWrapping`](editor/config/closed-loop-control.md#withcontinuouswrapping) and no soft limits. Everything else gets [`withSoftLimits`](editor/config/limits-and-protection.md#withsoftlimits), inside [`withHardLimits`](mechanism-configs/shared-options.md#withhardlimits).
{% endstep %}

{% step %}
#### Describe it to the simulator

Set [`withMomentOfInertia`](editor/config/simulation.md#withmomentofinertia) on the motor config.
{% endstep %}
{% endstepper %}

### Tune it

A `Pivot` does not account for gravity, so there is no kG to find.

{% stepper %}
{% step %}
#### Find kS

Set kP, kI and kD to 0. Raise `closedloop/feedforward/kS` until the pivot is just about to move, in each direction, and use the smaller value.
{% endstep %}

{% step %}
#### Raise kP, then add kD

Raise kP until it reaches the setpoint quickly, and stop when it starts to shake. Add a little kD if it overshoots. Without a profile a new setpoint is a sudden jump, so tune with small moves, a few degrees or centimeters at a time.
{% endstep %}

{% step %}
#### Add the motion profile last

Once the pivot reaches small setpoints cleanly, set a maximum velocity and acceleration it can really reach, and raise them a step at a time.
{% endstep %}

{% step %}
#### Find kV with the profile

kV only acts while the setpoint is moving, so it is tuned with the profile. Run a long move and raise kV until `mechanism/velocity` matches the setpoint velocity in the middle of it.
{% endstep %}
{% endstepper %}

### What changes for each kind

{% tabs %}
{% tab title="Turret" %}
* **Check both directions.** Cables and hoses drag more one way than the other, and more near the end of the travel. Tune kS and kP where the drag is worst.
* **Tune it while tracking.** A turret that aims at a target has a setpoint that never stops moving. Drive and spin the robot while it tracks, and raise kV and kP until it stays on target.
* **Check the seam.** With continuous wrapping, command a move across the wrapping point and check it takes the short way.
* **Check the unwind.** With soft limits, command an angle just past a limit and check the turret goes the long way round to reach it, or stops at the limit.
{% endtab %}

{% tab title="Wrist" %}
* **Tune it at several arm angles.** Because a `Pivot` does not account for gravity, an unbalanced wrist is pushed around by it as the arm moves. Tune with the arm level, where that is worst, then check with the arm up and down.
* **Raise kP to hold against gravity** if you cannot balance the wrist. Expect a small steady error that changes with the arm's angle.
* **Tune it holding a game piece**, which changes both its weight and its balance.
* **Check the limits with the arm in every position.** A wrist that clears the robot with the arm up may hit it with the arm down.
{% endtab %}
{% endtabs %}

## PivotConfig options

Every `PivotConfig` option is on the [PivotConfig](mechanism-configs/pivot-config.md) page under this one, with a graphic and an example for each. Options shared by every mechanism are in [Shared options](mechanism-configs/shared-options.md).

A `Pivot` needs its hard limits and a starting position before it can be created in simulation: see [what simulation needs](mechanism-configs/#what-simulation-needs).

## Code reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/TurretSubsystem.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands3/simple_robot-2027/src/main/java/first/robot/mechanisms/TurretMechanism.java" %}
