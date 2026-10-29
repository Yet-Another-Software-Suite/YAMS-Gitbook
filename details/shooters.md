---
icon: fan
---

# Shooters

A shooter is a `FlyWheel` in YAMS, and so is anything else that spins: rollers, intakes, feeders and indexers.

## How a flywheel works

<figure><img src="../.gitbook/assets/mech-flywheel-how.png" alt="A graph of flywheel speed against time. It spins up to the speed you asked for, sits inside a tolerance band where it is ready to shoot, drops when a game piece goes through, and recovers back into the band."><figcaption><p>A flywheel holds a speed.</p></figcaption></figure>

Where an arm or an elevator goes to a position, a flywheel goes to a speed. Most of the voltage comes from the `kV` term of [`SimpleMotorFeedforward`](editor/config/feedforward.md#withfeedforward), which is the voltage that speed needs, and a small `kP` corrects what is left and brings the wheel back after a game piece slows it.

A roller that only ever needs a voltage is still a `FlyWheel`: put its motor in [`ControlMode.OPEN_LOOP`](editor/config/motor-basics.md#withcontrolmode) and drive it with `set` or `setVoltage`, as in [Intakes, rollers and feeders](#intakes-rollers-and-feeders) below.

## Creating a flywheel

This is the shape of a shooter class, not a complete file: the imports and the motor controller object (`spark`, a `SparkMax`) are left out. [Step 2 of the Shooter/FlyWheels tutorial](../tutorials/shooter-flywheels/mechanism.md) builds the complete file. The code is the same for Commands v2 and Commands v3 except for the package you import from and the class that owns the mechanism.

{% tabs %}
{% tab title="Commands v2" %}
```java
import yams.commands2.config.SmartMotorControllerConfig;
import yams.commands2.mechanisms.FlyWheel;

public class ShooterSubsystem extends SubsystemBase
{
  private final SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withGearing(new MechanismGearing(1))
      .withClosedLoopController(0.1, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0.01))
      .withMomentOfInertia(Inches.of(4), Pounds.of(1))
      .withStatorCurrentLimit(Amps.of(40))
      .withZeroPower(MotorMode.COAST)
      .withTelemetry("ShooterMotor", TelemetryVerbosity.HIGH);
  private final SmartMotorController motor = new SparkWrapper(spark, DCMotor.getNEO(1), motorConfig);
  private final FlyWheelConfig config = new FlyWheelConfig()
      .withDiameter(Inches.of(4))
      .withTelemetry("Shooter", TelemetryVerbosity.HIGH);
  private final FlyWheel flywheel = new FlyWheel(config, motor); // the YAMS FlyWheel inside the shooter

  @Override
  public void periodic()
  {
    flywheel.updateTelemetry(); // every loop
  }

  @Override
  public void simulationPeriodic()
  {
    flywheel.simIterate(); // every loop, in simulation
  }
}
```

`this` is the `Subsystem`. The scheduler calls `periodic()` and `simulationPeriodic()` for you.
{% endtab %}

{% tab title="Commands v3" %}
```java
import yams.commands3.config.SmartMotorControllerConfig;
import yams.commands3.mechanisms.FlyWheel;

public class ShooterMechanism implements Mechanism
{
  private final SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withGearing(new MechanismGearing(1))
      .withClosedLoopController(0.1, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0.01))
      .withMomentOfInertia(Inches.of(4), Pounds.of(1))
      .withStatorCurrentLimit(Amps.of(40))
      .withZeroPower(MotorMode.COAST)
      .withTelemetry("ShooterMotor", TelemetryVerbosity.HIGH);
  private final SmartMotorController motor = new SparkWrapper(spark, DCMotor.getNEO(1), motorConfig);
  private final FlyWheelConfig config = new FlyWheelConfig()
      .withDiameter(Inches.of(4))
      .withTelemetry("Shooter", TelemetryVerbosity.HIGH);
  private final FlyWheel flywheel = new FlyWheel(config, motor); // the YAMS FlyWheel inside the shooter

  public void periodic()
  {
    flywheel.updateTelemetry(); // every loop
  }

  public void simulationPeriodic()
  {
    flywheel.simIterate(); // every loop, in simulation
  }
}
```

`this` is the Commands v3 `Mechanism`. A `Mechanism` has no periodic hook, so call `periodic()` yourself from `Robot.robotPeriodic()`, and run `simulationPeriodic()` in its own `Notifier`, started from `Robot.simulationInit()`. [Where the tutorials put things](../tutorials/project-setup.md#where-the-tutorials-put-things) shows both.
{% endtab %}
{% endtabs %}

The options in `motorConfig` are explained in [SmartMotorControllerConfig](editor/config/), and the options in `config` in [FlyWheelConfig](mechanism-configs/flywheel-config.md).

In the rest of your code, `shooter` is the `ShooterSubsystem` (or `ShooterMechanism`), and `flywheel` is the YAMS `FlyWheel` inside it.

{% hint style="info" %}
The gains (`kP` of 0.1, `kS` of 0.1, `kV` of 0.12) are placeholders. In simulation the wheel should spin up close to the speed you ask for, perhaps settling a little under or over it. On your robot they will be wrong: the [Shooter/FlyWheels tutorial](../tutorials/shooter-flywheels/tune.md) finds the real ones.
{% endhint %}

## Controlling a flywheel

A flywheel takes speed targets: `flywheel.run(RPM.of(3000))` spins at 3000 RPM until interrupted, `flywheel.runTo(RPM.of(3000), RPM.of(100))` ends once it is at speed, and `flywheel.getSpeed()` reads how fast it is going. [Controlling a mechanism](controlling-a-mechanism.md) lists every command, setpoint function and check, and [When mechanism commands end](run-vs-runto.md) explains how they work with a default command.

A shooter subsystem usually wraps them in a few command factories:

{% tabs %}
{% tab title="Commands v2" %}
```java
// In ShooterSubsystem
public Command spin(AngularVelocity speed)
{
  return flywheel.run(speed); // holds the speed, never ends
}

public Command spinUp(AngularVelocity speed)
{
  return flywheel.runTo(speed, RPM.of(100)); // ends once it is at speed
}

public Command stop()
{
  return flywheel.set(0); // closed loop off, output 0: the wheel coasts down
}

public Trigger atSpeed(AngularVelocity speed)
{
  return flywheel.near(speed, RPM.of(100));
}

// In RobotContainer
shooter.setDefaultCommand(shooter.stop());
```
{% endtab %}

{% tab title="Commands v3" %}
```java
// In ShooterMechanism
public Command spin(AngularVelocity speed)
{
  return flywheel.run(speed); // holds the speed, never ends
}

public Command spinUp(AngularVelocity speed)
{
  return flywheel.runTo(speed, RPM.of(100)); // ends once it is at speed
}

public Command stop()
{
  return run(coroutine -> coroutine.await(flywheel.set(0))) // closed loop off, output 0
      .withPriority(Command.LOWEST_PRIORITY)                // so any other command can take over
      .named("Shooter Stop");
}

public Trigger atSpeed(AngularVelocity speed)
{
  return flywheel.near(speed, RPM.of(100));
}

// In Robot
shooter.setDefaultCommand(shooter.stop());
```
{% endtab %}
{% endtabs %}

Make `set(0)` the default command, not `run(RPM.of(0))`. A speed of zero is a closed loop target, so the motor would brake the wheel to a stop against its momentum, drawing current to do it. `set(0)` turns the closed loop off and lets the wheel coast down, with `MotorMode.COAST`.

## Shooting

A shot is two flywheels working together: the shooter spins up, and a feeder pushes the game piece in once the shooter is at speed. Gate the feeder on the shooter's `atSpeed` trigger:

```java
// In RobotContainer (Commands v2) or Robot (Commands v3).
// feeder is a roller subsystem with a feed() command, as in the next section.
controller.rightTrigger().whileTrue(shooter.spin(RPM.of(3000)));
controller.rightTrigger().and(shooter.atSpeed(RPM.of(3000))).whileTrue(feeder.feed());
```

While the trigger is held the shooter holds 3000 RPM, and the feeder only runs while the shooter is within 100 RPM of it. On release, both default commands take over. [How do I shoot with a shooter?](../how-to/how-to-shoot-with-a-shooter.md) walks through it, and [Double FlyWheel](../tutorials/double-flywheel.md) covers two wheels spinning at different speeds.

## Intakes, rollers and feeders

A roller that only needs to push a game piece in or out does not need a speed. Make it a `FlyWheel` in its own subsystem, with its motor in open loop, and give it a voltage:

```java
// Fragment of FeederSubsystem (Commands v2). spark is the SparkMax for the roller.
private final SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
    .withControlMode(ControlMode.OPEN_LOOP)            // no PID or feedforward
    .withGearing(new MechanismGearing(3))
    .withMomentOfInertia(Inches.of(1), Pounds.of(0.5)) // for the simulation
    .withStatorCurrentLimit(Amps.of(40))
    .withZeroPower(MotorMode.COAST)
    .withTelemetry("FeederMotor", TelemetryVerbosity.HIGH);
private final SmartMotorController motor = new SparkWrapper(spark, DCMotor.getNEO(1), motorConfig);
private final FlyWheel roller = new FlyWheel(
    new FlyWheelConfig().withTelemetry("Feeder", TelemetryVerbosity.HIGH), motor);

public Command feed()
{
  return roller.setVoltage(Volts.of(8));  // push the game piece toward the shooter, until interrupted
}

public Command eject()
{
  return roller.set(-0.8);                // 80% backwards, until interrupted
}

public Command stop()
{
  return roller.set(0);                   // the default command
}
```

`set` takes a duty cycle from -1 to 1, and `setVoltage` a voltage. A voltage gives the same push as the battery drains, so prefer it. An intake is the same class with an `intake()` command. In Commands v3, write `stop()` as a lowest priority default command, as for the shooter above. [How do I set up an intake roller?](../how-to/how-do-i-set-up-an-intake-roller.md) has complete files for both.

## When a flywheel will not reach its speed## Configuring and tuning a flywheel, step by step

Do the configuration steps first, in simulation, then tune the real flywheel with [live tuning](live-tuning/). Tuning something that is configured wrong wastes the time.

{% hint style="danger" %}
Like all tuning, this can be dangerous. Keep clear of the mechanism, start with low current limits and small moves, and practice in simulation first.
{% endhint %}

### Configure it

{% stepper %}
{% step %}
#### Set the gearing

Give the reduction from the motor to the wheel with [`withGearing`](editor/config/gearing-and-units.md#withgearing). An overdrive, where the wheel spins faster than the motor, is a number below 1.
{% endstep %}

{% step %}
#### Choose the positive direction

Positive should be the direction that shoots or intakes. Flip [`withMotorInverted`](editor/config/motor-basics.md#withmotorinverted) if it is not.
{% endstep %}

{% step %}
#### Let it coast

Use `MotorMode.COAST` so the wheel is not braked to a stop every time the command ends, and set a [stator current limit](editor/config/limits-and-protection.md#current-limits).
{% endstep %}

{% step %}
#### Set the diameter

Set [`withDiameter`](mechanism-configs/flywheel-config.md#withdiameter) if you want to ask for surface speeds.
{% endstep %}

{% step %}
#### Describe it to the simulator

Set [`withMomentOfInertia`](editor/config/simulation.md#withmomentofinertia) on the motor config.
{% endstep %}
{% endstepper %}

### What is different when tuning a flywheel

The [Shooter/FlyWheels tutorial](../tutorials/shooter-flywheels/tune.md) walks through tuning a flywheel, and [Tuning order](live-tuning/#tuning-order) lists the order with the keys to watch. What sets a flywheel apart:

* **kV does most of the work.** It is the voltage the speed needs. Find kS, then raise kV until `mechanism/velocity` settles on the setpoint with no help from kP, and check it at a low and a high speed.
* **kP only needs to be small.** Its job is to bring the wheel back after a game piece slows it. Too much makes the speed ripple.
* **There is no kG and no position.** Nothing to hold up, and no soft limits.
* **Choose the tolerance by shooting.** Shoot at the edges of your `atSpeed` tolerance, and use the widest one where the shots still land, so the shooter is ready sooner.
* **A brownout is not a tuning problem.** If spinning up browns out the robot, add a [velocity profile](editor/config/motion-profiles.md#velocity-profiles-for-flywheels) or a [current limit](editor/config/limits-and-protection.md#current-limits), and test it against the [battery simulation](../understanding/battery-simulation.md).

## FlyWheelConfig options

Every `FlyWheelConfig` option is on the [FlyWheelConfig](mechanism-configs/flywheel-config.md) page under this one, with a graphic and an example for each. Options shared by every mechanism are in [Shared options](mechanism-configs/shared-options.md).

A `FlyWheel` needs nothing on its config to be simulated. Its [moment of inertia](editor/config/simulation.md#withmomentofinertia) is on the motor config.

## Code reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/ShooterSubsystem.java" %}

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands3/simple_robot-2027/src/main/java/first/robot/mechanisms/ShooterMechanism.java" %}
