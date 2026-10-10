---
icon: microchip
---

# TunerX in simulation

For TalonFX and TalonFXS motors, YAMS feeds its physics simulation results back through CTRE's vendorsim layer via Phoenix's `getSimState()` API. This means Phoenix Tuner X can connect to your simulation and display every motor signal, position, velocity, closed-loop error, applied voltage, current draw, exactly as it would on real hardware.

**To use TunerX in sim:**

1. In a terminal, run your robot code in simulation (for example, `./gradlew simulateJava`).
2. Open Phoenix Tuner X.
3. Connect to the simulation instance. It appears as a local device.
4. In the motor view, open the **Signal** tab. The physics simulation updates every signal in real time.

This is the same workflow as diagnosing real hardware. You can verify gain behavior, confirm soft/hard limit triggering, and plot signal graphs without ever touching a physical robot.

{% hint style="info" %}
TunerX vendorsim integration is specific to **CTRE TalonFX and TalonFXS** motors. REV Spark motors use WPILib's simulation framework and are visible through AdvantageScope or Elastic via NetworkTables instead.
{% endhint %}

TunerX shows the motor controller's own signals. To change YAMS gains, profiles and limits while the code runs, use [Live tuning](README.md).
