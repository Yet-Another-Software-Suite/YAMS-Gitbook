# How do I shoot with a shooter?

**Goal:** hold a button to spin the shooter up, and have the feeder push the game piece in only once the wheel is at speed.

A shot has three parts:

1. **Spin up:** the shooter `FlyWheel` goes to a speed and holds it.
2. **Wait for speed:** a `Trigger` turns true once the wheel has been near that speed for 0.1 seconds.
3. **Feed:** the feeder `FlyWheel` pushes the game piece into the wheel while the trigger is true.

This page assumes a shooter built as in the [Shooter/FlyWheels tutorial](../tutorials/shooter-flywheels.md), and a feeder built like the intake in [How do I set up an intake roller?](how-do-i-set-up-an-intake-roller.md) The shooter and the feeder are separate subsystems, one YAMS mechanism each, so each can run its own command at the same time.

<figure><img src="../.gitbook/assets/howto-shooter-at-speed.gif" alt="Animation of a flywheel spinning up to 3000 RPM and shooting two balls, with a band of 100 RPM around the setpoint. Each shot drops the speed by several hundred RPM, and it overshoots and rings as it recovers. A lane for the at speed trigger is true inside the band, including for an instant each time the speed passes through it. A lane for the feeder, which uses the trigger with a 0.1 second debounce, is only true once the speed has stayed in the band, so the second ball is not fed until the wheel has settled."><figcaption><p>Each shot slows the wheel and it rings as it recovers. The tolerance band and a debounce keep the feeder off until it has settled.</p></figcaption></figure>

## Step 1: Spin up

The shooter needs gearing that lets it reach your shot speed. This example drives a 4 inch wheel 1:1 from a NEO, which spins freely at about 5600 RPM, and shoots at 3000 RPM. These lines are a fragment of the shooter's `SmartMotorControllerConfig`:

{% code title="ShooterSubsystem.java (fragment)" %}
```java
private final SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
    .withControlMode(ControlMode.CLOSED_LOOP)
    // 1:1, so the wheel can reach 3000 RPM. A 12:1 reduction would top out near 470 RPM.
    .withGearing(new MechanismGearing(1))
    // PLACEHOLDER gains: a starting point for a NEO on a 4 inch wheel, not tuned values.
    .withClosedLoopController(0.1, 0, 0)
    .withSimClosedLoopController(0.1, 0, 0)
    // kS, kV, kA. kV is volts per rotation per second of the wheel.
    .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
    .withSimFeedforward(new SimpleMotorFeedforward(0, 0.12, 0))
    .withMomentOfInertia(Inches.of(2), Pounds.of(1))
    .withZeroPower(MotorMode.COAST)
    .withStatorCurrentLimit(Amps.of(40))
    .withTelemetry("ShooterMotor", TelemetryVerbosity.HIGH);
```
{% endcode %}

**Why kV matters most.** Almost all of a flywheel's voltage comes from `kV`, the voltage per unit of speed. A good first guess is 12 volts divided by the free speed of the wheel: 12 / (5600 RPM ÷ 60) is about 0.12 volts per rotation per second. With `kV` right, the wheel gets close to the target on feedforward alone, and `kP` only has to fix the last bit and recover after each shot. If `kV` is 0, `kP` does all the work and the wheel settles well below the target.

Add these two methods to the shooter class:

{% tabs %}
{% tab title="Commands v2" %}
{% code title="ShooterSubsystem.java (fragment)" %}
```java
/** Spin at a speed and hold it. Runs until interrupted. */
public Command spinUp(AngularVelocity speed) {
  return shooter.run(speed);
}

/** True once the wheel has stayed within 100 RPM of a speed for 0.1 seconds. */
public Trigger ready(AngularVelocity speed) {
  return shooter.near(speed, RPM.of(100)).debounce(0.1);
}
```
{% endcode %}
{% endtab %}

{% tab title="Commands v3" %}
{% code title="ShooterMechanism.java (fragment)" %}
```java
/** Spin at a speed and hold it. Runs until interrupted. */
public Command spinUp(AngularVelocity speed) {
  return shooter.run(speed);
}

/** True once the wheel has stayed within 100 RPM of a speed for 0.1 seconds. */
public Trigger ready(AngularVelocity speed) {
  return shooter.near(speed, RPM.of(100)).debounce(Seconds.of(0.1));
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

## Step 2: Wait for speed

Every shot takes speed out of the wheel, and the speed overshoots and rings as the closed loop brings it back. The 100 RPM tolerance keeps `ready` false while the wheel is too slow or too fast. The speed still passes through the tolerance for an instant on each swing, and the 0.1 second debounce ignores that: `ready` only turns true once the speed has stayed inside the tolerance. Without the debounce, the feeder pulses while the wheel is still ringing.

For a plain boolean outside of a `Trigger`, `FlyWheel` also has `isNear(speed, tolerance)`.

## Step 3: Feed

The feeder is a `FlyWheel` in `ControlMode.OPEN_LOOP`, exactly like the intake in [How do I set up an intake roller?](how-do-i-set-up-an-intake-roller.md), with a `feed()` command that returns `feeder.setVoltage(Volts.of(8))` and a default command that stops it.

Bind both to the same button. The shooter spins up as soon as the button is held, and the feeder only runs while the button is held **and** the shooter is ready.

{% tabs %}
{% tab title="Commands v2" %}
{% code title="RobotContainer.java (fragment)" %}
```java
private static final AngularVelocity SHOT_SPEED = RPM.of(3000);

private final ShooterSubsystem shooter = new ShooterSubsystem();
private final FeederSubsystem feeder = new FeederSubsystem();

private void configureBindings() {
  // Hold the right trigger: spin up.
  m_driverController.rightTrigger().whileTrue(shooter.spinUp(SHOT_SPEED));

  // While the trigger is held and the wheel is ready: feed.
  m_driverController.rightTrigger()
      .and(shooter.ready(SHOT_SPEED))
      .whileTrue(feeder.feed());
}
```
{% endcode %}
{% endtab %}

{% tab title="Commands v3" %}
{% code title="ExampleTeleop.java (fragment)" %}
```java
// In the opmode constructor. SHOT_SPEED is RPM.of(3000).
var shoot = controller.rightTrigger();

// Hold the right trigger: spin up.
shoot.whileTrue(robot.shooter.spinUp(SHOT_SPEED));

// While the trigger is held and the wheel is ready: feed.
shoot.and(robot.shooter.ready(SHOT_SPEED)).whileTrue(robot.feeder.feed());
```
{% endcode %}
{% endtab %}
{% endtabs %}

## Check it

1. Start the simulation and open **NetworkTables > Mechanisms > Shooter** in your dashboard.
2. Hold the right trigger. The shooter's speed should rise to about 3000 RPM.
3. The feeder should start only after the shooter's speed has settled, not while it is still climbing.

## If it did not work

* **The wheel settles below 3000 RPM and the feeder never runs.** `kV` is too low (or 0). Raise it until the wheel gets close on its own, then tune `kP`.
* **The wheel never gets near 3000 RPM, even at full voltage.** The gearing is too slow. Check `withGearing`: the motor's free speed divided by the reduction must be well above the shot speed.
* **The feeder pulses on and off.** The tolerance is too tight, or the debounce is missing. Try 100 to 150 RPM and 0.1 seconds.
* **Something else.** See [Troubleshooting](../tutorials/troubleshooting.md).

## Next

To shoot while the robot is driving, see [How do I shoot on the move?](how-do-i-shoot-on-the-move.md)
