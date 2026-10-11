---
icon: fan
---

# FlyWheelConfig

A flywheel is anything that spins to a speed: shooters, rollers, intakes, feeders and indexers. The [Shooters](../shooters.md) page shows a complete flywheel.

`FlyWheelConfig` has the [shared options](shared-options.md) except hard limits, and two of its own.

## `withDiameter`

The diameter of the wheel.

<figure><img src="../../.gitbook/assets/mech-flywheel-diameter.png" alt="A flywheel with its diameter marked, spinning at 3000 RPM and throwing a game piece from its surface. For a 4 inch wheel the circumference is 0.319 meters, 3000 RPM is 50 rotations per second, and the surface speed is 16 meters per second. getLinearVelocity and getAngularVelocity on the config convert between the two."><figcaption><p>The diameter converts rotation speed to surface speed.</p></figcaption></figure>

```java
FlyWheelConfig config = new FlyWheelConfig().withDiameter(Inches.of(4));

LinearVelocity  surface = config.getLinearVelocity(RPM.of(3000));
AngularVelocity spin    = config.getAngularVelocity(MetersPerSecond.of(16));
```

It is used for the drawing, and to convert between how fast the wheel spins and how fast its surface moves. The conversions throw if no diameter is set.

**Used in:** simulation and on the real robot. The conversions work on the real robot. The drawing is simulation only.

## `withSpeedometerSimulation`

Draws the flywheel's speed as a needle instead of a line that spins with the wheel.

<figure><img src="../../.gitbook/assets/mech-flywheel-speedometer.png" alt="Two Mechanism2d windows. Without the speedometer the flywheel's line spins with the wheel and is a blur at speed. With withSpeedometerSimulation and a maximum of 6000 RPM, the line points at the speed like a dial: straight down at zero, to the left at half and straight up at the maximum. Here it shows 3720 RPM."><figcaption><p>The flywheel's Mechanism2d with and without the speedometer.</p></figcaption></figure>

```java
new FlyWheelConfig().withSpeedometerSimulation(RPM.of(6000)); // the speed at the top of the dial
```

Give it the fastest speed you expect. `disableSpeedometerSimulation()` turns it back off.

**Used in:** simulation only.

## What else a flywheel needs

| For | Use |
| --- | --- |
| How heavy the wheel is in simulation | [`withMomentOfInertia`](../editor/config/simulation.md#withmomentofinertia) on the motor config |
| Holding a speed | [`SimpleMotorFeedforward`](../editor/config/feedforward.md#withfeedforward) with `kV`, plus a small `kP` |
| A smooth spin up | [A velocity profile](../editor/config/motion-profiles.md#a-velocity-profile) |
| A roller that only needs a voltage | [`ControlMode.OPEN_LOOP`](../editor/config/motor-basics.md#withcontrolmode) |
| Coasting when stopped | [`MotorMode.COAST`](../editor/config/motor-basics.md#withzeropower) |


## Code Reference

* `FlyWheelConfig`
* `FlyWheel`
