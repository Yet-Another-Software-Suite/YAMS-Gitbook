---
icon: people-arrows
---

# Followers

Two ways to have more than one motor move one mechanism: **tightly coupled** followers and **loosely coupled** followers.

<figure><img src="../../../.gitbook/assets/smc-followers.png" alt="Two panels. Tightly coupled, withFollowers: a leader and a follower are on one shaft or gearbox and the follower copies the leader&#x27;s output. The motors are mechanically locked, there is one closed loop and one config, the followers are the same vendor as the leader, and voltage and duty cycle reach every motor. Loosely coupled, withLooselyCoupledFollowers: two separate motors joined only by the mechanism get the same setpoint and each runs its own PID. They are not mechanically locked, each has its own closed loop and config, vendors can be mixed, and voltage and duty cycle are not forwarded, so the leader moves alone."><figcaption><p>Tightly coupled followers share a shaft. Loosely coupled followers are separate motors.</p></figcaption></figure>

|                        | Tightly coupled: `withFollowers`                            | Loosely coupled: `withLooselyCoupledFollowers`                                   |
| ---------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------- |
| The motors are         | Mechanically locked on one shaft or gearbox                 | Separate, joined only by the mechanism                                           |
| Closed loop            | One, on the leader. Followers copy its output               | One on each motor, on its own encoder                                            |
| Config                 | Only the leader has one                                     | Every motor has its own                                                          |
| Vendors                | Same vendor as the leader                                   | Can be mixed                                                                     |
| Voltage and duty cycle | Reach every motor                                           | **Not forwarded.** The leader moves alone                                        |
| Use it for             | Two motors on one gearbox, such as an elevator or a shooter | Two actuators that must arrive together, such as a cross bar lifted at both ends |

## `withFollowers`

Tightly coupled followers, for motors on the same shaft or gearbox. Each follower copies the leader's output, so only the leader needs a config.

```java
.withFollowers(Pair.of(new SparkMax(CANPorts.fromBusId(1), 4, MotorType.kBrushless), false),
               Pair.of(new SparkMax(CANPorts.fromBusId(1), 5, MotorType.kBrushless), true))
```

Pass the vendor's motor object, not a `SmartMotorController`. The boolean is whether that follower is inverted relative to the leader: motors facing each other across a gearbox usually are.

**Supported on every motor controller.** Followers must be the same vendor as the leader. A TalonFX and a TalonFXS are both CTRE, so either can follow the other. To mix brands, use [`withLooselyCoupledFollowers`](followers.md#withlooselycoupledfollowers).

| TalonFX                          | TalonFXS                         | SPARK MAX / Flex       | Thrifty Nova          |
| -------------------------------- | -------------------------------- | ---------------------- | --------------------- |
| ✅ TalonFX and TalonFXS followers | ✅ TalonFXS and TalonFX followers | ✅ SPARK followers only | ✅ Nova followers only |

## `withLooselyCoupledFollowers`

Loosely coupled followers, for motors that are not on the same shaft but must arrive at the same place, such as two linear actuators joined by a cross bar. Each follower is a full `SmartMotorController` with its own config and its own closed loop, and is sent the leader's position and velocity setpoints.

```java
SmartMotorControllerConfig config = new SmartMotorControllerConfig(this) /* ... */;
SmartMotorController right = new TalonFXWrapper(rightTalon, DCMotor.getKrakenX60(1), config.clone());
SmartMotorController left  = new SparkWrapper(leftSpark, DCMotor.getNEO(1),
                                              config.withLooselyCoupledFollowers(right));
```

**You can mix brands.** Because each motor closes its own loop, a loosely coupled follower can be any supported motor controller, whatever the leader is. The example above has a SPARK leading a TalonFX.

**Supported on:** every supported motor controller.

{% hint style="danger" %}
Loosely coupled followers are only sent position and velocity setpoints. Voltage and duty cycle requests are **not** forwarded, so driving the leader with a raw voltage moves the leader alone.
{% endhint %}

## Code reference

* [`SmartMotorControllerConfig`](https://yet-another-software-suite.github.io/YAMS/javadocs/yams/motorcontrollers/SmartMotorControllerConfig.html)
