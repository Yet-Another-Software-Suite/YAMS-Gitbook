---
icon: wave-square
---

# ADRC controllers

**ADRC in YAMS was written by FRC Team 8331**, who contributed it to YAMS. This page explains how it works and how it compares to PID and LQR. To set one up, see [How do I use ADRC instead of PID?](../../how-to/how-do-i-use-adrc.md).

## What is ADRC?

**Active disturbance rejection control (ADRC)** treats everything that pushes a mechanism off course as one unknown: gravity, friction, back-EMF, a game piece, a wrong model. It estimates that unknown every loop and cancels it, so the rest of the controller only has to drive a simple, ideal mechanism.

{% hint style="info" %}
**Key difference from PID**: with PID and a feedforward, you tune gains and model gravity and friction yourself with kS, kV and kG. With ADRC, you say what kind of mechanism it is and how quickly it should settle, and it measures the disturbances as it runs.
{% endhint %}

It has three parts:

* **An extended state observer (ESO)** estimates the mechanism's position, its speed and the total disturbance from the encoder and the voltage it applied.
* **A feedback law** drives the estimated position or speed to the setpoint, like a PD controller tuned from the settling time.
* **Disturbance cancellation** subtracts the estimated disturbance, divided by `b0`, from the voltage. This is what holds an arm up without a kG.

`b0` is how strongly a volt moves the mechanism. YAMS works it out from the motor, the gearing and the moment of inertia, so you rarely set it.

## Where it runs

The ADRC always runs on the SystemCore, on every motor controller, every closed loop control period (20 ms by default). The motor controller only receives the voltage it asks for. So:

* **The same settings behave the same on every vendor.** A TalonFX, a SPARK, a Thrifty Nova and an A301 all run the same ADRC.
* **The motor config's output limits apply to it.** `withClosedLoopControllerMaximumVoltage(...)` caps its voltage, and `withClosedLoopRampRate(...)` ramps it, on every controller. The ADRC itself has no limits.
* **A motion profile on the motor config shapes its setpoint**, as it does for a PID.

## The settings

| Setting | What it does | Where it comes from |
| --- | --- | --- |
| System order | What kind of mechanism it is | You, with `withSystemOrder(...)` |
| Settling time | How quickly it reaches a setpoint | You, with `withSettlingTime(...)`. The main thing to tune |
| `b0` | How strongly a volt moves the mechanism | Worked out from the motor, gearing and moment of inertia, or `withB0(...)` |
| Bandwidth | How hard the feedback pushes | Worked out from the settling time: 2 × (order + 1) ÷ settling time, in rad/s |
| Observer bandwidth | How quickly the observer follows the disturbance | 4 × the bandwidth |
| Bandwidth scale | Scales both bandwidths together | 1 by default. The settling time does the same job |

A shorter settling time makes both the feedback and the observer faster. Too short, and the mechanism overshoots, or the voltage chatters as the observer amplifies sensor noise.

## Two kinds of position loop

A motor's back-EMF slows it down as it speeds up. How quickly it does so, at the mechanism, is the back-EMF time constant:

τ = J·R ÷ (G²·Kt·Ke)

where J is the moment of inertia, R the motor's resistance, G the gear ratio, and Kt and Ke the motor's torque and back-EMF constants. A big reduction makes it very short.

* **Heavily geared (τ under 2 control periods).** The motor reaches the speed a voltage gives it almost at once, so the voltage sets the speed. The position is a **first order** system, and `b0` is the speed per volt.
* **Lightly geared or heavy (τ of 2 control periods or more).** The voltage sets the acceleration. The position is a **second order** system, and `b0` is the acceleration per volt.

`SystemOrder.ARM`, `PIVOT`, `POSITION` and `ELEVATOR` work out τ when the controller is built and choose for you. For example, a NEO on a 60:1 arm has τ of about 6 ms and is first order; on 12:1, about 160 ms and second order.

