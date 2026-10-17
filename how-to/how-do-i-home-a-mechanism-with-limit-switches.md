# How do I home a mechanism with limit switches?

A relative encoder only knows how far the mechanism has moved since the robot turned on, and a slipped belt or a skipped gear throws it off. A limit switch at a hard stop tells the mechanism where it is: when the mechanism presses it, the motor's encoder is set to that hard stop. This is homing. It works for arms, elevators, pivots and turrets.

[YAMG](https://yamg.yassrobotics.com) generates this code for you: turn on **Homing** in its form.

## Wrap the switch in a `Sensor`

Plug the switch into a SystemCore Smart IO port and read it with a `DigitalInput`. Wrap it in a [`Sensor`](../details/sensors.md), so that the simulation can press it at the hard stop, as the mechanism presses the real one.

```java
import static org.wpilib.units.Units.Degrees;

import org.wpilib.hardware.discrete.DigitalInput;
import yams.core.mechanisms.config.SensorConfig;
import yams.core.motorcontrollers.simulation.Sensor;

// The arm's hard stops: -10 degrees and 100 degrees.
private final Arm arm = new Arm(armConfig, motor);

// A normally open switch between the signal and power pins of Smart IO port 4.
private final DigitalInput lowerSwitchInput = new DigitalInput(4);
private final Sensor lowerSwitch = new SensorConfig("ArmLowerSwitch")
    // The input reads high (true) while the switch is pressed.
    .withField("Pressed", lowerSwitchInput::get, false)
    // In simulation, the switch is pressed while the simulated arm is at its lower hard stop.
    .withSimulatedValue("Pressed", () -> motor.getSimSupplier()
        .map(sim -> sim.getMechanismPosition().lte(Degrees.of(-9.95)))
        .orElse(false), true)
    .getSensor();
private boolean lowerSwitchWasPressed = false;

@Override
public void periodic() {
  // Home on the switch as it is pressed, before anything reads the encoder.
  boolean pressed = lowerSwitch.getAsBoolean("Pressed");
  if (pressed && !lowerSwitchWasPressed) {
    motor.setEncoderPosition(Degrees.of(-10));
  }
  lowerSwitchWasPressed = pressed;
  arm.updateTelemetry();
}
```

For an elevator, compare the height and set it in meters: `smcConfig.convertFromMechanism(sim.getMechanismPosition()).lte(Meters.of(0.0005))` and `motor.setEncoderPosition(Meters.of(0))`.

{% hint style="info" %}
With Commands v3, the mechanism's `periodic()` is not called for you: call it from `Robot.robotPeriodic()`, before the scheduler runs.
{% endhint %}

## Common pitfalls

### The switch is wired or read like a roboRIO's

A SystemCore Smart IO port is the opposite of a roboRIO DIO port. It has a pull-down resistor, not a pull-up, and runs at 3.3 V. Wire a switch between the port's **signal** and **power** pins: the input reads low (`false`) while the switch is open, and high (`true`) while it is closed. A normally open switch therefore reads `true` while it is pressed, so its field is `input.get()`. A normally closed switch reads `false` while it is pressed, so its field is `!input.get()`. See [Digital Inputs - Hardware](https://docs.wpilib.org/en/latest/docs/hardware/sensors/digital-inputs-hardware.html).

Code ported from a roboRIO robot often reads `!input.get()`, because a roboRIO's pull-up made a pressed switch read `false`. On a SystemCore that reads backwards, and the mechanism homes whenever the switch is *not* pressed.

{% hint style="danger" %}
Never wire a switch between the power and ground pins, as a roboRIO switch was wired between signal and ground: closing it connects power directly to ground, which trips the SystemCore's protection.
{% endhint %}

Check the switch before trusting it: press it by hand and watch the field on your dashboard.

### Soft limits keep the mechanism off the switch

The motor controller stops the mechanism at its soft limits. A soft limit inside the hard stop stops the mechanism before it reaches the switch, so the switch only homes the mechanism when it is already resting there, such as when the robot turns on. Put the soft limit at the hard stop, or drive the mechanism onto the switch with a homing command that moves it slowly.

### The encoder is set every loop

Set the encoder once, as the switch is pressed, not every loop while it is held. A mechanism resting on its lower switch would otherwise send the motor controller a new position over CAN every 20 ms, and a Talon waits for each one to be acknowledged.

### The switch is not where the hard stop is

Homing sets the encoder to where the mechanism is when the switch is pressed. A switch that presses a little before the hard stop homes the mechanism to the wrong position by that much. Set the encoder to the position the switch presses at: move the mechanism onto the switch by hand and read where it is.

### The switch bends or moves

The mechanism hits its switches at the hard stops, which can bend them, shift their mounts or wear their levers. A switch that moves presses at a different position, so homing sets the wrong position, and the mechanism is off by that much until it homes again. Check the switches regularly, such as in pit checks: that they are tight and straight, and that the mechanism reads the hard stop when it presses each one.

### The `Sensor` is declared before the mechanism

The simulated value's lambda uses `motor` (or the mechanism). Java does not let a field's initializer use a field declared after it, even in a lambda, so declare the `Sensor` after them.

### The simulation presses the switch from the encoder

Press the simulated switch from where the simulated mechanism really is, `motor.getSimSupplier()`, rather than from the encoder, `arm.getAngle()`. A real switch is pressed by the mechanism, not by what the encoder says. If the encoder is off, a switch pressed from the encoder is pressed in the wrong place, and homing never corrects it.

### The sensor is not read every loop

A `Sensor`'s simulated values only apply when its field is read. Read it in `periodic()`, every loop, rather than only inside a command that may not be running.

### Two devices share a Smart IO port

A duty cycle absolute encoder, such as a Through Bore Encoder on the SystemCore, also uses a Smart IO port. Give each switch and each encoder its own port.

### A SPARK does not home in simulation

In REVLib 2027.0.0-alpha-7, the simulated SPARK sets its encoder back to its own position on the next loop, whatever sets it. The switches are still pressed at the hard stops in simulation, but `setEncoderPosition` does not change the simulated SPARK's encoder. Homing a real SPARK is not affected.
