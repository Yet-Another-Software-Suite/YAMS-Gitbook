---
icon: arrows-spin
---

# Closed loop control

Closed loop control is what takes a mechanism to a setpoint and holds it there. These options need [`withControlMode(ControlMode.CLOSED_LOOP)`](motor-basics.md#withcontrolmode), which is the default.

{% hint style="warning" %}
Not every option on this page is supported on every motor controller. Each option has a table showing where it works, and an option a controller cannot do throws a `SmartMotorControllerConfigurationException` or an `IllegalArgumentException` at startup. Each option says which. The [config overview](README.md#not-every-option-works-on-every-motor-controller) lists them all together.
{% endhint %}

For how to find good values, see [live tuning](../../live-tuning/README.md).

## `withClosedLoopController`

Sets the PID gains.

<figure><img src="../../../.gitbook/assets/smc-pid.png" alt="A graph of position against time moving to a setpoint with three sets of gains: a low kP that is slow, a high kP that overshoots, and kP with kD that is quick and settled. kP pushes harder the further away, kI pushes harder the longer it is off, kD eases off as it gets close. The input is rotations, or meters for a distance mechanism, and the output is volts."><figcaption><p>What the gains do to a move.</p></figcaption></figure>

```java
.withClosedLoopController(4, 0, 0.2)                 // kP, kI, kD
.withClosedLoopController(new PIDController(4, 0, 0.2))
```

The gains work in rotations and volts, or meters and volts for a [distance based controller](#withlinearclosedloopcontroller). YAMS converts them to whatever the motor controller uses, so the same numbers behave the same on every vendor.

Passing an `LQRController` instead replaces the PID with an LQR: see [LQR controllers](../lqr-controllers.md).

**Supported on every motor controller.** Every controller accepts PID gains. An [LQR controller](../lqr-controllers.md) always runs on the SystemCore, on every controller.

| TalonFX | TalonFXS | SPARK MAX / Flex | Thrifty Nova |
| --- | --- | --- | --- |
| ✅ | ✅ | ✅ | ✅ PID runs on the SystemCore |

### Slots

Each call can name a slot, so one motor can hold several sets of gains and switch between them.

<figure><img src="../../../.gitbook/assets/smc-slots.png" alt="Four slots, SLOT_0 to SLOT_3, each holding its own PID gains and feedforward, for example one for holding nothing, one for holding a game piece and one for climbing. SLOT_0 is in use. SLOT_3 is not available on TalonFX or TalonFXS. motor.setClosedLoopSlot switches slot at once."><figcaption><p>Four slots, each with its own gains and feedforward.</p></figcaption></figure>

```java
.withClosedLoopController(4, 0, 0, ClosedLoopControllerSlot.SLOT_0)
.withClosedLoopController(6, 0, 0, ClosedLoopControllerSlot.SLOT_1)
// later, while running
motor.setClosedLoopSlot(ClosedLoopControllerSlot.SLOT_1);
```

**Not supported everywhere.** Selecting `SLOT_3` on a TalonFX or TalonFXS throws an `IllegalArgumentException`, and a feedforward configured for `SLOT_3` on a TalonFX throws a `SmartMotorControllerConfigurationException`.

| TalonFX | TalonFXS | SPARK MAX / Flex | Thrifty Nova |
| --- | --- | --- | --- |
| ⚠️ `SLOT_0` to `SLOT_2` | ⚠️ `SLOT_0` to `SLOT_2` | ✅ | ✅ |

## `withClosedLoopTolerance`

How close to the setpoint counts as being there.

<figure><img src="../../../.gitbook/assets/smc-tolerance.png" alt="A graph of position moving to a setpoint with a shaded band of plus and minus the tolerance around it. Once the position stays inside the band the mechanism counts as at the setpoint."><figcaption><p>The tolerance is a band around the setpoint.</p></figcaption></figure>

```java
.withClosedLoopController(4, 0, 0)
.withClosedLoopTolerance(Degrees.of(1))
```

Call it after `withClosedLoopController`: it is set on the PID controllers that exist at that moment, and throws a `SmartMotorControllerConfigurationException` if there are none.

**Not supported everywhere.** It throws a `SmartMotorControllerConfigurationException` on a TalonFX or TalonFXS.

| TalonFX | TalonFXS | SPARK MAX / Flex | Thrifty Nova |
| --- | --- | --- | --- |
| ❌ | ❌ | ⚠️ not with an [LQR controller](../lqr-controllers.md) | ⚠️ not with an [LQR controller](../lqr-controllers.md) |

## `withContinuousWrapping`

Tells the controller that the two ends of the range are the same place, so it takes the short way round.

<figure><img src="../../../.gitbook/assets/smc-continuous-wrapping.png" alt="Two dials going from 170 degrees to minus 170 degrees. Without continuous wrapping the mechanism goes 340 degrees the long way round. With continuous wrapping from minus 180 to 180 degrees it goes 20 degrees across the seam, the short way."><figcaption><p>From 170° to -170°, without and with continuous wrapping.</p></figcaption></figure>

```java
.withClosedLoopController(50, 0, 0.5)
.withContinuousWrapping(Degrees.of(-180), Degrees.of(180))
```

It needs a closed loop controller first, and cannot be combined with [soft limits](limits-and-protection.md#withsoftlimits) or a distance based controller, because a mechanism with ends cannot wrap.

**Supported on:** every supported motor controller.

## `withClosedLoopControllerMaximumVoltage`

Caps the voltage the closed loop controller may ask for.

<figure><img src="../../../.gitbook/assets/smc-max-voltage.png" alt="A graph of closed loop output against time. The controller asks for nearly 12 volts at the start of a move, but the motor only gets the 6 volt maximum until the request drops below it."><figcaption><p>A 6 V maximum on a move that asked for more.</p></figcaption></figure>

```java
.withClosedLoopControllerMaximumVoltage(Volts.of(6))
```

**Not supported everywhere.** On a TalonFX or TalonFXS it is applied as the peak forward and reverse voltage. On a SPARK it throws a `SmartMotorControllerConfigurationException` unless the loop runs on the SystemCore.

| TalonFX | TalonFXS | SPARK MAX / Flex | Thrifty Nova |
| --- | --- | --- | --- |
| ✅ | ✅ | ⚠️ only with an exponential profile or an [LQR controller](../lqr-controllers.md) | ✅ |

## Where the loop runs

Some combinations run on the motor controller, at 1 kHz, and some on the SystemCore, every 20 ms by default. It matters for a few options, and YAMS raises an alert when the loop is on the SystemCore.

<figure><img src="../../../.gitbook/assets/smc-where-it-runs.png" alt="A table of where the closed loop runs. TalonFX and TalonFXS run PID and profiles on the controller and LQR on the SystemCore. SPARK MAX and SPARK Flex run PID and trapezoidal profiles on the controller, and exponential profiles and LQR on the SystemCore. The Thrifty Nova runs everything on the SystemCore. Below it, three timelines over 240 milliseconds show how often each loop runs: on the controller every 1 millisecond, which is 1 kilohertz and 240 runs; on the SystemCore every 20 milliseconds by default, which is 12 runs; and on the SystemCore at 10 milliseconds, which is 24 runs. withClosedLoopControlPeriod only changes the SystemCore loop. The period of the loop on the controller cannot be changed."><figcaption><p>Where the closed loop runs, and how often.</p></figcaption></figure>

### PID gains do not carry over between the two

PID gains tuned with the loop on the motor controller will never be the same as gains tuned with the loop on the SystemCore, even on the same mechanism. The reason is the closed loop control period:

* **On the motor controller** the loop runs at 1 kHz, once every millisecond, right next to the motor and its encoder. That is 20 runs for every one run of the SystemCore loop at its default period. This period is fixed by the motor controller and cannot be changed.
* **On the SystemCore** the loop runs once every control period, 20 ms by default, and each measurement and each output also has to cross the CAN bus.

A slower loop reacts later to every error, so it cannot use gains as high as a fast one before it starts to oscillate. `kD` and `kI` are affected the most, because they depend directly on the time between two runs of the loop.

What this means in practice:

* **Tune again whenever the loop moves.** On a SPARK that happens when you switch between a trapezoidal profile, which runs on the controller, and an exponential profile or an [LQR controller](../lqr-controllers.md), which run on the SystemCore. Moving a mechanism to a different brand of motor controller can move the loop too.
* **Tune again after changing [`withClosedLoopControlPeriod`](#withclosedloopcontrolperiod).** A different period is a different loop.
* **Do not copy gains from another team or another robot** without checking where their loop ran and how fast.

### `withClosedLoopControlPeriod`

Changes how often the SystemCore loop runs. The default is 20 ms. It does not change a loop that runs on the motor controller: that period is fixed at 1 ms (1 kHz).

```java
.withClosedLoopControlPeriod(Milliseconds.of(10))
```

**Not supported everywhere.** Where it is not supported it throws an `IllegalArgumentException` on a TalonFX, TalonFXS or Thrifty Nova, and a `SmartMotorControllerConfigurationException` on a SPARK.

| TalonFX | TalonFXS | SPARK MAX / Flex | Thrifty Nova |
| --- | --- | --- | --- |
| ❌ | ❌ | ⚠️ only with an exponential profile or an [LQR controller](../lqr-controllers.md) | ⚠️ only in closed loop mode |

## `withLinearClosedLoopController`

Chooses whether the controller works in rotations or in meters.

<figure><img src="../../../.gitbook/assets/smc-linear-units.png" alt="Two panels. Angle based, the default: error is measured in rotations, kP is in volts per rotation and profile limits are in rotations per second, for arms, pivots, turrets and flywheels. Distance based: error in meters, kP in volts per meter and profile limits in meters per second, for elevators and other linear mechanisms."><figcaption><p>The units the gains and limits are in.</p></figcaption></figure>

It turns on by itself when you give a distance profile, a distance exponential profile or an `ElevatorFeedforward`, and it only takes effect when a [circumference](gearing-and-units.md#withmechanismcircumference) is set. [How to disable linear closed loop control?](../../../how-to/how-to-disable-linear-closed-loop-control.md) explains when to turn it off.

**Supported on:** every supported motor controller.


## Code Reference

* [`SmartMotorControllerConfig`](https://yet-another-software-suite.github.io/YAMS/javadocs/yams/motorcontrollers/SmartMotorControllerConfig.html)