<figure><img src="../../.gitbook/assets/adrc-pivot-forms.png" alt="Two AdvantageScope graphs of a NEO pivot on a 60:1 reduction stepping to 90, -45, 135 and 0 degrees. On the left, run as a second order system with a 0.6 second settling time, the measured angle in yellow swings far past every setpoint, up to about 215 degrees on the step to 135, and keeps oscillating. On the right, with PIVOT choosing first order and a 0.3 second settling time, the measured angle meets each blue setpoint step quickly and stays on it, and the red motor voltage spikes to 12 volts and settles back to zero after each move."><figcaption><p>The same 60:1 pivot. Left: forced to second order at 0.6 s, it rings. Right: <code>PIVOT</code> chooses first order, at 0.3 s.</p></figcaption></figure>

The wrong form works badly, so choosing it needs the moment of inertia. An `ELEVATOR` without one stays first order; `ARM`, `PIVOT` and `POSITION` refuse to start without one. `FIRST` to `FOURTH` fix the order yourself, with your own `b0`.

### Units

An elevator's ADRC works in meters, so `ELEVATOR` requires `withLinearClosedLoopController(true)`. Its `b0` is in meters per second per volt. Every other position loop works in rotations of the mechanism, and a flywheel's in rotations per second.

## `b0` and the disturbance

The observer absorbs a `b0` that is off by up to a factor of two: whatever the model misses becomes part of the disturbance it cancels. Off by more, and it struggles:

* **`b0` too high** makes the ADRC push too little, then correct late: it overshoots. On a 12:1 arm, twice the derived `b0` overshot by 13.6°.
* **`b0` too low** makes it push harder than it needs: it is slower to settle, but calmer. Half the derived `b0` cut that arm's overshoot from 2.7° to 1.2°.

`closedloop/adrc/disturbance` shows what the observer estimates. On an arm at rest it is the gravity it holds against; on a shooter at speed, mostly back-EMF.

<figure><img src="../../.gitbook/assets/adrc-shooter-graph.png" alt="An AdvantageScope graph of the ADRC shooter stepping from 0 to 50, down to 25 and up to 75 rotations per second. The yellow measured speed reaches each blue setpoint step within about half a second. The red motor voltage spikes on each step, then holds at about 6.5 volts at 50 rotations per second, 3 volts at 25 and 9.3 volts at 75, the voltage each speed needs, with no kV."><figcaption><p>The shooter finds the voltage each speed needs without a kV.</p></figcaption></figure>

## Smoothing the setpoint: the tracking differentiator

A step setpoint asks the mechanism to be somewhere else at once, and the ADRC pushes as hard as its settling time allows to get there. A **tracking differentiator** (TD) is a filter in front of the ADRC that turns the step into a glide. Each loop it gives the ADRC a smooth setpoint, `y`, and how fast that setpoint is moving, `yd`, which the ADRC uses as the setpoint's speed (or, on a flywheel, its acceleration). It is off by default.

```java
new ADRCConfig()
    .withSystemOrder(SystemOrder.ARM)
    .withSettlingTime(Seconds.of(0.3))
    // A 0.08 s time constant: a 90° step is followed in about 0.46 s.
    .withTrackingDifferentiator(TrackingDifferentiator.Method.LD, 0.08)
```

A [motion profile](exponential-profiles.md) on the motor config does the same job with real speed and acceleration limits. Use one or the other: a TD on top of a profile only adds lag. The TD is set when the robot starts, and is not live tunable.

### The methods

The TD works in the loop's units: mechanism rotations for an arm or pivot, meters for an elevator, rotations per second for a flywheel. Parameters are passed in order; any left off take a default, and the defaults are slow, so set them.

