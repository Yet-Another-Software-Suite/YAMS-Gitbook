---
icon: bolt-lightning
---

# Limiting power consumption

## Current limits

There are two types of currents involved in a `SmartMotorController`, the **Supply** current and the **Stator** current.

* The current input of the motor controller is the **Supply** current.
* The current output of the motor controller is the **Stator** current.

<figure><img src="../../.gitbook/assets/power-supply-stator.png" alt="A battery at 12 volts feeds a motor controller running at 25 percent output, which feeds a motor with 3 volts across it. The supply current is 20 amps at 12 volts and the stator current is 80 amps at 3 volts, the same 240 watts. The supply current limit protects the battery, breaker and bus voltage. The stator current limit protects the motor and limits torque"><figcaption><p>The two currents, and what each limit protects.</p></figcaption></figure>

{% hint style="danger" %}
Some motor controllers, like SparkMax and SparkFlex, only support a stator current limit. `SparkWrapper` throws a `SmartMotorControllerConfigurationException` if you set a supply current limit.
{% endhint %}

CTRE has excellent documentation explaining the relationship between the supply and stator currents.

{% embed url="https://v6.docs.ctr-electronics.com/en/stable/docs/hardware-reference/talonfx/improving-performance-with-current-limits.html" %}

If your mechanism pushes on a game piece or can get stuck, a current limit **will** make it stutter while the current is held at the limit. That strains the mechanism a little, but it protects the motor and stops one stuck mechanism from browning out the robot.

### Setting current limits in YAMS

```java
.withStatorCurrentLimit(Amps.of(40)) // stator: current through the motor, limits torque and heat
.withSupplyCurrentLimit(Amps.of(30)) // supply: current from the battery, protects the breaker and voltage
```

This is a fragment of a `SmartMotorControllerConfig`. Leave out the supply limit on a SPARK MAX or SPARK Flex, which only has a stator limit.

### Starting limits

These are starting points for a mechanism, not rules. Raise a limit if the mechanism is too weak or too slow, and lower it if the battery voltage dips too far in the [simulation test below](#testing-brownout-scenarios-in-simulation). Drive motors usually run higher stator limits than mechanisms.

| Motor      | Usually on                    | Stator limit to start | Supply limit to start             |
| ---------- | ----------------------------- | --------------------- | --------------------------------- |
| Kraken X60 | TalonFX                       | 60 A                  | 40 A                              |
| NEO        | SPARK MAX, TalonFXS, Nova     | 40 A                  | 30 A, where the controller has one |
| NEO 550    | SPARK MAX, TalonFXS           | 20 A                  | 20 A, where the controller has one |
| NEO Vortex | SPARK Flex, TalonFXS          | 60 A                  | 40 A, where the controller has one |

A NEO 550 is small and overheats quickly when stalled, so keep its stator limit low on intakes and rollers that can jam.

## Ramp rates

A ramp rate is the shortest time the motor's output may take to go from 0% to 100%.

{% hint style="info" %}
A ramp rate is the last thing to reach for. It delays everything the motor does, which makes the robot seem slow. Try the fixes in [Limit each load in this order](#limit-each-load-in-this-order) first.
{% endhint %}

<figure><img src="../../.gitbook/assets/power-ramp-rate.png" alt="Without a ramp rate the motor output jumps to full at once and the current spikes. With a 0.25 second ramp rate the output climbs over 0.25 seconds and the current is a lower, wider bump, with a slower start"><figcaption><p>A ramp rate trades the current spike for a slower start.</p></figcaption></figure>

There are two ramp rates defined in motor controllers.

* Closed loop ramp rate
* Open loop ramp rate

These ramp rates are applied whenever the motor controller is in the appropriate mode. So the **closed loop ramp rate** is used when the motor controller **PID** is used. The **open loop ramp rate** is used when the dutycycle or voltage control is used.

{% hint style="warning" %}
The Thrifty Nova has a single ramp rate for open and closed loop control. `NovaWrapper` throws a `SmartMotorControllerConfigurationException` if the two ramp rates differ, so set only one of them, or both to the same value.
{% endhint %}

### Setting ramp rates in YAMS

```java
.withClosedLoopRampRate(Seconds.of(0.25)) // at least 0.25 s from 0 to full output under closed loop
.withOpenLoopRampRate(Seconds.of(0.25))   // the same for voltage and duty cycle commands
```

## Testing brownout scenarios in simulation

YAMS combines the current draw of every simulated mechanism into one shared battery model, so simulated voltage sag reflects the _whole_ robot pulling power at once, not just one mechanism in isolation. That makes simulation the place to choose your limits: run the busiest moment of your match, watch the battery voltage, and lower the limits on the biggest loads until the dip is acceptable.

<figure><img src="../../.gitbook/assets/power-limits-battery.gif" alt="Animation of the current drawn by a swerve drive, shooter, elevator and arm, with each mechanism's supply current limit marked. Two battery voltage traces are drawn: without limits the voltage dips to about 8.8 volts when the peaks stack, and with limits it stays near 9.7 volts"><figcaption><p>The same few seconds with and without supply current limits. Example currents and limits.</p></figcaption></figure>

{% stepper %}
{% step %}
#### Find the worst moment

In simulation, do the things that happen together in a match: accelerate the drive, spin up the shooter, raise the elevator. Graph the battery voltage and each motor's `current/supply`.
{% endstep %}

{% step %}
#### Start with the biggest load

The drive is usually the largest draw, so limiting it does the most. Then the next largest.
{% endstep %}

{% step %}
#### Limit each load in this order

Stop at the first thing that brings the dip under control:

1. A [motion profile](config/motion-profiles.md), if one suits the mechanism, so it asks for less current in the first place.
2. A supply current limit, where the motor controller has one.
3. A stator current limit, where it does not. SPARK MAX and SPARK Flex have no supply limit.
4. A ramp rate, only if nothing else works. It slows everything the motor does.
{% endstep %}

{% step %}
#### Tune the limits live

At `TelemetryVerbosity.HIGH`, `current/limit/supply` and `current/limit/stator` are tunable under `NT:/Tuning/`. Change them while the simulation runs and watch the dip shrink. See [Live tuning](../live-tuning/README.md).
{% endstep %}

{% step %}
#### Check the mechanism still does its job

A limit that is too low makes the mechanism slow or stalls it. Lower it until the voltage is acceptable, not further.
{% endstep %}
{% endstepper %}

See [Battery Simulation](../../understanding/battery-simulation.md) for how each mechanism loads the battery, and how to enable realistic discharge over a match to reproduce brownout-prone scenarios before they happen on the field.
