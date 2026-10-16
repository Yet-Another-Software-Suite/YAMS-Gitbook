---
icon: elevator
---

# ElevatorConfig

An elevator moves a carriage along a line. The [Elevators](../elevators.md) page shows a complete elevator.

`ElevatorConfig` has the [shared options](shared-options.md) and three of its own.

## `withCarriageWeight`

Everything the motor lifts: the carriage, whatever is mounted on it, and a game piece if it carries one.

<figure><img src="../../.gitbook/assets/mech-elevator-carriage.png" alt="Two carriages, one of 8 pounds and one of 20 pounds, and a graph of simulated height. With the same gains the 8 pound carriage reaches the setpoint quickly, and the 20 pound carriage rises more slowly and sags below it."><figcaption><p>The same gains on a light and a heavy carriage.</p></figcaption></figure>

```java
new ElevatorConfig().withCarriageWeight(Pounds.of(16));
```

**Used in:** simulation only. **Required** for an `Elevator` to be created in simulation.

## `withAngle`

The angle the elevator is drawn at. The default is 90°, straight up.

<figure><img src="../../.gitbook/assets/mech-elevator-angle.png" alt="Three elevators. A vertical elevator, the default at 90 degrees, has gravity simulated. A slanted elevator set with withAngle at 50 degrees is drawn slanted but still simulated with full gravity. A horizontal slide, set with an angle of 0 and withHorizontalElevator, has no gravity simulated."><figcaption><p>withAngle sets the drawing. withHorizontalElevator turns gravity off.</p></figcaption></figure>

```java
new ElevatorConfig().withAngle(Degrees.of(50));
```

{% hint style="warning" %}
`withAngle` only changes the drawing. It does not reduce the gravity the simulation applies: a slanted elevator is simulated with full gravity, as if it were vertical.
{% endhint %}

**Used in:** simulation only.

## `withHorizontalElevator`

Turns gravity off in the simulation, for a linear slide that moves level with the ground.

```java
new ElevatorConfig()
    .withAngle(Degrees.of(0))
    .withHorizontalElevator();
```

A slide with no gravity needs no `kG`, so it will want [different simulation gains](../editor/config/simulation.md#withsimclosedloopcontroller-and-withsimfeedforward) from a vertical elevator.

**Used in:** simulation only.

## What else an elevator needs

| For                        | Use                                                                                                                                      |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| The physical stops         | [`withHardLimits`](shared-options.md#withhardlimits) on `ElevatorConfig`                                                                 |
| The limits the motor obeys | [`withSoftLimits`](../editor/config/limits-and-protection.md#withsoftlimits) on the motor config                                         |
| Heights in meters          | [`withDrumRadius` or `withMechanismCircumference`](../editor/config/gearing-and-units.md#withmechanismcircumference) on the motor config |
| A cascading elevator       | [`withCascadingElevatorStages`](../editor/config/gearing-and-units.md#withcascadingelevatorstages)                                       |
| Holding against gravity    | [`ElevatorFeedforward`](../editor/config/feedforward.md#withfeedforward) with `kG`                                                       |
| A constant force spring    | [Elevators with a constant force spring](../live-tuning/#elevators-with-a-constant-force-spring)                                         |

## Code reference

* `ElevatorConfig`
* `Elevator`
