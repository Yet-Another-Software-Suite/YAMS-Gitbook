---
icon: bolt-lightning
---

# Limiting Power Consumption

## Current limits

There are two types of currents involved in a `SmartMotorController`, the **Supply** current and the **Stator** current.&#x20;

* The current input of the motor controller is the **Supply** current.&#x20;
* The current output of the motor controller is the **Stator** current.

{% hint style="danger" %}
Some motor controllers, like SparkMax and SparkFlex do not let you set these separately
{% endhint %}

CTRE has excellent documentation explaining the relationship between the supply and stator currents.

{% embed url="https://v6.docs.ctr-electronics.com/en/stable/docs/hardware-reference/talonfx/improving-performance-with-current-limits.html" %}

If your mechanism interacts with a game element or has a chance of getting stuck a current limit **WILL** cause the mechanism to stutter until the current is back below the limit. This can cause strains on the mechanism as it constantly starts then stops but it could save your motor controller and prevent brownouts because your mechanism will not be able to pull more power than necessary.

### Setting Current Limits in YAMS

```java
SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withStatorCurrentLimit(Amps.of(200)) // Current to the motor
      .withSupplyCurrentLimit(Amps.of(40)) // Current to the motor controller
```

## Ramp Rates

Ramp rates is the minimum time it takes for a motor to go from 0% to 100% power to the motor.&#x20;

{% hint style="info" %}
If the ramp rate is omitted from configurations you will commonly see mechanisms taking up a majority of power whenever they go from rest to moving with a closed loop controller.

Whenever possible use current limits instead of ramp rates. Ramp rates delay input response and make the robot seem "slow" or "delayed".
{% endhint %}

There are two ramp rates defined in motor controllers.

* Closed loop ramp rate
* Open loop ramp rate

These ramp rates are applied whenever the motor controller is in the appropriate mode. So the **closed loop ramp rate** is used when the motor controller **PID** is used. The **open loop ramp rate** is used when the dutycycle or voltage control is used.

### Setting Ramp Rates in YAMS

```java
SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
      .withClosedLoopRampRate(Seconds.of(0.25)) // 0.25 seconds minimum 
      .withOpenLoopRampRate(Seconds.of(0.25)) // 0.25 seconds minimum
```

## Testing brownout scenarios in simulation

YAMS combines the current draw of every simulated mechanism into one shared battery model, so simulated voltage sag reflects the *whole* robot pulling power at once, not just one mechanism in isolation. See [Battery Simulation](battery-simulation.md) for how to enable realistic discharge over a match and reproduce brownout-prone scenarios before they happen on the field.
