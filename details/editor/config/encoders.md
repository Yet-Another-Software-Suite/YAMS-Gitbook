---
icon: compass-drafting
---

# Encoders

Every motor has a built in encoder. Many mechanisms add an external one that knows where the mechanism is when the robot turns on.

{% hint style="warning" %}
Not every option on this page is supported on every motor controller. Each option has a table showing where it works, and an option a controller cannot do throws at startup. The [config overview](README.md#not-every-option-works-on-every-motor-controller) lists them all together.
{% endhint %}

<figure><img src="../../../.gitbook/assets/smc-encoder-chain.png" alt="A chain from left to right: the rotor with its built in relative encoder, which starts at zero each boot; the gearing set by withGearing; the mechanism; the encoder gearing set by withExternalEncoderGearing; and the external encoder, set by withExternalEncoder, which is often absolute and remembers its position."><figcaption><p>The built in encoder, the external encoder, and the gearing between them.</p></figcaption></figure>

## `withStartingPosition`

What the built in encoder is set to when the robot turns on.

<figure><img src="../../../.gitbook/assets/smc-starting-position.png" alt="Two panels with an arm that really is at 90 degrees when the robot turns on. With no starting position the encoder says 0 degrees and every angle is 90 degrees wrong. With withStartingPosition(Degrees.of(90)) the encoder says 90 degrees and angles match the real arm."><figcaption><p>An arm that starts at 90°, with and without a starting position.</p></figcaption></figure>

```java
.withStartingPosition(Degrees.of(90))
.withStartingPosition(Meters.of(0))    // needs a circumference
```

Without it, wherever the mechanism happens to be at startup is called zero. [`withSimStartingPosition`](simulation.md#withsimstartingposition) sets a different value for simulation only.

**Supported on:** every supported motor controller.

## `withEncoderInverted`

Inverts the built in encoder. Use `withExternalEncoderInverted` for the external one.

<figure><img src="../../../.gitbook/assets/smc-encoder-inverted.png" alt="Two panels. When the encoder reading goes up as the motor output is positive, the PID pushes toward the setpoint. When the encoder reading goes down instead, the PID pushes away from the setpoint and the mechanism runs off."><figcaption><p>What a backwards encoder does to closed loop control.</p></figcaption></figure>

```java
.withExternalEncoderInverted(true)
```

Most built in encoders are fixed to the rotor and follow [`withMotorInverted`](motor-basics.md#withmotorinverted), so they rarely need this.

**Not supported everywhere.** `withEncoderInverted` only works on a TalonFXS and throws on every other controller. `withExternalEncoderInverted` works on every controller that has an external encoder configured.

| TalonFX | TalonFXS | SPARK MAX / Flex | Thrifty Nova |
| --- | --- | --- | --- |
| ❌ | ✅ | ❌ | ❌ |

`withExternalEncoderInverted` throws if no external encoder is configured.

## `withExternalEncoder`

Attaches an external encoder. It must be one the motor controller can read directly.

```java
.withExternalEncoder(spark.getAbsoluteEncoder())
.withExternalEncoder(new CANcoder(5, new CANBus(CANPort.CAN_S0)))
```

The encoders each wrapper accepts are listed in the [feature matrix](../feature-matrix.md#external-encoders). [How do I use absolute encoders on my Arm?](../../../how-to/how-do-i-use-absolute-encoders-on-my-arm.md) has a complete example for each vendor.

### Keep the discontinuity point outside the travel

This applies to **every absolute encoder, whether it is geared or mounted 1:1**.

An absolute encoder only knows where it is within one turn of its own shaft. The place where its reading wraps is the **discontinuity point**. It is clearly marked on an absolute encoder, usually as a white line on the casing and another on the bearing: when the two lines line up, the encoder is at its discontinuity point. That is where it is with a discontinuity point of 1 rotation and no zero offset. A discontinuity point of 0.5 rotations moves it to 180° from the line on the casing. The line itself does not move. A zero offset moves it too: see [Zero offsets and the discontinuity point](#zero-offsets-and-the-discontinuity-point). If the encoder passes it anywhere inside the mechanism's travel, two different mechanism positions give the same reading and YAMS cannot tell them apart.

<figure><img src="../../../.gitbook/assets/smc-encoder-hard-stops.gif" alt="Animation of a disabled arm being pushed by hand. Its 1:1 absolute encoder is drawn with a white line on its casing and one on its bearing. It reads minus 180 to 180 degrees, so its discontinuity point is 180 degrees from the line on the casing. With a hard stop at 150 degrees the arm stops there and YAMS keeps reporting the right angle. With only a soft limit at 150 degrees the arm is pushed past it and past the discontinuity point, where the reading jumps by 360 degrees to minus 155."><figcaption><p>A hard stop keeps the encoder away from its discontinuity point. A soft limit does not, because it does nothing while the robot is disabled.</p></figcaption></figure>

An absolute encoder should only be used when careful installation guarantees that it never reaches its discontinuity point anywhere in the range of motion:

* **Adequate hard stops.** The mechanism must be physically unable to leave the range. [Soft limits](limits-and-protection.md#withsoftlimits) are not enough: they do nothing when the robot is disabled and pushed by hand, and the encoder is read when the robot boots.
* **Less than one encoder turn of travel.** The travel between the hard stops must be less than one rotation of the encoder. A 1:1 encoder on a mechanism that turns all the way round, or more, cannot meet this.
* **The discontinuity point outside the travel.** On a 1:1 encoder, [`withExternalEncoderDiscontinuityPoint(Rotations.of(0.5))`](#withexternalencoderdiscontinuitypoint) should be sufficient. On a geared encoder, also install it so that the encoder never reaches its discontinuity point anywhere between the hard stops, with some margin at both ends. Move the mechanism from one hard stop to the other by hand and watch the white lines to check.

{% hint style="success" %}
**Without gearing, a discontinuity point of 0.5 rotations should be sufficient.** With [`withExternalEncoderDiscontinuityPoint(Rotations.of(1))`](#withexternalencoderdiscontinuitypoint) the reading goes from 0 to 360° and wraps at the line on the casing. With `Rotations.of(0.5)` it goes from -180° to 180° and wraps 180° from the line. The line on the casing does not move, only the discontinuity point does. With a [zero offset](#zero-offsets-and-the-discontinuity-point) set at the mechanism's zero, the discontinuity point is 180° from the mechanism's zero, so a 1:1 mechanism that stays within 180° of its zero never reaches it.
{% endhint %}

<figure><img src="../../../.gitbook/assets/smc-encoder-discontinuity-range.gif" alt="Animation of a 1:1 arm moving between hard stops at minus 40 and 110 degrees, with the two discontinuity point settings. The absolute encoder is drawn with a white line on its casing and one on its bearing, which line up at the arm's zero in both panels. With 1 rotation the discontinuity point is at the line on the casing, inside the travel, so the reading jumps by 360 degrees when the arm goes below zero. With 0.5 rotations the discontinuity point is 180 degrees from the line, where the arm never goes, and the reading follows the arm the whole way."><figcaption><p>With 1 rotation the discontinuity point is at the line on the casing. With 0.5 rotations it is 180° from it, and the line has not moved.</p></figcaption></figure>

[Gearing the encoder](#withexternalencodergearing) is where this needs real care, because the encoder then turns further than the mechanism and the discontinuity point setting alone is no longer enough. [That section](#only-gear-an-absolute-encoder-if-it-cannot-wrap) has its own animation.

**Supported on every motor controller.** Every controller accepts an external encoder, but only ones it can read itself. An encoder from another vendor cannot be used.

| TalonFX | TalonFXS | SPARK MAX / Flex | Thrifty Nova |
| --- | --- | --- | --- |
| ✅ `CANcoder`, `CANdi` | ✅ `CANcoder`, `CANdi` | ✅ its own absolute, alternate or external encoder | ✅ data port absolute or quadrature encoder, Thrifty CAN encoder |

## `withExternalEncoderGearing`

The reduction between the mechanism and the external encoder, for an encoder that is not mounted 1:1 on the mechanism's output. The default is 1:1.

```java
.withExternalEncoderGearing(new MechanismGearing(GearBox.fromStages("1:2")))
.withExternalEncoderGearing(0.5)
```

The number is how many times the encoder turns for one turn of the mechanism: `2` means the encoder turns twice as far as the mechanism. It appears on the right of the diagram at the top of this page. It throws if no external encoder is configured.

### Only gear an absolute encoder if it cannot wrap

Gearing an absolute encoder up gives finer resolution, but it also uses up the encoder's one turn faster, so it reaches its [discontinuity point](#keep-the-discontinuity-point-outside-the-travel) sooner.

<figure><img src="../../../.gitbook/assets/smc-encoder-gearing-wrap.gif" alt="Animation of the same arm moving between hard stops 120 degrees apart, read through two encoder gearings. Each absolute encoder is drawn with a white line on its casing and a white line on its bearing, which line up at the discontinuity point. With a gearing of 2 the encoder turns 240 degrees, the lines never meet, and YAMS always reports the right angle. With a gearing of 4 the encoder needs 480 degrees, the lines meet partway through the travel, and YAMS then reports an angle 90 degrees too low."><figcaption><p>The same arm and hard stops. Both use a discontinuity point of 1 rotation, at the line on the casing. With a gearing of 2 the white lines never meet. With a gearing of 4 they do, and the reported angle is wrong.</p></figcaption></figure>

Encoder gearing can and should be used on an absolute encoder **only** when careful installation guarantees that it never wraps anywhere in the range of motion. The [three conditions for every absolute encoder](#keep-the-discontinuity-point-outside-the-travel) still apply, and the second one gets stricter: the travel between the hard stops, **multiplied by the gearing**, must be less than one rotation. In the animation, 120° x 2 = 240° fits and 120° x 4 = 480° does not.

If you cannot guarantee that, mount the encoder 1:1 on the mechanism's output, where a discontinuity point of 0.5 rotations should be sufficient.

{% hint style="warning" %}
YAMS reports a warning when the gearing makes the encoder turn further than the mechanism. It is safe to ignore **only** when all three conditions are met.
{% endhint %}

**Supported on:** every supported motor controller.

## `withExternalEncoderZeroOffset`

Moves an absolute encoder's zero to the mechanism's zero.

<figure><img src="../../../.gitbook/assets/smc-zero-offset.png" alt="Two dials. The encoder reads 37 degrees when the mechanism is at its zero. Subtracting a zero offset of 37 degrees makes YAMS report 0 degrees. To find it, move the mechanism to where you want zero, read the external encoder, and use that reading as the offset."><figcaption><p>The offset is what the encoder reads when the mechanism is at zero.</p></figcaption></figure>

```java
.withExternalEncoderZeroOffset(Degrees.of(37))
```

A negative offset is wrapped into the positive range for you. The distance version needs a [circumference](gearing-and-units.md#withmechanismcircumference).

**Not supported everywhere.** On a quadrature encoder it throws.

| TalonFX | TalonFXS | SPARK MAX / Flex | Thrifty Nova |
| --- | --- | --- | --- |
| ✅ | ✅ | ⚠️ absolute encoders only | ⚠️ absolute and CAN encoders only |

### Zero offsets and the discontinuity point

The zero offset and the discontinuity point work together, and the order matters: **the offset is applied first, and the range is applied to the result**. So the discontinuity point is measured from the encoder's zero, not from the line on the casing.

<figure><img src="../../../.gitbook/assets/smc-encoder-zero-offset-discontinuity.gif" alt="Animation of the same arm and 1:1 absolute encoder, reading minus 180 to 180 degrees, with the encoder installed a quarter turn round. With no zero offset the encoder reads zero where the white lines line up and its discontinuity point is opposite that, inside the arm's travel, so YAMS reports an angle 90 degrees off and then jumps by 360 degrees. With a zero offset of 90 degrees, the place where the encoder reads zero and the discontinuity point have both moved a quarter turn, the discontinuity point is outside the travel, and YAMS reports the right angle. The white line on the casing has not moved."><figcaption><p>The zero offset moves the zero, and the discontinuity point moves with it. The line on the casing stays where it is.</p></figcaption></figure>

| Zero offset | Discontinuity point of 1 rotation | Discontinuity point of 0.5 rotations |
| --- | --- | --- |
| None | Wraps where the white lines line up | Wraps 180° from where the white lines line up |
| Set at the mechanism's zero | Wraps at the mechanism's zero | Wraps 180° from the mechanism's zero |

What this means in practice:

* **The white lines tell you where the wrap is only when there is no zero offset.** Once an offset is set, work from the mechanism's zero instead.
* **Set the zero offset first, then choose the discontinuity point.** With the offset at the mechanism's zero and a discontinuity point of 0.5 rotations, a 1:1 encoder wraps as far from zero as it can, however it was installed.
* **A geared encoder is still limited to one turn.** The same rule applies to the encoder's own shaft: 180° of the encoder is `180° / gearing` of the mechanism. See [Only gear an absolute encoder if it cannot wrap](#only-gear-an-absolute-encoder-if-it-cannot-wrap).

This comes from the vendors' own documentation, which YAMS passes these two settings straight to:

* **CTRE** ([`MagnetSensorConfigs`](https://api.ctr-electronics.com/phoenix6/stable/java/com/ctre/phoenix6/configs/MagnetSensorConfigs.html)): the magnet offset "is added to the reported position", and the discontinuity point "determines the point at which the absolute sensor wraps around, keeping the absolute position (after offset) in the range [x-1, x)".
* **REV** ([`AbsoluteEncoderConfig`](https://codedocs.revrobotics.com/java/com/revrobotics/spark/config/absoluteencoderconfig)): the zero offset is "the position that is reported as zero", and with zero centering "the position will be reported in the range (-0.5, 0.5], instead of the default range [0, 1)". REV does not spell out the order, but a range centered on the reported zero means the wrap is half a turn from the offset zero.

## `withExternalEncoderDiscontinuityPoint`

Chooses where an absolute encoder's reading wraps.

<figure><img src="../../../.gitbook/assets/smc-discontinuity-point.png" alt="Two number lines. With a discontinuity point of 1 rotation the encoder reads 0 to 360 degrees, for mechanisms that only use positive angles. With 0.5 rotations it reads minus 180 to 180 degrees, for mechanisms that move either side of zero. The reading jumps from one end to the other at the ends of the range."><figcaption><p>The two ranges an absolute encoder can report.</p></figcaption></figure>

```java
.withExternalEncoderDiscontinuityPoint(Rotations.of(0.5)) // reads -180° to 180°
.withExternalEncoderDiscontinuityPoint(Rotations.of(1))   // reads 0° to 360°
```

Only `0.5` and `1` rotations are accepted. Like the zero offset, it is for absolute encoders only.

**Not supported everywhere.** On a quadrature encoder it throws.

| TalonFX | TalonFXS | SPARK MAX / Flex | Thrifty Nova |
| --- | --- | --- | --- |
| ✅ | ✅ | ⚠️ absolute encoders only | ⚠️ absolute and CAN encoders only |

## `withUseExternalFeedbackEncoder`

Chooses which encoder the closed loop controller follows. The default is `true`: the external encoder, when there is one.

<figure><img src="../../../.gitbook/assets/smc-external-feedback.png" alt="Two panels. With the option on, the default, the PID follows the external encoder. With it off, the PID follows the built in encoder and the external encoder only sets the built in encoder's starting position."><figcaption><p>The external encoder as the feedback sensor, or only as the starting point.</p></figcaption></figure>

```java
.withUseExternalFeedbackEncoder(false)
```

## `withFeedbackSynchronizationThreshold`



When the loop follows the built in encoder, re-seeds it from the external encoder whenever the two disagree by more than this.

<figure><img src="../../../.gitbook/assets/smc-feedback-sync.png" alt="A graph of position against time. The built in encoder slowly drifts away from the external absolute encoder. When the difference passes the threshold the built in encoder is re-seeded and the two lines meet again."><figcaption><p>The built in encoder is re-seeded when it drifts past the threshold.</p></figcaption></figure>

```java
.withUseExternalFeedbackEncoder(false)
.withFeedbackSynchronizationThreshold(Degrees.of(2))
```

It is not available for distance mechanisms: it throws if a circumference is set.

**Not supported everywhere.** It throws where it is not supported.

| TalonFX | TalonFXS | SPARK MAX / Flex | Thrifty Nova |
| --- | --- | --- | --- |
| ❌ | ❌ | ⚠️ only with an exponential profile or an LQR controller | ⚠️ only in closed loop mode |


## Code Reference

* [`SmartMotorControllerConfig`](https://yet-another-software-suite.github.io/YAMS/javadocs/yams/motorcontrollers/SmartMotorControllerConfig.html)
