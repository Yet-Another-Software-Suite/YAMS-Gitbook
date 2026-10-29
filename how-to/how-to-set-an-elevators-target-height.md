---
icon: elevator
---

# How do I set an elevator's target height?

**Goal:** send your `Elevator` to a height from a button, an autonomous routine or your own code.

This page assumes you already have an `Elevator` in a subsystem. If you do not, build one with the [Elevator tutorial](../tutorials/elevator.md) first; it ends with complete files you can copy.

An elevator's target is a `Distance` (`org.wpilib.units.measure.Distance`), for example `Meters.of(1)` or `Inches.of(30)`. YAMS turns it into motor rotations using the drum radius or circumference you set in the `SmartMotorControllerConfig` (see [How do I command my elevator in meters?](how-to-find-your-mechanism-circumference.md)). There are three ways to set it:

| Use | When | Ends? |
| --- | --- | --- |
| `elevator.run(height)` | Hold a height for as long as a button is held, or as a default command. | No. It runs until something interrupts it. |
| `elevator.runTo(height, tolerance)` | A step in a sequence or autonomous routine: go there, then move on. | Yes, once the carriage has stayed within `tolerance` of the height for 0.1 seconds. |
| `elevator.setMeasurementPositionSetpoint(height)` | Your own code, inside a command you already wrote. | It is a method, not a command. It sets the target once. |

All three start the closed loop controller again if an open loop command (`set` or `setVoltage`) stopped it. For more on the difference, see [When mechanism commands end](../details/run-vs-runto.md).

## Add the commands to your subsystem

These methods go in the subsystem (Commands v2) or mechanism (Commands v3) that owns the `elevator` field. They are fragments: the rest of the class is in the [Elevator tutorial](../tutorials/elevator/commands.md).

{% tabs %}
{% tab title="Commands v2" %}
{% code title="ExampleSubsystem.java (fragment)" %}
```java
/** Go to a height and hold it. Never ends on its own. */
public Command run(Distance height) {
  return elevator.run(height);
}

/** Go to a height, then end once the carriage is within tolerance. */
public Command runTo(Distance height, Distance tolerance) {
  return elevator.runTo(height, tolerance);
}

/** Set the target from your own code. */
public void setHeightSetpoint(Distance height) {
  elevator.setMeasurementPositionSetpoint(height);
}
```
{% endcode %}
{% endtab %}

{% tab title="Commands v3" %}
{% code title="ExampleMechanism.java (fragment)" %}
```java
/** Go to a height and hold it. Never ends on its own. */
public Command setHeight(Distance height) {
  return elevator.run(height);
}

/** Go to a height, then end once the carriage is within tolerance. */
public Command moveTo(Distance height, Distance tolerance) {
  return elevator.runTo(height, tolerance);
}

/** Hold a height at the lowest priority, so any other command can take over. For the default command. */
public Command hold(Distance height) {
  return run(coroutine -> coroutine.await(elevator.run(height)))
      .withPriority(Command.LOWEST_PRIORITY)
      .named("Elevator Hold");
}

/** Set the target from your own code. */
public void setHeightSetpoint(Distance height) {
  elevator.setMeasurementPositionSetpoint(height);
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

## Bind them to buttons

Every target must be inside your limits. This example assumes the tutorial's elevator, which travels from 0 to 3 meters.

{% tabs %}
{% tab title="Commands v2" %}
{% code title="RobotContainer.java (fragment)" %}
```java
// Hold the carriage at the bottom when nothing else is using it.
m_exampleSubsystem.setDefaultCommand(m_exampleSubsystem.run(Meters.of(0)));

// Hold 0.5 m or 1 m while A or B is held.
m_driverController.a().whileTrue(m_exampleSubsystem.run(Meters.of(0.5)));
m_driverController.b().whileTrue(m_exampleSubsystem.run(Meters.of(1)));

// Go to 1 m, then (once within 2 cm) do the next thing.
m_driverController.x().onTrue(
    m_exampleSubsystem.runTo(Meters.of(1), Centimeters.of(2))
        .andThen(Commands.print("Elevator is at 1 m")));
```
{% endcode %}
{% endtab %}

{% tab title="Commands v3" %}
{% code title="ExampleTeleop.java (fragment)" %}
```java
// In Robot(): hold the carriage at the bottom when nothing else is using it.
exampleMechanism.setDefaultCommand(exampleMechanism.hold(Meters.of(0)));

// In the opmode constructor: hold 0.5 m or 1 m while A or B is held.
controller.a().whileTrue(robot.exampleMechanism.setHeight(Meters.of(0.5)));
controller.b().whileTrue(robot.exampleMechanism.setHeight(Meters.of(1)));
```
{% endcode %}
{% endtab %}
{% endtabs %}

In Commands v3 a `Mechanism` already has a `run(...)` method that takes a coroutine, so the wrappers are called `setHeight`, `moveTo` and `hold`, as in the tutorial.

If the target comes from a `Supplier<Distance>`, `run(supplier)` and `runTo(supplier, tolerance)` follow the supplier every loop. `runTo` ends once the carriage has been within the tolerance of the supplier's current value for 0.1 seconds.

## Check it

1. Start the simulation (or enable the robot) and open **NetworkTables > Mechanisms > Elevator > mechanism** in your dashboard. `Elevator` is the name you gave `ElevatorConfig.withTelemetry`.
2. Hold B. The drawn carriage should rise to 1 meter and stay there.
3. Release B. The default command takes over and the carriage returns to the bottom.

## If it did not work

* **The carriage stops short of the target.** The target is outside your soft limits. YAMS clamps the setpoint to the nearest limit and prints a warning in the Driver Station. A `runTo` to a clamped target never ends.
* **The carriage sags below the target, or slides down when you let go.** `kG` is still 0. The tutorial's gains are placeholders. Find `kG` first (the voltage that holds the carriage still), then `kP`, with [Step 5: Tune it](../tutorials/elevator/tune.md).
* **The carriage goes to the wrong height, off by the same factor every time.** The drum radius or circumference is wrong. See [How do I command my elevator in meters?](how-to-find-your-mechanism-circumference.md)
* **Something else.** See [Troubleshooting](../tutorials/troubleshooting.md).

## Next

Tune the elevator so it reaches the target quickly and holds it: [Step 5: Tune it](../tutorials/elevator/tune.md).
