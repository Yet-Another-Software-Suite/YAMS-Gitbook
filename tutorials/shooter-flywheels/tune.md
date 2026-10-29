---
description: Tune the flywheel's gains while the simulation runs, then copy them into your code.
---

# Step 5: Tune it

The gains from step 1 are placeholders worked out from the motor: `kV` is 0.19 and `kP` is 0.2. On this page you find your own, while the simulation runs, the same way you will tune the real flywheel on the robot. You change one number at a time on a dashboard and watch a graph. Nothing is saved until you copy the numbers into your code at the end.

A flywheel is tuned differently from [an arm](../arm/tune.md) or an elevator. There is no gravity to hold up and no motion profile. `kV` does most of the work, so you find it first, with `kP` turned off, then add a little `kP` to fix what is left.

{% hint style="info" %}
**In simulation you are tuning the simulation's gains.** While the code runs in simulation, live tuning changes the `withSimClosedLoopController` and `withSimFeedforward` values. On the real robot it changes `withClosedLoopController` and `withFeedforward`. The two sets are kept apart because a simulated flywheel is never exactly like the real one.
{% endhint %}

## Before you start

* Start the simulation and enable **Teleoperated**, as in [step 4](simulate.md). The wheel is still.
* Open [AdvantageScope](https://docs.advantagescope.org/) and connect it to the simulation (**File > Connect to Simulator**).
* Do not press any of the shooter's buttons while you tune. A button runs another shooter command, which stops live tuning. If that happens, start it again as in part 1.

## 1. Start live tuning

YAMS gives the flywheel a `Live Tuning` command. While it runs, numbers you type on the dashboard are applied to the flywheel straight away.

{% tabs %}
{% tab title="Java, Commands v2" %}
The command is published at `NT:/Tuning/ShooterSubsystem/Live Tuning`, named after the subsystem.

In [Elastic](https://frc-elastic.gitbook.io/docs), add `Tuning/ShooterSubsystem/Live Tuning` to a tab and press its button.
{% endtab %}

{% tab title="Java, Commands v3" %}
The command is published at `NT:/Tuning/ShooterMechanism/Live Tuning`, named after the mechanism.

Set the boolean `NT:/Tuning/ShooterMechanism/Live Tuning/running` to `true`. In AdvantageScope, first turn on tuning mode (part 2), then click the value to change it.
{% endtab %}

{% tab title="C++" %}
The command is published at `NT:/Tuning/ShooterSubsystem/Live Tuning`, named after the subsystem.

In [Elastic](https://frc-elastic.gitbook.io/docs), add `Tuning/ShooterSubsystem/Live Tuning` to a tab and press its button.
{% endtab %}
{% endtabs %}

**Check it:** the command shows as running. The wheel is still.

## 2. Find the numbers and graph the flywheel

The flywheel's values are under two tables. Both use the names from `withTelemetry`: `Shooter` for the flywheel and `ShooterMotor` for its motor.

| | Where | Used for |
| --- | --- | --- |
| Values you type | `NT:/Tuning/Shooter/ShooterMotor/` | Gains, setpoints, limits |
| Values you watch | `NT:/Mechanisms/Shooter/ShooterMotor/` | How fast the wheel spins and what the motor is doing |

1. In AdvantageScope, turn on **tuning mode**: the sliders icon beside the search box at the top of the sidebar. Without it, typed values are ignored.
2. Make a **Line Graph** and drag in, from `NT:/Mechanisms/Shooter/ShooterMotor/`:
   * `mechanism/velocity`: how fast the wheel spins.
   * `closedloop/setpoint/velocity`: how fast it is asked to spin.
   * `motor/outputVoltage`: the voltage the motor is given.

{% hint style="warning" %}
**You type RPM, you read rotations per second.** A setpoint typed as `3000` (RPM) is graphed as `50` (rotations per second). Divide RPM by 60 to compare. To graph in RPM, see [See both in the same units with AdvantageScope](../../details/live-tuning/keys-and-units.md#see-both-in-the-same-units-with-advantagescope).
{% endhint %}

**Check it:** the velocity lines are at 0.

## 3. kV: the voltage for each speed

`kV` is how many volts the motor needs for each rotation per second of wheel speed. Get it right and the wheel reaches the setpoint on feedforward alone.

1. Type `0` into `NT:/Tuning/Shooter/ShooterMotor/closedloop/feedback/kP` and `0` into `closedloop/feedforward/kV`. Now nothing drives the wheel.
2. Type `3000` into `closedloop/setpoint/velocity`. The setpoint line jumps to 50, and the wheel stays still.
3. Type `0.1` into `closedloop/feedforward/kV`. The wheel spins up, but settles well below 50.
4. Raise `kV` a little at a time until `mechanism/velocity` settles on the setpoint line. For this NEO and 1.5:1 gearing it ends up near `0.19`. If the wheel settles above the setpoint, `kV` is too high: lower it.
5. Type `2000` into the setpoint (33 on the graph). The wheel should settle on it too, with the same `kV`.

**Check it:** at both 2000 and 3000 RPM the measured speed settles on the setpoint, with `kP` still at 0.

`kS` is the voltage that just overcomes friction. The simulation has no friction, so leave it at 0 here. On the real robot, raise `closedloop/feedforward/kS` until the wheel only just starts to turn, then back it off a little, before you tune `kV`.

## 4. kP: recover faster

`kV` gets the wheel to speed. `kP` adds a push when the wheel is slower or faster than the setpoint, such as when a game piece goes through and drags it down. It is volts for each rotation per second of error.

1. Type `0.1` into `closedloop/feedback/kP`.
2. Move the setpoint between `2000` and `3000` and watch how fast the speed gets there.
3. Raise `kP` a step at a time (0.2, 0.3, ...). Each step should make the wheel reach the new speed a little faster.
4. Stop, and go back one step, when the speed overshoots the setpoint or the line gets jittery.

<figure><img src="../../.gitbook/assets/howto-shooter-at-speed.gif" alt="Animation of a flywheel spinning up to 3000 RPM and shooting two balls. Each shot drops the speed by several hundred RPM, and it overshoots and rings as it recovers. The feeder only runs once the speed has stayed inside a band around the setpoint."><figcaption><p>On the real robot each shot slows the wheel. A good kP brings it back quickly without ringing.</p></figcaption></figure>

**Check it:** the speed reaches each new setpoint quickly and settles on it without bouncing around it.

A flywheel needs no `kD` and no motion profile: the wheel may speed up as fast as the motor allows.

For more on each value, see [Tuning order](../../details/live-tuning/README.md#tuning-order).

## 5. Copy the numbers into your code

Dashboard changes are lost when the code restarts. Write the numbers you ended up with into the config from step 1. In simulation they go into the `Sim` versions.

{% tabs %}
{% tab title="Java, Commands v2" %}
{% code title="subsystems/ShooterSubsystem.java" %}
```java
// Before
.withSimClosedLoopController(0.2, 0, 0)
.withSimFeedforward(new SimpleMotorFeedforward(0, 0.19, 0))

// After: your numbers. SimpleMotorFeedforward takes kS, kV, kA.
.withSimClosedLoopController(0.3, 0, 0)
.withSimFeedforward(new SimpleMotorFeedforward(0, 0.188, 0))
```
{% endcode %}
{% endtab %}

{% tab title="Java, Commands v3" %}
{% code title="mechanisms/ShooterMechanism.java" %}
```java
// Before
.withSimClosedLoopController(0.2, 0, 0)
.withSimFeedforward(new SimpleMotorFeedforward(0, 0.19, 0))

// After: your numbers. SimpleMotorFeedforward takes kS, kV, kA.
.withSimClosedLoopController(0.3, 0, 0)
.withSimFeedforward(new SimpleMotorFeedforward(0, 0.188, 0))
```
{% endcode %}
{% endtab %}

{% tab title="C++" %}
{% code title="cpp/subsystems/ShooterSubsystem.cpp" %}
```cpp
// Before
.WithSimClosedLoopController(0.2, 0, 0)
.WithSimFeedforward(Feedforward{0_V, wpi::units::unit_t<Feedforward::kv_unit>{0.19}})

// After: your numbers. kP, kI, kD, then kS and kV.
.WithSimClosedLoopController(0.3, 0, 0)
.WithSimFeedforward(Feedforward{0_V, wpi::units::unit_t<Feedforward::kv_unit>{0.188}})
```
{% endcode %}
{% endtab %}
{% endtabs %}

The numbers above are an example. Use the ones you found.

**Check it:** restart the simulation and enable it. Hold **A** and **B**: the wheel reaches 2000 and 3000 RPM quickly and holds each one.

## If it did not work

* **Typing a value does nothing.** Turn on AdvantageScope's tuning mode, check the `Live Tuning` command is still running, and check you typed under `NT:/Tuning`, not `NT:/Mechanisms`.
* **Typing `0` as the setpoint does not stop the wheel.** Live tuning ignores a velocity setpoint of 0. To stop the wheel, disable the robot, or press a shooter button (which also stops live tuning).
* **Live tuning stopped on its own.** A button press, or the default command, took over the flywheel. Start it again.
* **The speed swings above and below the setpoint.** `kP` is too high. Lower it.
* **The numbers were gone after a restart.** They only last until the code stops. Copy them into your code as in part 5.

More problems and fixes: [Troubleshooting](../troubleshooting.md).

## Next

Step 6 puts the code on the real robot. Follow [the arm's step 6: Run it on the robot](../arm/robot.md). It works the same for the flywheel, with these differences:

* The CAN ID is the shooter's `SparkMax`, 4 in this tutorial.
* A flywheel has no starting position and no soft limits, so skip those parts.
* For the direction check, positive should spin the wheel the way that throws a game piece **out**.
* On the first enable, hold **A** (2000 RPM) before you try **B**.
* Tune `kS`, then `kV`, then `kP` again on the robot, as in parts 3 and 4. Friction means a real wheel needs some `kS`. Copy the robot's numbers into `withClosedLoopController` and `withFeedforward`.
