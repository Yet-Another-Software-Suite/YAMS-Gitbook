---
icon: rotate
---

# PivotConfig

A pivot is a rotating mechanism that gravity does not pull on: a turret, or a wrist that is balanced. The [Turrets/Wrists](../turrets-wrists.md) page shows a complete pivot.

<figure><img src="../../.gitbook/assets/mech-pivot-vs-arm.png" alt="Two panels. An arm swings up and down, gravity pulls on it, it needs kG to hold and uses ArmFeedforward. A pivot, seen from above, turns about a vertical axis or is balanced, needs no kG and uses SimpleMotorFeedforward."><figcaption><p>Use an Arm when gravity matters and a Pivot when it does not.</p></figcaption></figure>

`PivotConfig` has only the [shared options](shared-options.md): telemetry, hard limits, color and position. Everything else about a pivot is on its motor config.

```java
new PivotConfig()
    .withHardLimits(Degrees.of(-200), Degrees.of(200)) // wiring stops it spinning forever
    .withTelemetry("Turret", TelemetryVerbosity.HIGH);
```

## What else a pivot needs

| For | Use |
| --- | --- |
| The physical stops | [`withHardLimits`](shared-options.md#withhardlimits). **Required** in simulation. |
| A turret that can spin all the way round | [`withContinuousWrapping`](../editor/config/closed-loop-control.md#withcontinuouswrapping) on the motor config, and no soft limits |
| A turret limited by its wiring | [`withSoftLimits`](../editor/config/limits-and-protection.md#withsoftlimits) on the motor config |
| The pivot's weight in simulation | [`withMomentOfInertia`](../editor/config/simulation.md#withmomentofinertia) on the motor config |
| Feedforward | [`SimpleMotorFeedforward`](../editor/config/feedforward.md#withfeedforward): there is no gravity to hold against |
| A turret geared to turn more than once | [EasyCRT](../easycrt.md) |


## Code Reference

* `PivotConfig`
* `Pivot`
