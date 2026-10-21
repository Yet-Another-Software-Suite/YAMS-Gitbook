---
icon: sliders
---

# YAMS Live Tuner

The [YAMS Live Tuner](https://tuning.yassrobotics.com/) is a web page that tunes an elevator, arm, pivot or flywheel through [live tuning](README.md). It connects to the robot or simulation over NetworkTables, finds every YAMS motor controller with live tuning, and walks you through checking it before anything moves: a safety checklist, the motor inversion, and the range of motion. Then you send setpoints and change gains while watching a graph.

It writes the same `NT:/Tuning` keys you would edit in AdvantageScope or Elastic, so everything on the [Live tuning](README.md) page still applies.

{% hint style="warning" %}
The Live Tuner drives the motor at a voltage while it checks the inversion and the range of motion. That needs a YAMS release newer than 2026.10.19, which adds `voltage/enabled` and `voltage/setpoint` to live tuning, and registers the `Live Tuning` command for mechanisms such as `Elevator` and `Arm`.
{% endhint %}

The screenshots below come from the YAMS Commands v3 examples running in simulation, with each mechanism's simulation in its own `Notifier`: the BroncBotz 2025 elevator and coral arm, and the BroncBotz 2026 turret and kicker. Each motor is set up with `withTelemetry(name, TelemetryVerbosity.HIGH)` and its mechanism with `withTelemetry(...)` too, which is all live tuning needs.

{% hint style="danger" %}
Like all tuning, this can be **dangerous** on a real robot. Practice in simulation first. Press **Space** anywhere on the page to stop Live Tuning straight away, and keep a hand on the Driver Station's disable.
{% endhint %}

## Connect

Open [tuning.yassrobotics.com](https://tuning.yassrobotics.com/). Pick where the robot code is running, then press **Connect**:

* **Robot (team)**: the robot, from your team number.
* **Simulation**: robot code running in simulation on this computer.
* **Custom host**: any other address.

Once connected, the badge shows the host and the robot's mode. Enable the robot, in Teleop or Test, from the Driver Station before anything moves.

<figure><img src="../../.gitbook/assets/live-tuner-connect.png" alt="The Live Tuner header connected to a simulation on localhost, with the robot in Teleop and enabled"><figcaption><p>Connected to a simulation, with the robot enabled in Teleop.</p></figcaption></figure>

{% hint style="info" %}
No robot code to hand? Open [tuning.yassrobotics.com/?demo](https://tuning.yassrobotics.com/?demo) to try the Live Tuner on a robot simulated in the page.
{% endhint %}

{% hint style="info" %}
If your browser asks whether the page may connect to devices on your local network, allow it. The robot and the simulation are on your local network.
{% endhint %}

## Tune a mechanism

The Live Tuner's numbered steps unlock one after another. Each one has to be finished, or skipped where that is allowed, before the next. Pick the tab for your kind of mechanism.

{% tabs %}
{% tab title="Elevators" %}
{% stepper %}
{% step %}
#### Pick the mechanism

Choose the motor controller under **Telemetry name**, and the subsystem (Commands v2) or mechanism (Commands v3) whose `Live Tuning` command it belongs to. The lists are filled in from what the robot publishes under `NT:/Tuning`, so a motor controller only shows up once live tuning is [turned on](README.md#turn-it-on) for it.

Set **Type** to **Elevator**. Setpoints are in meters.

<figure><img src="../../.gitbook/assets/live-tuner-elevator-mechanism.png" alt="Step 1 with Elevator/ElevatorMotor as the telemetry name, Elevator as the Live Tuning command and Elevator as the type, so setpoints are in meters"><figcaption><p>The BroncBotz 2025 elevator: its motor, its mechanism's <code>Live Tuning</code> command, and the Elevator type.</p></figcaption></figure>

Changing anything in this step resets every confirmation below it.
{% endstep %}

{% step %}
#### Go through the pre-motion checklist

Nothing moves until every item is ticked. **Current trip** stops Live Tuning if the stator current stays above it for 250 ms, which catches a stalled or jammed mechanism.

<figure><img src="../../.gitbook/assets/live-tuner-elevator-checklist.png" alt="The pre-motion checklist with all four items ticked, and the current trip set to 40 A"><figcaption><p>Every item confirmed, with a 40 A current trip.</p></figcaption></figure>
{% endstep %}

{% step %}
#### Check the inversions

This step drives the motor at a small voltage, with no closed loop, to check that a positive voltage moves the mechanism in its positive direction and that the position reading goes the same way.

Read the warning, tick the confirmation, and press **Start voltage control**. If the inversions were already checked on this robot, **Skip (already checked)** goes straight to the next step.

<figure><img src="../../.gitbook/assets/live-tuner-elevator-inversions-confirm.png" alt="Step 3 before it starts: a warning about driving open loop and followers fighting each other, a confirmation checkbox, and the Start voltage control and Skip buttons"><figcaption><p>Voltage control only starts once you confirm you are watching the mechanism.</p></figcaption></figure>

Drag the slider, or type a voltage and press **Enter** or **Apply**. The slider goes back to 0 V when you let go; a typed voltage holds until you change it. Start with a fraction of a volt and raise it until the mechanism moves. Gravity pulls an elevator down, so it can need more voltage to move up than down.

<figure><img src="../../.gitbook/assets/live-tuner-elevator-inversions-up.png" alt="Holding 1 V: the output reads 1.00 V and Positive voltage moves up is ticked"><figcaption><p>At +1 V the position rises, so <strong>Positive voltage moves up</strong> is ticked.</p></figcaption></figure>

Then drive it the other way. Once both directions are ticked, press **Set 0 V** and **Finish check**.

<figure><img src="../../.gitbook/assets/live-tuner-elevator-inversions-done.png" alt="Both Positive voltage moves up and Negative voltage moves down ticked, and Finish check enabled"><figcaption><p>Both directions checked.</p></figcaption></figure>

If the position goes the wrong way, stop and fix `withMotorInverted(...)`, or the encoder inversion, in the robot code before going on.
{% endstep %}

{% step %}
#### Verify the range of motion

The Live Tuner only sends setpoints inside a range that the mechanism has actually been driven across. There are two ways to set it:

* **Find ends by driving**: hold the drive buttons to drive to each physical end of travel, then mark each end.
* **Enter range and test it**: type the lower and upper ends, then hold each drive button until the mechanism gets there. It starts from the soft limits the robot publishes.

Press **Start verification**, set **Drive voltage**, and hold a drive button. Voltage is only applied while you hold it, and it is halved close to each end to limit overshoot. A stall, or the current trip, on the way marks that end where the mechanism stopped. While it drives, the banner at the top reads **VERIFYING RANGE**, and Space stops it.

Once both ends are marked, press **Finish verification**. Live Tuning stops if the mechanism leaves the verified range.

<figure><img src="../../.gitbook/assets/live-tuner-elevator-range-verified.png" alt="Step 4 verified from 0.000 m to 0.250 m, after reaching each end of the entered range"><figcaption><p>Verified from 0 m to 0.25 m with <strong>Enter range and test it</strong>.</p></figcaption></figure>
{% endstep %}

{% step %}
#### Live tune

Press **Start Live Tuning**. This starts the mechanism's `Live Tuning` command, so the setpoints and gains you send are applied every loop.

Send a setpoint by typing it, with the slider, or with **Lower**, **25%**, **Mid**, **75%** and **Upper**. The graph shows the measured position against the setpoint, with the stator current and the verified range.

<figure><img src="../../.gitbook/assets/live-tuner-elevator.gif" alt="The elevator following Mid and 25% setpoints, first with kP cut to a fifth, where it lags, then with kP put back, where it follows them"><figcaption><p>kP cut to a fifth, then put back: the elevator lags each setpoint, then follows it.</p></figcaption></figure>

**Change the gains**

The **Gains** card shows the values the robot publishes under `NT:/Tuning`. Type a new value and press **Enter** to apply it. [What to tune and what to watch](README.md#what-to-tune-and-what-to-watch) lists what each gain does. On an elevator, start with kG: the output while it holds still is the voltage it takes to hold the carriage up against gravity. Then raise kP until it reaches each setpoint without overshooting.

<figure><img src="../../.gitbook/assets/live-tuner-elevator-gains.png" alt="The Gains card: kP 33.966, kI 0, kD 0; kS 0.21471 V, kV 10.39, kA 0, kG 0.23861 V; max velocity 1 m/s, max acceleration 0.5 m/s²; stator limit 40 A and closed-loop ramp 0 s"><figcaption><p>The elevator's gains, motion profile and limits.</p></figcaption></figure>

**Alternate between two setpoints**

Turn on **Alternate setpoints** to switch between **Setpoint A** and **Setpoint B** every 5 seconds. Each step is the same, so you can change a gain and compare the next step with the last one.

<figure><img src="../../.gitbook/assets/live-tuner-elevator-alternate.png" alt="Alternate setpoints on, switching between 0.063 m and 0.188 m: the graph shows the elevator stepping down and back up"><figcaption><p>Stepping between 0.063 m and 0.188 m.</p></figcaption></figure>
{% endstep %}

{% endstepper %}
{% endtab %}

{% tab title="Arms and pivots" %}
{% stepper %}
{% step %}
#### Pick the mechanism

Choose the motor controller under **Telemetry name**, and the subsystem (Commands v2) or mechanism (Commands v3) whose `Live Tuning` command it belongs to. The lists are filled in from what the robot publishes under `NT:/Tuning`, so a motor controller only shows up once live tuning is [turned on](README.md#turn-it-on) for it.

Set **Type** to **Arm** for an arm that gravity pulls on, or **Pivot** for anything else that turns, such as a turret. Setpoints are in degrees.

<figure><img src="../../.gitbook/assets/live-tuner-arm-mechanism.png" alt="Step 1 with CoralArm/CoralArmMotor as the telemetry name, CoralArm as the Live Tuning command and Arm as the type"><figcaption><p>The BroncBotz 2025 coral arm, as an Arm.</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/live-tuner-pivot-mechanism.png" alt="Step 1 with Turret/TurretMotor as the telemetry name, Turret as the Live Tuning command and Pivot as the type"><figcaption><p>The BroncBotz 2026 turret, as a Pivot.</p></figcaption></figure>

For a pivot, tick **Continuous (no range of motion)** only if it can spin freely forever without wrapping wires.

Changing anything in this step resets every confirmation below it.
{% endstep %}

{% step %}
#### Go through the pre-motion checklist

Nothing moves until every item is ticked. **Current trip** stops Live Tuning if the stator current stays above it for 250 ms, which catches a stalled or jammed mechanism.

<figure><img src="../../.gitbook/assets/live-tuner-arm-checklist.png" alt="The pre-motion checklist with every item ticked, and the current trip set"><figcaption><p>Every item confirmed.</p></figcaption></figure>
{% endstep %}

{% step %}
#### Check the inversions

This step drives the motor at a small voltage, with no closed loop, to check that a positive voltage moves the mechanism in its positive direction and that the position reading goes the same way.

Read the warning, tick the confirmation, and press **Start voltage control**. If the inversions were already checked on this robot, **Skip (already checked)** goes straight to the next step.

Drag the slider, or type a voltage and press **Enter** or **Apply**. The slider goes back to 0 V when you let go; a typed voltage holds until you change it. Start with a fraction of a volt and raise it until the mechanism moves. Near horizontal, gravity can be stronger than a small voltage on an arm, so try a little more voltage before changing an inversion.

<figure><img src="../../.gitbook/assets/live-tuner-arm-inversions-up.png" alt="Holding 1 V on the coral arm: the output reads 1.00 V and Positive voltage moves forward is ticked"><figcaption><p>At +1 V the angle rises, so <strong>Positive voltage moves forward</strong> is ticked.</p></figcaption></figure>

Then drive it the other way. Once both directions are ticked, press **Set 0 V** and **Finish check**.

<figure><img src="../../.gitbook/assets/live-tuner-arm-inversions-done.png" alt="Both Positive voltage moves forward and Negative voltage moves reverse ticked, and Finish check enabled"><figcaption><p>Both directions checked.</p></figcaption></figure>

If the position goes the wrong way, stop and fix `withMotorInverted(...)`, or the encoder inversion, in the robot code before going on.
{% endstep %}

{% step %}
#### Verify the range of motion

The Live Tuner only sends setpoints inside a range that the mechanism has actually been driven across. There are two ways to set it:

* **Find ends by driving**: hold the drive buttons to drive to each physical end of travel, then mark each end.
* **Enter range and test it**: type the lower and upper ends, then hold each drive button until the mechanism gets there. It starts from the soft limits the robot publishes.

Press **Start verification**, set **Drive voltage**, and hold a drive button. Voltage is only applied while you hold it, and it is halved close to each end to limit overshoot. A stall, or the current trip, on the way marks that end where the mechanism stopped. While it drives, the banner at the top reads **VERIFYING RANGE**, and Space stops it.

Once both ends are marked, press **Finish verification**. Live Tuning stops if the mechanism leaves the verified range. A continuous pivot has no end of travel, so it has no range to verify.

<figure><img src="../../.gitbook/assets/live-tuner-arm-range-verified.png" alt="Step 4 verified from -90° to 45° on the coral arm"><figcaption><p>The coral arm verified from -90° to 45°.</p></figcaption></figure>
{% endstep %}

{% step %}
#### Live tune

Press **Start Live Tuning**. This starts the mechanism's `Live Tuning` command, so the setpoints and gains you send are applied every loop.

Send a setpoint by typing it, with the slider, or with **Lower**, **25%**, **Mid**, **75%** and **Upper**. The graph shows the measured angle against the setpoint, with the stator current and the verified range.

<figure><img src="../../.gitbook/assets/live-tuner-arm.gif" alt="The coral arm with kP cut to a fifth sagging below its setpoints, then with kP put back following them"><figcaption><p>The coral arm. With kP at a fifth of its value, gravity holds it below its setpoint; with kP put back, it follows.</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/live-tuner-pivot.gif" alt="The turret with kP cut to a fifth creeping toward its setpoints, then with kP put back reaching them"><figcaption><p>The turret, a pivot. kP starts at a fifth of its value, then goes back.</p></figcaption></figure>

**Change the gains**

The **Gains** card shows the values the robot publishes under `NT:/Tuning`. Type a new value and press **Enter** to apply it. [What to tune and what to watch](README.md#what-to-tune-and-what-to-watch) lists what each gain does. On an arm, start with kG: raise it until the arm holds still when it is horizontal. A pivot has no kG. Then raise kP until the mechanism reaches each setpoint without overshooting.

<figure><img src="../../.gitbook/assets/live-tuner-arm-gains.png" alt="The coral arm's Gains card with its feedback, feedforward, motion profile and limits"><figcaption><p>The coral arm's gains.</p></figcaption></figure>

**Alternate between two setpoints**

Turn on **Alternate setpoints** to switch between **Setpoint A** and **Setpoint B** every 5 seconds. Each step is the same, so you can change a gain and compare the next step with the last one.

<figure><img src="../../.gitbook/assets/live-tuner-arm-alternate.png" alt="Alternate setpoints on: the graph shows the coral arm stepping between two angles"><figcaption><p>The coral arm stepping between two angles.</p></figcaption></figure>
{% endstep %}

{% endstepper %}
{% endtab %}

{% tab title="Flywheels" %}
{% stepper %}
{% step %}
#### Pick the mechanism

Choose the motor controller under **Telemetry name**, and the subsystem (Commands v2) or mechanism (Commands v3) whose `Live Tuning` command it belongs to. The lists are filled in from what the robot publishes under `NT:/Tuning`, so a motor controller only shows up once live tuning is [turned on](README.md#turn-it-on) for it.

Set **Type** to **Flywheel**. Setpoints are in RPM.

<figure><img src="../../.gitbook/assets/live-tuner-flywheel-mechanism.png" alt="Step 1 with Kicker/KickerMotor as the telemetry name, Kicker as the Live Tuning command and Flywheel as the type"><figcaption><p>The BroncBotz 2026 kicker, as a Flywheel.</p></figcaption></figure>

Changing anything in this step resets every confirmation below it.
{% endstep %}

{% step %}
#### Go through the pre-motion checklist

Nothing moves until every item is ticked. **Current trip** stops Live Tuning if the stator current stays above it for 250 ms, which catches a stalled or jammed mechanism. Also set **Max speed (RPM)**: every velocity setpoint is clamped to it.

<figure><img src="../../.gitbook/assets/live-tuner-flywheel-checklist.png" alt="The flywheel checklist with every item ticked, a 40 A current trip and a 1500 RPM max speed"><figcaption><p>Every item confirmed, with a 40 A current trip and a 1500 RPM max speed.</p></figcaption></figure>
{% endstep %}

{% step %}
#### Check the inversions

This step drives the motor at a small voltage, with no closed loop, to check that a positive voltage moves the mechanism in its positive direction and that the position reading goes the same way.

Read the warning, tick the confirmation, and press **Start voltage control**. If the inversions were already checked on this robot, **Skip (already checked)** goes straight to the next step.

Drag the slider, or type a voltage and press **Enter** or **Apply**. The slider goes back to 0 V when you let go; a typed voltage holds until you change it. Start with a fraction of a volt and raise it until the mechanism moves. A flywheel keeps spinning after you let go, so set 0 V and let it stop before you drive it the other way.

<figure><img src="../../.gitbook/assets/live-tuner-flywheel-inversions-up.png" alt="Holding 0.5 V on the kicker: Positive voltage moves forward is ticked"><figcaption><p>At +0.5 V the kicker spins forward.</p></figcaption></figure>

Then drive it the other way. Once both directions are ticked, press **Set 0 V** and **Finish check**.

<figure><img src="../../.gitbook/assets/live-tuner-flywheel-inversions-done.png" alt="Holding -0.5 V with both Positive voltage moves forward and Negative voltage moves reverse ticked, and Finish check enabled"><figcaption><p>Both directions checked.</p></figcaption></figure>

If the position goes the wrong way, stop and fix `withMotorInverted(...)`, or the encoder inversion, in the robot code before going on.
{% endstep %}

{% step %}
#### Skip the range of motion

A flywheel spins continuously, so it has no range to verify. The max speed caps it instead.
{% endstep %}

{% step %}
#### Live tune

Press **Start Live Tuning**. This starts the mechanism's `Live Tuning` command, so the speeds and gains you send are applied every loop.

Send a speed by typing it, with the slider, or with **Idle (0)** and the buttons at a quarter, half, three quarters and all of the max speed. The graph shows the measured speed against the setpoint, with the stator current.

<figure><img src="../../.gitbook/assets/live-tuner-flywheel.gif" alt="The kicker with kV at 0.25 settling below each setpoint, then with kV at 0.374 settling on it"><figcaption><p>The kicker. With kV at 0.25 it runs slow; at 0.374 it settles on each setpoint.</p></figcaption></figure>

**Change the gains**

The **Gains** card shows the values the robot publishes under `NT:/Tuning`. Type a new value and press **Enter** to apply it. [What to tune and what to watch](README.md#what-to-tune-and-what-to-watch) lists what each gain does. On a flywheel, start with kV: raise or lower it until the measured speed settles on the setpoint. Then add kP to recover speed quickly after each shot.

<figure><img src="../../.gitbook/assets/live-tuner-flywheel-gains.png" alt="The kicker's Gains card: kP 0, kI 0, kD 0; kS 0, kV 0.374, kA 0; stator limit 40 A and closed-loop ramp 0 s"><figcaption><p>kV at 0.374.</p></figcaption></figure>

**Alternate between two setpoints**

Turn on **Alternate setpoints** to switch between **Setpoint A** and **Setpoint B** every 5 seconds. Each step is the same, so you can change a gain and compare the next step with the last one.

<figure><img src="../../.gitbook/assets/live-tuner-flywheel-alternate.png" alt="Alternate setpoints on: the graph shows the kicker stepping between two speeds"><figcaption><p>The kicker stepping between two speeds.</p></figcaption></figure>
{% endstep %}

{% endstepper %}
{% endtab %}
{% endtabs %}

## Stop

Press **Space**, or **STOP**, at any time and from anywhere on the page. The Live Tuner sets the setpoint to where the mechanism is, or slows a flywheel down first, and then ends the `Live Tuning` command. The subsystem's default command, if it has one, takes over from there. Closing the page stops Live Tuning too, and so does disabling the robot from the Driver Station.

When you are happy with the gains, copy them into the robot code.
