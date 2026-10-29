---
icon: microchip
---

# TunerX in simulation

For TalonFX and TalonFXS motors, YAMS feeds its physics simulation results back through CTRE's vendorsim layer via Phoenix's `getSimState()` API. This means Phoenix Tuner X can connect to your simulation and display every motor signal (position, velocity, closed loop error, applied voltage, current draw) exactly as it would on real hardware.

**To use TunerX in sim:**

1. Run your robot code in simulation, for example with **WPILib: Simulate Robot Code** in VS Code, or `./gradlew simulateJava` in a terminal.
2. Open Phoenix Tuner X.
3. Open the flyout menu in the upper left. Click the arrow next to the connection field and choose the **localhost** preset. CTRE describes it as the setting for simulation. See [Connecting Tuner X](https://v6.docs.ctr-electronics.com/en/stable/docs/tuner/connecting.html) in the Phoenix 6 docs.
4. Open the **Devices** page. Every simulated TalonFX and TalonFXS shows up with the CAN ID from your code, all on one CAN bus.
5. Select a motor and plot its signals. Enable the robot and move the mechanism: the position, velocity, applied voltage and current change as the simulated mechanism moves.

**Check:** the devices list shows one entry for each TalonFX or TalonFXS in your code. If it is empty, check the simulation is still running and the connection field says localhost.

{% hint style="warning" %}
In simulation every CAN device appears on the same CAN bus, so two simulated devices with the same CAN ID on different buses clash. Give them unique IDs.
{% endhint %}

This is the same workflow as diagnosing real hardware. You can verify gain behavior, confirm soft/hard limit triggering, and plot signal graphs without ever touching a physical robot.

{% hint style="info" %}
TunerX vendorsim integration is specific to **CTRE TalonFX and TalonFXS** motors. REV Spark motors use WPILib's simulation framework and are visible through AdvantageScope or Elastic via NetworkTables instead.
{% endhint %}

TunerX shows the motor controller's own signals. To change YAMS gains, profiles and limits while the code runs, use [Live tuning](README.md).
