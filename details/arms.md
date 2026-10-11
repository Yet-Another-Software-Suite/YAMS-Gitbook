---
icon: hand-fist
---

# Arms

An `Arm` is a mechanism that swings about a pivot and has to hold itself up against gravity: an intake arm, a shooter hood that tilts, a climber hook.

## How an arm works

<figure><img src="../.gitbook/assets/mech-arm.png" alt="An arm drawn at 40 degrees above a level dashed line marked 0 degrees, with straight up marked 90 degrees. Its length, set by withLength, runs from the pivot to the end of the arm, and gravity pulls down on the end. Level is zero because gravity pulls hardest on a level arm and ArmFeedforward uses kG times the cosine of the angle."><figcaption><p>An arm is measured from level.</p></figcaption></figure>

Following WPILib, an arm is at 0° when it is parallel to the ground and 90° when it points straight up. Gravity pulls hardest on a level arm and not at all on a vertical one, and [`ArmFeedforward`](editor/config/feedforward.md#withfeedforward) uses that angle to work out how much voltage holds the arm still. So the arm's encoder has to read 0 when the arm is level.

If gravity does not matter to your mechanism, because it turns about a vertical axis or is balanced, use a [Pivot](turrets-wrists.md) instead.

<figure><img src="../.gitbook/assets/mech-flow.png" alt="A flow diagram. A command such as arm.run or arm.runTo gives a setpoint to the mechanism, which gives it to the SmartMotorController. The controller runs the profile, PID, feedforward and limits and sends volts to the motor, and reads back the encoder position and velocity. updateTelemetry is called every loop to publish values and draw the mechanism. simIterate is called every loop in simulation to step the physics."><figcaption><p>The same chain runs every mechanism.</p></figcaption></figure>

A YAMS `Arm` is a thin layer over one [`SmartMotorController`](editor/README.md). The `Arm` knows what the mechanism is, turns your commands into setpoints, and runs its simulation. The motor controller does the closed loop control. Keep one mechanism per subsystem, so live tuning and command requirements stay simple.

## Creating an arm

The code is the same for Commands v2 and Commands v3 except for the package you import from and the class that owns the mechanism.

{% tabs %}
{% tab title="Commands v2" %}
```java
import yams.commands2.config.SmartMotorControllerConfig;
import yams.commands2.mechanisms.Arm;

public class ArmSubsystem extends SubsystemBase
{
  private final SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
      .withClosedLoopController(4, 0, 0)
      .withFeedforward(new ArmFeedforward(0, 0.3, 1.2))
      .withTrapezoidalProfile(DegreesPerSecond.of(180), DegreesPerSecondPerSecond.of(90))
      .withSoftLimits(Degrees.of(-30), Degrees.of(100))
      .withStartingPosition(Degrees.of(0))
      .withMomentOfInertia(Feet.of(3), Pounds.of(4))
      .withStatorCurrentLimit(Amps.of(40))
      .withZeroPower(MotorMode.BRAKE)
      .withTelemetry("ArmMotor", TelemetryVerbosity.HIGH);
  private final SmartMotorController motor = new SparkWrapper(spark, DCMotor.getNEO(1), motorConfig);
  private final ArmConfig config = new ArmConfig()
      .withLength(Feet.of(3))
      .withHardLimits(Degrees.of(-100), Degrees.of(200))
      .withTelemetry("Arm", TelemetryVerbosity.HIGH);
  private final Arm arm = new Arm(config, motor);

  @Override
  public void periodic()
  {
    arm.updateTelemetry(); // every loop
  }

  @Override
  public void simulationPeriodic()
  {
    arm.simIterate(); // every loop, in simulation
  }
}
```

`this` is the `Subsystem`. The scheduler calls `periodic()` and `simulationPeriodic()` for you.
{% endtab %}

{% tab title="Commands v3" %}
```java
import yams.commands3.config.SmartMotorControllerConfig;
import yams.commands3.mechanisms.Arm;

public class ArmMechanism implements Mechanism
{
  private final SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
      .withClosedLoopController(4, 0, 0)
      .withFeedforward(new ArmFeedforward(0, 0.3, 1.2))
      .withTrapezoidalProfile(DegreesPerSecond.of(180), DegreesPerSecondPerSecond.of(90))
      .withSoftLimits(Degrees.of(-30), Degrees.of(100))
      .withStartingPosition(Degrees.of(0))
      .withMomentOfInertia(Feet.of(3), Pounds.of(4))
      .withStatorCurrentLimit(Amps.of(40))
      .withZeroPower(MotorMode.BRAKE)
      .withTelemetry("ArmMotor", TelemetryVerbosity.HIGH);
  private final SmartMotorController motor = new SparkWrapper(spark, DCMotor.getNEO(1), motorConfig);
  private final ArmConfig config = new ArmConfig()
      .withLength(Feet.of(3))
      .withHardLimits(Degrees.of(-100), Degrees.of(200))
      .withTelemetry("Arm", TelemetryVerbosity.HIGH);
  private final Arm arm = new Arm(config, motor);

  public void periodic()
  {
    arm.updateTelemetry(); // every loop
  }

  public void simulationPeriodic()
  {
    arm.simIterate(); // every loop, in simulation
  }
}
```

`this` is the Commands v3 `Mechanism`. A `Mechanism` has no periodic hook, so call `periodic()` and `simulationPeriodic()` yourself from `Robot.robotPeriodic()`.
{% endtab %}
{% endtabs %}

The options in `motorConfig` are explained in [SmartMotorControllerConfig](editor/config/README.md), and the options in `config` in [ArmConfig](mechanism-configs/arm-config.md).

## Commands

Every command below exists with the same name in Commands v2 and Commands v3.

<figure><img src="../.gitbook/assets/mech-commands.png" alt="Three timelines starting when the command is scheduled. run(setpoint), also called setAngle or setHeight, runs until something else takes the mechanism and holds the setpoint the whole time. runTo(setpoint, tolerance) ends when the mechanism is within the tolerance, and then the default command takes over. set(dutycycle) and setVoltage(volts) are open loop with no PID, run until interrupted, and the closed loop controller comes back when they end."><figcaption><p>When each kind of command ends.</p></figcaption></figure>

| Command | What it does | Ends |
| --- | --- | --- |
| `setAngle(Angle)`, `run(Angle)` | Go to the angle and hold it. The two are the same. | Never, until interrupted. |
| `runTo(Angle, Angle tolerance)` | Go to the angle. | When the arm is within the tolerance. |
| `set(double)` | Output a duty cycle, with the closed loop off. | Never, until interrupted. |
| `setVoltage(Voltage)` | Output a voltage, with the closed loop off. | Never, until interrupted. |

`setAngle`, `run` and `runTo` also take a `Supplier<Angle>`, and `set` and `setVoltage` a supplier too, for a setpoint that changes while the command runs.

{% tabs %}
{% tab title="Commands v2" %}
```java
// In the subsystem
public Command hold(Angle angle)
{
  return arm.setAngle(angle); // never ends: a good default command
}

public Command moveTo(Angle angle)
{
  return arm.runTo(angle, Degrees.of(2)); // ends when it gets there
}

// In RobotContainer
arm.setDefaultCommand(arm.hold(Degrees.of(0)));
Command score = arm.moveTo(Degrees.of(80)).andThen(intake.eject());
```
{% endtab %}

{% tab title="Commands v3" %}
```java
// In the mechanism
public Command hold(Angle angle)
{
  return run(coroutine -> coroutine.await(arm.setAngle(angle)))
      .withPriority(Command.LOWEST_PRIORITY) // so any other command can take over
      .named("Arm Hold");
}

public Command moveTo(Angle angle)
{
  return arm.runTo(angle, Degrees.of(2)); // ends when it gets there
}

// In Robot
arm.setDefaultCommand(arm.hold(Degrees.of(0)));
Command score = Command.noRequirements(coroutine -> {
  coroutine.await(arm.moveTo(Degrees.of(80)));
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
arm.setMechanismPositionSetpoint(Degrees.of(45));
```

These do not require the subsystem or mechanism, so nothing stops two callers fighting over the setpoint. Prefer the commands unless you need these. [Setpoint Methods vs Command Helpers](setpoint-methods.md) compares the two in detail.

## Asking where it is

<figure><img src="../.gitbook/assets/mech-triggers.png" alt="A number line from minus 30 to 120 degrees with an arm at 72 degrees. near 80 within 10 is true. between 0 and 60 is false. gte 45 is true. lte 45 is false. max and min, at the ends of the travel, are false. Each is a Trigger, and isNear, isBetween, isGte, isLte, isAtMax and isAtMin return the same answers as booleans."><figcaption><p>The same questions as Triggers or as booleans.</p></figcaption></figure>

| As a `Trigger` | As a boolean | True when |
| --- | --- | --- |
| `near(angle, within)` | `isNear(angle, within)` | The arm is within `within` of `angle`. |
| `between(start, end)` | `isBetween(start, end)` | The arm is between the two angles. |
| `gte(angle)`, `lte(angle)` | `isGte(angle)`, `isLte(angle)` | The arm is at or above, or at or below, the angle. |
| `max()`, `min()` | `isAtMax()`, `isAtMin()` | The arm is at its soft limit, or its hard limit if no soft limit is set. |

`getAngle()` returns the arm's current angle.

```java
arm.near(Degrees.of(80), Degrees.of(2)).onTrue(intake.eject());
if (arm.isNear(Degrees.of(0), Degrees.of(2))) { /* stowed */ }
```

## Absolute encoders

An arm that does not know where it is when the robot turns on has to be started in the same place every time. An absolute encoder removes that.

{% hint style="warning" %}
YAMS reads an absolute encoder through the motor controller, so it must be one that motor controller supports: see [Encoders](editor/config/encoders.md#withexternalencoder). For any other encoder, read it yourself and call `SmartMotorController.setEncoderPosition` once at startup.
{% endhint %}

[How do I use absolute encoders on my Arm?](../how-to/how-do-i-use-absolute-encoders-on-my-arm.md) has a complete example for each vendor, including the [zero offset](editor/config/encoders.md#withexternalencoderzerooffset) that makes the encoder read 0 when the arm is level.

## Configuring an arm

Every `ArmConfig` option is on the [ArmConfig](mechanism-configs/arm-config.md) page under this one, with a graphic and an example for each. Options shared by every mechanism are in [Shared options](mechanism-configs/shared-options.md).

An `Arm` needs its length, hard limits and a starting position before it can be created in simulation: see [what simulation needs](mechanism-configs/README.md#what-simulation-needs).

## Code Reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/ArmSubsystem.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands3/simple_robot-2027/src/main/java/first/robot/mechanisms/ArmMechanism.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/ExponentiallyProfiledArmSubsystem.java" %}