| Method | Parameters (default) | How it moves to a new setpoint |
| --- | --- | --- |
| `LD`, linear | `lambda` (1): a time constant, seconds | Glides there with no overshoot, reaching 98% in about 5.8 × `lambda`, at a peak speed of about step ÷ (2.7 × `lambda`). |
| `INTD`, saturating | `alpha` (0.5), `beta` (1), `gamma` (1), `r` (1) | Cruises at `r × beta ÷ (1 - alpha)` when far away, then lands smoothly. `gamma` = 4 × `beta` lands without overshoot. |
| `EULER`, exponential | `alpha` (0.9), 0 to 1 | A first order lag. Its `yd` still jumps on a step. |
| `TOD`, time optimal | `r` (1): an acceleration | Full acceleration toward the setpoint, then full braking. |
| `RED`, nonlinear | `lambda1` (1), `lambda2` (1) | A super-twisting filter. Overshoots. |

### Choosing and tuning one

Measured on the TD alone at the 20 ms default period, stepping an arm 90°:

| Method and parameters | Within 2% after | Overshoot |
| --- | --- | --- |
| `LD` 1 (default) | 5.8 s | 0% |
| `LD` 0.1 | 0.52 s | 0% |
| `LD` 0.05 | 0.22 s | 0% |
| `INTD` 0.5, 1, 1, 1 (default) | 8.1 s | 17% |
| `INTD` 0.5, 1, 4, 20 | 0.52 s | 0% |
| `TOD` 4 | Never holds it | 8% |
| `RED` 3, 4 | 0.40 s | 13% |

* **Start with `LD`.** Its one parameter is in seconds, means the same on every mechanism, and never overshoots. Pick `lambda` so 5.8 × `lambda` is how long a move should take, then check the mechanism can reach the peak speed, step ÷ (2.7 × `lambda`). Lengthen it if the voltage saturates during a move.
* **Use `INTD` when step sizes vary a lot.** Leave `alpha` at 0.5, choose `beta` and `r` so the cruise speed `r × beta ÷ (1 - alpha)` is a little under the mechanism's top speed, and set `gamma` to 4 × `beta`. Raise `r` to land faster.
* **Avoid `EULER`, `TOD` and `RED`.** `EULER`'s `yd` kicks on every step, which is what a TD is meant to prevent. `TOD` and `RED` switch sign every loop, so at a 20 ms period their `yd` chatters at rest, and the ADRC passes the chatter to the motor.

The [ADRC guide](https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/ADRCGuide.md#smoothing-the-setpoint-the-tracking-differentiator) has every parameter's units and the elevator and flywheel measurements.

## ADRC, PID or LQR?

| | PID + feedforward | LQR | ADRC |
| --- | --- | --- | --- |
| What you tune | kP, kI, kD, kS, kV, kA, kG | Error and effort tolerances | Settling time, maybe `b0` |
| Model needed | None, but tuning stands in for one | Motor, gearing, inertia, accurately | Motor, gearing, inertia, roughly |
| Gravity and friction | You model them with kG and kS | You add a feedforward | Estimated and cancelled |
| Where it runs | On the motor controller where it can | SystemCore | SystemCore |
| Loop rate | Up to 1 kHz on the controller | 50 Hz by default | 50 Hz by default |

Reach for ADRC when a mechanism's load changes, or its gravity and friction are hard to model. Keep a PID on the motor controller when you need its 1 kHz loop, or already have gains that work.

## Telemetry

At `HIGH` verbosity, the ADRC publishes under the motor's `closedloop/adrc/` entry. `b0`, `settlingTime` and `bandwidthScale` are also tunable under `NT:/Tuning`. See [ADRC keys](../live-tuning/keys-and-units.md#adrc-keys).

## Using it without a mechanism

`ADRCController` also runs on its own, like a WPILib `PIDController`: call `calculate(measurement, setpoint)` each loop and apply its voltage. It does not limit its output, so clamp it, and wrap it in a WPILib `SlewRateLimiter` if it needs a ramp. Then tell it what it really applied with `calculate(measurement, setpoint, appliedVoltage)`, so it does not mistake the clamp for a disturbance. The [ADRC guide](https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/ADRCGuide.md) covers its full API, the tracking differentiator, input delay and the cascaded observer.
