# How do I zero a mechanism without limit switches?

**Goal:** make your arm or elevator's encoder read the right position, when there is no limit switch and no absolute encoder.

A motor's built-in encoder is **relative**: it only knows how far the mechanism has moved since it was last set. YAMS sets it once, when the robot code starts, to the starting position in your config. Everything after that is counted from there. If the mechanism was somewhere else at power on, or a belt skipped, every position is off by the same amount.

There are three ways to set it without a switch, from simplest to most automatic. If you can add a sensor, an [absolute encoder](how-do-i-use-absolute-encoders-on-my-arm.md) or a [limit switch](how-do-i-home-a-mechanism-with-limit-switches.md) is more reliable than any of them.

## 1. Always start in the same place

Pick a position the mechanism can always rest in when the robot is turned on, usually against its lower hard stop, and tell YAMS that position:

{% code title="ElevatorSubsystem.java (fragment)" %}
```java
// The carriage rests on its lower hard stop when the robot turns on.
.withStartingPosition(Meters.of(0))
```
{% endcode %}

For an arm, use an `Angle`, such as `.withStartingPosition(Degrees.of(-10))` for an arm resting on a stop 10 degrees below horizontal.

This only works if the whole team follows the rule: **the mechanism must be at that position whenever the robot code starts**, including after a code deploy or a brownout. Put it on the pre-match checklist.

## 2. A "zero here" button

When the encoder is wrong, a human can push the mechanism to its hard stop and press a button that says "you are here". `SmartMotorController.setEncoderPosition` sets the encoder to the position you give it. `motor` is the `SmartMotorController` your mechanism was built with; these methods go in the same class.

{% tabs %}
{% tab title="Commands v2" %}
{% code title="ElevatorSubsystem.java (fragment)" %}
```java
/** Tell the encoder the carriage is on its lower hard stop. Works while disabled. */
public Command zeroHere() {
  return Commands.runOnce(() -> motor.setEncoderPosition(Meters.of(0)))
      .ignoringDisable(true)
      .withName("Elevator Zero Here");
}
```
{% endcode %}

Bind it to a button the drive team will not press by accident, for example `m_operatorController.back().onTrue(elevator.zeroHere());`.
{% endtab %}

{% tab title="Commands v3" %}
{% code title="ElevatorMechanism.java (fragment)" %}
```java
/** Tell the encoder the carriage is on its lower hard stop. */
public Command zeroHere() {
  return Command.noRequirements(coroutine -> motor.setEncoderPosition(Meters.of(0)))
      .named("Elevator Zero Here");
}
```
{% endcode %}

Bind it in your opmode to a button the drive team will not press by accident, for example `controller.back().onTrue(robot.elevator.zeroHere());`.
{% endtab %}
{% endtabs %}

Disable the robot, push the carriage down onto the stop by hand, and press the button.

## 3. Drive into the hard stop and watch the current

The robot can find the hard stop itself: drive the mechanism slowly toward it, and when it stops moving and the current rises, it has arrived. This is current, or stall, homing. It is the same idea as [detecting a game piece from motor current](how-do-i-sense-activity-based-off-of-the-current.md).

The outline:

1. Apply a small voltage toward the hard stop, enough to move the mechanism slowly (start around 1 to 2 volts).
2. Each loop, check whether the stator current is over a threshold, well below your current limit, and the mechanism's speed is close to zero.
3. Once both have been true for about 0.25 seconds (use a `Debouncer`), set the encoder to the hard stop position and hold it there with a position setpoint.

{% tabs %}
{% tab title="Commands v2" %}
{% code title="ElevatorSubsystem.java (fragment)" %}
```java
// Measure these on your robot: graph current/stator while the carriage pushes on the stop.
private static final Current STALL_CURRENT = Amps.of(20);
private final Debouncer stallDebouncer = new Debouncer(0.25);

private boolean isStalled() {
  return stallDebouncer.calculate(
      motor.getStatorCurrent().gte(STALL_CURRENT)
          && motor.getMeasurementVelocity().isNear(MetersPerSecond.of(0), MetersPerSecond.of(0.02)));
}

/** Drive down slowly until the carriage stalls on its hard stop, then zero the encoder. */
public Command stallHome() {
  return elevator.setVoltage(Volts.of(-1.5))
      .until(this::isStalled)
      .andThen(Commands.runOnce(() -> {
        motor.setEncoderPosition(Meters.of(0));
        elevator.setMeasurementPositionSetpoint(Meters.of(0));
      }))
      .withName("Elevator Stall Home");
}
```
{% endcode %}
{% endtab %}

{% tab title="Commands v3" %}
{% code title="ElevatorMechanism.java (fragment)" %}
```java
// Measure these on your robot: graph current/stator while the carriage pushes on the stop.
private static final Current STALL_CURRENT = Amps.of(20);
private final Debouncer stallDebouncer = new Debouncer(0.25);

private boolean isStalled() {
  return stallDebouncer.calculate(
      motor.getStatorCurrent().gte(STALL_CURRENT)
          && motor.getMeasurementVelocity().isNear(MetersPerSecond.of(0), MetersPerSecond.of(0.02)));
}

/** Drive down slowly until the carriage stalls on its hard stop, then zero the encoder. */
public Command stallHome() {
  return run(coroutine -> {
    while (true) {
      if (isStalled()) {
        break;
      }
      elevator.setVoltageSetpoint(Volts.of(-1.5));
      coroutine.yield();
    }
    motor.setEncoderPosition(Meters.of(0));
    // Hold the bottom. This also starts the closed loop again.
    elevator.setMeasurementPositionSetpoint(Meters.of(0));
  }).named("Elevator Stall Home");
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

The speed check matters: the current also rises for a moment when the motor starts, but the mechanism is moving then. Soft limits can stop the mechanism before it reaches the hard stop; if they do, homing never finishes (see the soft limit pitfall in [How do I home a mechanism with limit switches?](how-do-i-home-a-mechanism-with-limit-switches.md#soft-limits-keep-the-mechanism-off-the-switch)).

## Check it

1. Move the mechanism away from the hard stop, then zero it with the method you chose.
2. Under **NetworkTables > Mechanisms**, the motor's `measurement/position` (or the arm's angle) should read the hard stop position, 0 m here.
3. Send it to a known height and measure with a tape measure: it should match.

## If it did not work

* **The position is off by the same amount every time.** The mechanism was not at the starting position when the code started, or the zero position in your code is not where the hard stop really is.
* **Stall homing ends too early.** The threshold is too low, or the start up spike is longer than the debounce. Raise the threshold or the debounce time.
* **Stall homing never ends.** The threshold is at or above the current limit, or a soft limit stops the mechanism short of the hard stop.
* **Something else.** See [Troubleshooting](../tutorials/troubleshooting.md).

## Next

Before the first enable on the robot, work through [Step 6: Run it on the robot](../tutorials/arm/robot.md).
