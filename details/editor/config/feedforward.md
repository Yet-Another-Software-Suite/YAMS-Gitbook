---
icon: forward
---

# Feedforward

A feedforward predicts the voltage a move needs from what you know about the mechanism, before any error has built up. The PID then only has to fix what the prediction missed.

<figure><img src="../../../.gitbook/assets/smc-feedforward-terms.png" alt="A bar of the output voltage split into parts: kS to overcome friction, kG to hold against gravity, kV times velocity to keep moving, kA times acceleration to speed up, and a small PID part that fixes whatever is left. All but the PID part are feedforward, known before the mechanism moves."><figcaption><p>The parts of the output voltage at one moment in a move.</p></figcaption></figure>

## `withFeedforward`

Takes one of WPILib's three feedforward classes.

<figure><img src="../../../.gitbook/assets/smc-feedforward-types.png" alt="Three feedforward types. SimpleMotorFeedforward, for flywheels, rollers and turrets, has kS, kV and kA and nothing to hold up. ArmFeedforward, for arms and pivots, adds kG times the cosine of the angle. ElevatorFeedforward, for elevators, adds a constant kG."><figcaption><p>The three feedforward types.</p></figcaption></figure>

```java
.withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0.01))   // kS, kV, kA
.withFeedforward(new ArmFeedforward(0.1, 0.35, 1.2, 0.02))      // kS, kG, kV, kA
.withFeedforward(new ElevatorFeedforward(0.1, 0.4, 3.5, 0.1))   // kS, kG, kV, kA
```

| Gain | What it is for | How to find it |
| --- | --- | --- |
| `kS` | The voltage that just overcomes friction. | Raise the voltage slowly until the mechanism starts to move. |
| `kG` | The voltage that holds the mechanism against gravity. | The voltage at which it neither rises nor falls. For an arm, measure it with the arm level. |
| `kV` | The voltage per unit of speed. | Volts above `kS` divided by the steady speed it gives. |
| `kA` | The voltage per unit of acceleration. | Usually small. Raise it if the mechanism lags at the start of a move. |

`ArmFeedforward` and `SimpleMotorFeedforward` gains are per rotation of the mechanism. `ElevatorFeedforward` gains are per meter, and giving one makes the controller [distance based](closed-loop-control.md#withlinearclosedloopcontroller).

[Live tuning](../../live-tuning/README.md) has the full order for finding these.

### These are not an exponential profile's `kV` and `kA`

An [exponential profile](../exponential-profiles.md) also has a `kV` and a `kA`. They share the names and the units, but they are separate settings that do different jobs, and changing one pair does not change the other.

| | Feedforward `kV` and `kA` | Exponential profile `kV` and `kA` |
| --- | --- | --- |
| Set with | `withFeedforward(...)` | `withExponentialProfile(...)` |
| What they do | Turn the velocity and acceleration of the setpoint into volts for the motor. | Shape the plan. The maximum voltage divided by `kV` is the profile's top speed, and divided by `kA` is its maximum acceleration. |
| Raising them | Sends the motor more voltage. | Makes the profile slower. No extra voltage is sent. |
| Tuned by | Watching how well the mechanism follows the setpoint. | Watching whether the mechanism can keep up with the profile. |

A mechanism with an exponential profile needs both pairs: the profile's to plan the move, and the feedforward's to follow it. Do not copy one pair into the other. The profile's values are often set higher than the feedforward's, so the plan asks for less than the motor can give and leaves voltage in reserve.

[Measuring kV and kA](../exponential-profiles.md#measuring-kv-and-ka-optional) on the exponential profiles page measures the profile's pair, and [refining the profile live](../exponential-profiles.md#refining-the-profile-live) tunes them.

### Slots

A feedforward can be given a slot, so it switches together with the [PID gains in that slot](closed-loop-control.md#slots).

<figure><img src="../../../.gitbook/assets/smc-slots.png" alt="Four slots, SLOT_0 to SLOT_3, each holding its own PID gains and feedforward, for example one for holding nothing, one for holding a game piece and one for climbing. SLOT_0 is in use. SLOT_3 is not available on TalonFX or TalonFXS. motor.setClosedLoopSlot switches slot at once."><figcaption><p>Each slot holds a feedforward alongside its PID gains, and both switch together.</p></figcaption></figure>

```java
.withFeedforward(new ArmFeedforward(0.1, 0.35, 1.2), ClosedLoopControllerSlot.SLOT_0)
.withFeedforward(new ArmFeedforward(0.1, 0.60, 1.2), ClosedLoopControllerSlot.SLOT_1) // holding a game piece
```

**Supported on:** every supported motor controller.


## Code Reference

* [`SmartMotorControllerConfig`](https://yet-another-software-suite.github.io/YAMS/javadocs/yams/motorcontrollers/SmartMotorControllerConfig.html)
