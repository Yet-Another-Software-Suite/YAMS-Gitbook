---
icon: robot
---

# Simulation-only PID + feedforward

Simulation is not exactly 1:1 to the real robot, no matter how much we wish it was. Therefore it is useful to setup a sim-only PID and FF value that will mimic the robot fairly well.

## Configure this in YAMS

```java
SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Feedback Constants (PID Constants)
  .withClosedLoopController(50, 0, 0)
  .withSimClosedLoopController(50, 0, 0)
  // Feedforward Constants
  .withFeedforward(new ArmFeedforward(0, 0, 0))
  .withSimFeedforward(new ArmFeedforward(0, 0, 0))
```
