---
icon: chart-line
---

# Motion profiles

A motion profile turns a jump in the goal into a smooth path of setpoints for the closed loop controller to follow. Setting one profile replaces any other.

## `withTrapezoidalProfile`

A maximum velocity and a maximum acceleration.

<figure><img src="../../../.gitbook/assets/smc-trapezoidal-profile.png" alt="Two graphs. Position: the goal is a flat line and the setpoint the PID follows rises to it in a smooth S shape. Velocity: the setpoint&#x27;s velocity ramps up at the maximum acceleration, holds at the maximum velocity, and ramps back down, making a trapezoid."><figcaption><p>The setpoint glides to the goal, and its velocity makes a trapezoid.</p></figcaption></figure>

```java
.withTrapezoidalProfile(DegreesPerSecond.of(180), DegreesPerSecondPerSecond.of(360)) // arms, pivots
.withTrapezoidalProfile(MetersPerSecond.of(1.5), MetersPerSecondPerSecond.of(3))     // elevators
```

The distance version needs a [circumference](gearing-and-units.md#withmechanismcircumference) and makes the controller [distance based](closed-loop-control.md#withlinearclosedloopcontroller).

Choose limits the mechanism can really reach. If the profile asks for more than the motor can give, the mechanism falls behind and the PID fights to catch up: see [when tuning does not help](../../live-tuning/#when-tuning-does-not-help).

## A velocity profile

For mechanisms that hold a speed, the same method takes a maximum acceleration and a maximum jerk instead.

<figure><img src="../../../.gitbook/assets/smc-velocity-profile.png" alt="A graph of velocity against time. The requested speed jumps up in a step. The setpoint rises to it in a smooth curve limited by a maximum acceleration and a maximum jerk, which is how quickly acceleration changes."><figcaption><p>A velocity setpoint limited by acceleration and jerk.</p></figcaption></figure>

```java
.withTrapezoidalProfile(RotationsPerSecondPerSecond.of(200),
                        RotationsPerSecondPerSecond.per(Second).of(2000))
```

`withVelocityTrapezoidalProfile(true)` marks an existing trapezoidal profile as a velocity profile. The overload above does that for you.

## `withExponentialProfile`

An exponential profile follows what a DC motor can actually do: hard acceleration at first, easing off as it speeds up.

<figure><img src="../../../.gitbook/assets/exponential-profile-compare.png" alt="A comparison of a trapezoidal profile, which accelerates at a constant rate, and an exponential profile, which accelerates hard at first and eases off as the motor speeds up"><figcaption><p>Trapezoidal and exponential profiles.</p></figcaption></figure>

```java
.withExponentialProfile(Volts.of(10), DCMotor.getKrakenX60(1), KilogramSquareMeters.of(0.5)) // arm or flywheel
.withExponentialProfile(Volts.of(10), DCMotor.getKrakenX60(2), Kilograms.of(8), Inches.of(1)) // elevator
.withExponentialProfile(Volts.of(10), RotationsPerSecond.of(2), RotationsPerSecondPerSecond.of(8))
```

You give it a maximum voltage and either the mechanism (its motor and its moment of inertia, or its mass and drum radius) or a maximum velocity and acceleration. YAMS works out starting values of the profile's kV and kA from those. Tune kV and kA directly from there, as CTRE's Motion Magic Expo documentation describes, and pass your tuned values with `ExponentialProfile.Constraints.fromCharacteristics(maxVoltage, kV, kA)`.

[Exponential profiles](../exponential-profiles.md) explains when to use one and how to tune it.

## Where each profile runs

**Supported on every motor controller.** Every controller accepts both profiles, but they do not run in the same place.

| TalonFX                                               | TalonFXS                                              | SPARK MAX / Flex                                                           | Thrifty Nova             |
| ----------------------------------------------------- | ----------------------------------------------------- | -------------------------------------------------------------------------- | ------------------------ |
| ✅ on the controller (Motion Magic, Motion Magic Expo) | ✅ on the controller (Motion Magic, Motion Magic Expo) | ✅ trapezoidal on the controller (MAXMotion), exponential on the SystemCore | ✅ both on the SystemCore |

Where the profile runs decides which other options are available: see [where the loop runs](closed-loop-control.md#where-the-loop-runs).

## Code reference

* [`SmartMotorControllerConfig`](https://yet-another-software-suite.github.io/YAMS/javadocs/yams/motorcontrollers/SmartMotorControllerConfig.html)
