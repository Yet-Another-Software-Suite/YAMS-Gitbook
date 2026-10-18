---
icon: gear
layout:
  width: default
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
  actions:
    visible: true
---

# Smart Motor Controllers

## A quick primer on Motors and Motor Controllers

{% embed url="https://youtu.be/0BfR_viZSmQ" %}

{% embed url="https://youtu.be/pIKvQX5p1PE" %}

The presentation from the video series continues onto PID, Feedforward, and Motion Profiles as well!

{% file src="../../.gitbook/assets/Motors.pdf" %}

## The difference between Rotor, Mechanism, and Measurement

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

* **Rotor** is the motor output shaft.
* **Mechanism** is the gearbox AND sprocket output shaft.
* **Measurement** is the distance that the **Mechanism** travelled via its [**Mechanism Circumference**](../../how-to/how-to-find-your-mechanism-circumference.md)**.**

{% hint style="info" %}
Cascading elevator **Mechanism** gearings should be divided by the stages in the Elevator!
{% endhint %}

An easy way to do this with YAMS is by using `GearBox.fromReductionStages`and `Sprocket.fromStages`!

## Configuring a motor

Everything about a motor is described by a `SmartMotorControllerConfig`: gearing, closed loop control, limits, encoders, followers and simulation. The [SmartMotorControllerConfig](config/README.md) pages explain every option with a graphic and an example, and say which motor controllers support it.

{% hint style="warning" %}
Not every option works on every motor controller. See [which options are not supported everywhere](config/README.md#not-every-option-works-on-every-motor-controller).
{% endhint %}

## Previous Configurations

YAMS resets the motor controller's previous configuration by default, so settings left over from another run cannot turn into a headache. [`withResetPreviousConfig`](config/motor-basics.md#withresetpreviousconfig) changes that.

## Power optimization is crucial to prevent brownouts

Power optimization can be done with [supply, and stator current limits, ramp rates](limiting-power-consumption.md), and adequately geared gearboxes and motors. In simulation, every mechanism's current draw feeds into one shared [Battery Simulation](../../understanding/battery-simulation.md), so voltage sag from combined load, and, optionally, a battery discharging over a match, is realistic before you ever get to the field.

## Tuning our closed loop controllers

{% hint style="info" %}
## Motion Profiles

The rule of thumb is:

1. Use trapezoid profile if your DC motor is always current-limited or you’re doing torque control
2. Use exponential profile if your DC motor is never current-limited
3. Use [the algorithm in this whitepaper](https://www.chiefdelphi.com/t/whitepaper-trapezoidal-exponential-motion-profiling/443468/12?u=nstrike) if you’re sometimes current-limited (not implemented in YAMS, yet)
{% endhint %}

Every `SmartMotorController` in YAMS supports motion profiled PID and feedforward, though not always in the same place: see [where the loop runs](config/closed-loop-control.md#where-the-loop-runs). The options are covered in [Closed loop control](config/closed-loop-control.md), [Feedforward](config/feedforward.md) and [Motion profiles](config/motion-profiles.md), and [Live tuning](../live-tuning/README.md) shows how to find the values.

Simulation never exactly matches real life, so you can set [separate PID and feedforward values for simulation only](simulation-only-pid-+-feedforward.md).

You can learn how to tune PID and feedforwards from WPILib here!

{% embed url="https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/tuning-turret.html" %}

## External encoders

YAMS accepts external encoders that the motor controller can read itself, which means the same vendor. [Encoders](config/encoders.md) covers attaching one, its gearing, its zero offset and which encoder the closed loop follows, and the [feature matrix](feature-matrix.md#external-encoders) lists the encoders each wrapper accepts.

{% hint style="danger" %}
Absolute encoders cannot go beyond 1 rotation or 360 degrees, even less with discontinuity points!

DO **NOT** USE AN ABSOLUTE ENCODER WITH A GEAR RATIO WITH AN INPUT GREATER THAN 1 UNLESS YOU KNOW WHAT YOU'RE DOING!
{% endhint %}

## Follower motors

There are two kinds of follower. **Tightly coupled** followers are motors on the same shaft or gearbox and use [`withFollowers`](config/followers.md#withfollowers). **Loosely coupled** followers are motors that are not directly connected but must move together, and use [`withLooselyCoupledFollowers`](config/followers.md#withlooselycoupledfollowers). [Followers](config/followers.md) has the full comparison.

<figure><img src="../../.gitbook/assets/smc-followers.png" alt="Two panels. Tightly coupled, withFollowers: a leader and a follower are on one shaft or gearbox and the follower copies the leader's output. The motors are mechanically locked, there is one closed loop and one config, the followers are the same vendor as the leader, and voltage and duty cycle reach every motor. Loosely coupled, withLooselyCoupledFollowers: two separate motors joined only by the mechanism get the same setpoint and each runs its own PID. They are not mechanically locked, each has its own closed loop and config, vendors can be mixed, and voltage and duty cycle are not forwarded, so the leader moves alone."><figcaption><p>Tightly coupled followers share a shaft. Loosely coupled followers are separate motors.</p></figcaption></figure>

### Loosely Coupled Follower Motors

Each loosely coupled follower is sent the leader's position and velocity setpoints, and reaches them with its own closed loop. A mechanism class works with them: give it the leader.

{% hint style="warning" %}
Loosely coupled followers are **not** sent voltage or duty cycle requests, except a duty cycle of 0 to stop. They are not sent `setEncoderPosition`, and are not updated with the leader: call `updateTelemetry()` and `simIterate()` on each one. [What reaches the followers](config/followers.md#what-reaches-the-followers) has the full list.
{% endhint %}

## You don't have to rely on YAMS to configure your SMC

YAMS is not the only way to configure your SmartMotorController! You can always fetch the config object, if available, from `SmartMotorController` like shown below

```java
(SparkFlex)smartMotorController.getMotorController();
(SparkFlexConfig)smartMotorController.getMotorControllerConfig();
```

## Some Mechanism config options modify SMC configs

Mechanism options like `.withSoftLimits` often will call the `SmartMotorControllerConfig.withSoftLimits` and do nothing else. This is to provide a seamless configuration for relevant areas in different configs.

## Simulation without a mechanism

`SmartMotorController`s require `SmartMotorController.simIterate` to be called in `simulationPeriodic` of your subsystem to provide simulation IF your motor controller is not used by a mechanism. It also proper to include the `SmartMotorController.updateTelemetry` as well.

```java
  @Override
  public void periodic()
  {
    smartMotorController.updateTelemetry();
  }
  
  @Override
  public void simulationPeriodic()
  {
    smartMotorController.simIterate();
  }
```

## Passing a Vendor Config

`SmartMotorControllerConfig` can start from a vendor config that you supply, so vendor specific settings can be set alongside the YAMS options. See [`withVendorConfig`](config/motor-basics.md#withvendorconfig).
