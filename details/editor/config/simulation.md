---
icon: flask
---

# Simulation

These options only change what happens in simulation. The real robot ignores them.

Every option on this page is supported on every supported motor controller.

## `withSimClosedLoopController` and `withSimFeedforward`

A second set of gains used only in simulation. They take the same arguments, including [slots](closed-loop-control.md#slots), as [`withClosedLoopController`](closed-loop-control.md#withclosedloopcontroller) and [`withFeedforward`](feedforward.md#withfeedforward).

<figure><img src="../../../.gitbook/assets/smc-sim-gains.png" alt="A diagram. On the real robot the closed loop controller uses the gains from withClosedLoopController and withFeedforward to drive the real mechanism, which has friction, backlash and flex. In simulation it uses the gains from withSimClosedLoopController and withSimFeedforward to drive a simulated mechanism, an ideal motor and load."><figcaption><p>Real gains for the real mechanism, simulation gains for the simulated one.</p></figcaption></figure>

```java
.withClosedLoopController(4, 0, 0.2)
.withSimClosedLoopController(12, 0, 0)
.withFeedforward(new ArmFeedforward(0.15, 0.35, 1.2))
.withSimFeedforward(new ArmFeedforward(0, 0.3, 1.2))
```

[Simulation-only PID + feedforward](../simulation-only-pid-+-feedforward.md) explains why the two usually differ.

## `withSimStartingPosition`

Where the simulated mechanism starts, when that should differ from the real [starting position](encoders.md#withstartingposition).

```java
.withSimStartingPosition(Degrees.of(45))
```

It is most useful with an absolute encoder: the real robot reads its position from the encoder, and simulation needs to be told one.

## `withMomentOfInertia`

How hard the simulated load is to spin up. It is only needed when a `SmartMotorController` is simulated on its own. A mechanism class works it out from the mechanism's length and mass.

```java
.withMomentOfInertia(KilogramSquareMeters.of(0.02))
.withMomentOfInertia(Inches.of(4), Pounds.of(1)) // a rod of this length and weight
```

## `withSimulationPeriod`

How often the simulated physics are stepped. The default is 20 ms, and it is independent of [`withClosedLoopControlPeriod`](closed-loop-control.md#withclosedloopcontrolperiod).

```java
.withSimulationPeriod(Milliseconds.of(5))
```

Each `simIterate()` call steps the physics by exactly this period, so it must match how often `simIterate()` is called. To step every 1 ms, as fast as a closed loop on the motor controller runs, call it from a `Notifier`: see [How do I run a high fidelity simulation?](../../../how-to/how-do-i-run-a-high-fidelity-simulation.md)

<figure><img src="../../../.gitbook/assets/smc-simulation.png" alt="Two graphs. Simulated speed: a small moment of inertia spins up quickly and a large one more slowly, set by withMomentOfInertia. Simulated position: 20 millisecond steps, the default, make a coarse staircase and 5 millisecond steps make a finer curve, set by withSimulationPeriod."><figcaption><p>Moment of inertia and simulation period.</p></figcaption></figure>

## Running the simulation

A mechanism class steps its motor's simulation for you. A `SmartMotorController` used without one needs `simIterate()` called from `simulationPeriodic()`: see [simulation without a mechanism](../README.md#simulation-without-a-mechanism). Every simulated motor also draws from the shared [battery simulation](../../../understanding/battery-simulation.md).


## Code Reference

* [`SmartMotorControllerConfig`](https://yet-another-software-suite.github.io/YAMS/javadocs/yams/motorcontrollers/SmartMotorControllerConfig.html)
