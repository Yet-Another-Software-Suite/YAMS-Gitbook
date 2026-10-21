---
description: Run the elevator in simulation and drive it with a controller.
---

# Step 4: Simulate it

The code from the first three steps runs on your computer exactly as it will on the robot. YAMS simulates the motor, the gearing and the elevator, so the closed loop, the limits and the commands all behave the way they will on the real mechanism.

## Start the simulation

{% tabs %}
{% tab title="Java, Commands v2" %}
Open the command palette with <kbd>Ctrl+Shift+P</kbd> and run **WPILib: Simulate Robot Code**. If you are asked which extensions to run, tick **Sim GUI**.
{% endtab %}

{% tab title="Java, Commands v3" %}
Open the command palette with <kbd>Ctrl+Shift+P</kbd> and run **WPILib: Simulate Robot Code**. If you are asked which extensions to run, tick **Sim GUI**.

The buttons are bound in the `ExampleTeleop` op mode, so that is the op mode to run when you enable.

If the simulated mechanism never moves, check that `Robot.java` has a `simulationInit()` that calls `startSimulation(...)` for the mechanism.
{% endtab %}

{% tab title="C++" %}
Open the command palette with <kbd>Ctrl+Shift+P</kbd> and run **WPILib: Simulate Robot Code**. If you are asked which extensions to run, tick **Sim GUI**.

The first C++ build compiles the whole project, so it takes a few minutes. Later builds are much faster.
{% endtab %}
{% endtabs %}

<figure><img src="../../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

## Drive it

1. Connect a controller to your computer, then drag it from **System Joysticks** to **Joystick\[0]**.
2. Open the simulated mechanism with **NetworkTables -> SmartDashboard -> Elevator -> mechanism**.

<figure><img src="../../.gitbook/assets/Robot Simulation 8_30_2025 7_06_08 PM.png" alt=""><figcaption></figcaption></figure>

3. Resize **Elevator/mechanism** to your liking.
4. Press **Teleoperated** in **Robot State**. The controller now drives the simulated elevator just as it would the real one.

<figure><img src="../../.gitbook/assets/Robot Simulation 2025-08-30 18-37-59.gif" alt=""><figcaption></figcaption></figure>

Congratulations on successfully programming your Elevator! :tada::tada:

## Where to go next

* Tune the gains while the simulation runs: [Live tuning](../../details/live-tuning/README.md)
* Tune it in your browser with the [YAMS Live Tuner](https://tuning.yassrobotics.com/): [Tune in your browser with the YAMS Live Tuner](../../details/live-tuning/README.md#tune-in-your-browser-with-the-yams-live-tuner)
* See everything the mechanism can do: [Elevators](../../details/elevators.md) and [ElevatorConfig](../../details/mechanism-configs/elevator-config.md)
* Look at what the mechanism publishes: [Telemetry](../../understanding/telemetry.md)
* Keep your configs tidy as the robot grows: [Organizing configs in constants](../../details/config-organization.md)
