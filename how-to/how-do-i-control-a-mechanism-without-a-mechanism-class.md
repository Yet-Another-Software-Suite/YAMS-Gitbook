---
description: >-
  Control each SmartMotorController yourself, such as when the motors of one
  mechanism need different setpoints.
---

# How do I control a Mechanism without a Mechanism Class?

YAMS mechanism classes (`Arm`, `Elevator`, `FlyWheel`, etc.) drive one motor, along with its followers. Motors on one shaft or gearbox are [followers](../details/editor/config/followers.md#withfollowers). Motors that aren't, but must move together, such as one on each side of an intake arm, are [loosely coupled followers](../details/editor/config/followers.md#withlooselycoupledfollowers), and still work with a mechanism class.

Control each `SmartMotorController` yourself, as this page shows, when the motors need different setpoints, such as two flywheels spinning at different speeds, or when one subsystem holds more than one mechanism, such as a shooter and an indexer. You still get YAMS's telemetry, current limiting and profiling.

<figure><img src="../.gitbook/assets/smc-followers.png" alt="Two panels. Tightly coupled, withFollowers: a leader and a follower are on one shaft or gearbox and the follower copies the leader's output. The motors are mechanically locked, there is one closed loop and one config, the followers are the same vendor as the leader, and voltage and duty cycle reach every motor. Loosely coupled, withLooselyCoupledFollowers: two separate motors joined only by the mechanism get the same setpoint and each runs its own PID. They are not mechanically locked, each has its own closed loop and config, vendors can be mixed, and voltage and duty cycle are not forwarded, so the leader moves alone."><figcaption><p>Tightly coupled followers share a shaft. Loosely coupled followers are separate motors.</p></figcaption></figure>

## Subsystems with `SmartMotorController`'s

Subsystems given when you create the `SmartMotorControllerConfig` are only used for YAMS generated commands. This means that if you're using a YAMS generated command like `Arm.run`, `Arm.setVoltage`, or `Elevator.run` that command will be the **ONLY** command allowed to run on that subsystems.&#x20;

## Controlling an Elevator without Elevator.run()

IF you know how to use Command Based Programming well enough you can create your own commands and control the `SmartMotorController` without the Mechanism but remember to create the Mechanism and Mechanism config because it will modify and re-apply the `SmartMotorControllerConfig` &#x20;

The following example shows exactly how you can recreate the commands used in YAMS without using the helpful functions to create them in the first place and could allow you to use multiple mechanisms in the same Subsystem if you needed it. (For example a shooter and indexer)

An example of this for an Elevator would be as follows

{% tabs %}
{% tab title="Commands v2" %}
<pre class="language-java"><code class="lang-java">public class ExampleSubsystem extends SubsystemBase {
  
  // Chain driven elevator drum calculation
  private Distance chainPitch = Inches.of(0.25);
  private int teethCount = 22;
  private Distance drumCircumference = chainPitch.times(teethCount);
  private Distance drumRadius = drumCircumference.div(2*Math.PI);
  
  // Gearing from the motor rotor to final shaft.
  // In this example GearBox.fromReductionStages(3,4) is the same as GearBox.fromStages("3:1","4:1") which corresponds to the gearbox attached to your motor.
  private MechanismGearing gearing = new MechanismGearing(GearBox.fromReductionStages(3,4));
  
  // Profiled PID Controller
  private double kP = 4, kI = 0, kD = 0;
  private LinearVelocity maxVel = MetersPerSecond.of(0.5);
  private LinearAcceleration maxAccel = MetersPerSecondPerSecond.of(0.5);
  
  // Feedforwards
  private ElevatorFeedforward realFeedforward = new ElevatorFeedforward(0,0,0);
  private ElevatorFeedforward simFeedforward = new ElevatorFeedforward(0,0,0);
  
  private MotorMode idle = MotorMode.BRAKE;
  private Current statorLimit = Amps.of(40);
  // Ramp rate is how long the motor should take to run from 0 to 100%
  private Time openLoopRampRate = Seconds.of(0.25);
  private Time closedLoopRampRate = Seconds.of(0.25);
  
  private SmartMotorControllerConfig leftSmcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Elevator drum radius: derived from chain pitch and tooth count.
  .withDrumRadius(drumRadius)
  // Starting height of the elevator, used for simulation.
  .withStartingPosition(Meters.of(0.5))
  // Feedback Constants (PID Constants)
  .withClosedLoopController(kP, kI, kD)
  .withTrapezoidalProfile(maxVel, maxAccel)
  // Uncomment below to set sim only values.
  //.withSimClosedLoopController(4, 0, 0)
  // Feedforward Constants
  .withFeedforward(realFeedforward)
  .withSimFeedforward(simFeedforward)
  // Telemetry name and verbosity level
  .withTelemetry("LeftElevatorMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example gearbox(3,4) is the same as gearbox("3:1","4:1") which corresponds to the gearbox attached to your motor.
  .withGearing(gearing)
  // Motor properties to prevent over currenting.
  .withMotorInverted(false)
  .withZeroPower(idle)
  .withStatorCurrentLimit(statorLimit)
  .withClosedLoopRampRate(closedLoopRampRate)
  .withOpenLoopRampRate(openLoopRampRate);
  
  private SmartMotorControllerConfig rightSmcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Elevator drum radius: derived from chain pitch and tooth count.
  .withDrumRadius(drumRadius)
  // Starting height of the elevator, used for simulation.
  .withStartingPosition(Meters.of(0.5))
  // Feedback Constants (PID Constants)
  .withClosedLoopController(kP, kI, kD)
  .withTrapezoidalProfile(maxVel, maxAccel)
  // Uncomment below to set sim only values.
  //.withSimClosedLoopController(4, 0, 0)
  // Feedforward Constants
  .withFeedforward(realFeedforward)
  .withSimFeedforward(simFeedforward)
  // Telemetry name and verbosity level
  .withTelemetry("RightElevatorMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example gearbox(3,4) is the same as gearbox("3:1","4:1") which corresponds to the gearbox attached to your motor.
  .withGearing(gearing)
  // Motor properties to prevent over currenting.
  .withMotorInverted(false)
  .withZeroPower(idle)
  .withStatorCurrentLimit(statorLimit)
  .withClosedLoopRampRate(closedLoopRampRate)
  .withOpenLoopRampRate(openLoopRampRate);

  // Vendor motor controller object
  private SparkMax leftSpark = new SparkMax(CANPort.CAN_S0, 4, MotorType.kBrushless);
  private SparkMax rightSpark = new SparkMax(CANPort.CAN_S0, 6, MotorType.kBrushless);

  // Create our SmartMotorController from our Spark and config with the NEO.
  private SmartMotorController leftSparkSmartMotorController = new SparkWrapper(leftSpark, DCMotor.getNEO(1), leftSmcConfig);
  private SmartMotorController rightSparkSmartMotorController = new SparkWrapper(rightSpark, DCMotor.getNEO(1), rightSmcConfig);
  
  // Include this only if you want a pretty sim
  /*
  private ElevatorConfig elevconfig = new ElevatorConfig()
      .withHardLimits(Meters.of(0), Meters.of(3))
      .withTelemetry("Elevator", TelemetryVerbosity.HIGH)
      .withCarriageWeight(Pounds.of(16));

  // Elevator Mechanism
  private Elevator elevator = new Elevator(elevconfig, leftSparkSmartMotorController);
  */
  
  /**
   * Set the dutycycle of the SmartMotorController without interrupting 
   *
   * @param dutycycle Value between [-1, 1] which will control the speed of the motor.
   * @return StartRun command which turns off the closed loop controller, sets the dutycycle of the SMC then turns the closed loop controller back on when interrupted.
   */
<strong>  public Command setDutyCycle(double dutycycle)
</strong><strong>  {
</strong><strong>    return startRun(()->{leftSparkSmartMotorController.stopClosedLoopController();rightSparkSmartMotorController.stopClosedLoopController();}, // Stop the closed loop controller since the motor is in ControlMode.CLOSED_LOOP
</strong><strong>           () -> {leftSparkSmartMotorController.setDutyCycle(dutycycle);rightSparkSmartMotorController.setDutyCycle(dutycycle);}) // Apply the dutycycle given
</strong><strong>                   .finallyDo(()->{leftSparkSmartMotorController.startClosedLoopController();rightSparkSmartMotorController.startClosedLoopController();}) // Start the closed loop controller when this command is interrupted
</strong><strong>                   .withName("Custom SetDutyCycle"); // Be nice, give your command name :)
</strong><strong>  }
</strong>  
  /**
   * Set the position of the spark motor controller.
   * @param position Mechanism position to go to.
   * @return RunCommand which will set the position of the mechanism until interrupted.
   */
<strong>  public Command setMechanismPosition(Angle position)
</strong><strong>  {
</strong><strong>    return run(() -> {leftSparkSmartMotorController.setPosition(position);rightSparkSmartMotorController.setPosition(position);})
</strong><strong>            .withName("Custom SetPosition");
</strong><strong>  }
</strong>  
  /**
   * Set the position of the spark motor controller.
   * @param height Measurement position to go to.
   * @return RunCommand which will set the position of the measurement until interrupted.
   */
<strong>  public Command setMechanismPosition(Distance height)
</strong><strong>  {
</strong><strong>    return run(() -> {leftSparkSmartMotorController.setPosition(height);rightSparkSmartMotorController.setPosition(height);})
</strong><strong>            .withName("Custom SetPosition with Height");
</strong><strong>  }
</strong>  
  @Override
  public void periodic() {
    // This method will be called once per scheduler run
    leftSparkSmartMotorController.updateTelemetry();
    rightSparkSmartMotorController.updateTelemetry();
  }

  @Override
  public void simulationPeriodic() {
    // This method will be called once per scheduler run during simulation
    leftSparkSmartMotorController.simIterate();
    rightSparkSmartMotorController.simIterate();
  }
}
</code></pre>
{% endtab %}

{% tab title="Commands v3" %}
```java
public class ExampleMechanism implements Mechanism {
  
  // Chain driven elevator drum calculation
  private Distance chainPitch = Inches.of(0.25);
  private int teethCount = 22;
  private Distance drumCircumference = chainPitch.times(teethCount);
  private Distance drumRadius = drumCircumference.div(2*Math.PI);
  
  // Gearing from the motor rotor to final shaft.
  // In this example GearBox.fromReductionStages(3,4) is the same as GearBox.fromStages("3:1","4:1") which corresponds to the gearbox attached to your motor.
  private MechanismGearing gearing = new MechanismGearing(GearBox.fromReductionStages(3,4));
  
  // Profiled PID Controller
  private double kP = 4, kI = 0, kD = 0;
  private LinearVelocity maxVel = MetersPerSecond.of(0.5);
  private LinearAcceleration maxAccel = MetersPerSecondPerSecond.of(0.5);
  
  // Feedforwards
  private ElevatorFeedforward realFeedforward = new ElevatorFeedforward(0,0,0);
  private ElevatorFeedforward simFeedforward = new ElevatorFeedforward(0,0,0);
  
  private MotorMode idle = MotorMode.BRAKE;
  private Current statorLimit = Amps.of(40);
  // Ramp rate is how long the motor should take to run from 0 to 100%
  private Time openLoopRampRate = Seconds.of(0.25);
  private Time closedLoopRampRate = Seconds.of(0.25);
  
  private SmartMotorControllerConfig leftSmcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Elevator drum radius: derived from chain pitch and tooth count.
  .withDrumRadius(drumRadius)
  // Starting height of the elevator, used for simulation.
  .withStartingPosition(Meters.of(0.5))
  // Feedback Constants (PID Constants)
  .withClosedLoopController(kP, kI, kD)
  .withTrapezoidalProfile(maxVel, maxAccel)
  // Uncomment below to set sim only values.
  //.withSimClosedLoopController(4, 0, 0)
  // Feedforward Constants
  .withFeedforward(realFeedforward)
  .withSimFeedforward(simFeedforward)
  // Telemetry name and verbosity level
  .withTelemetry("LeftElevatorMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example gearbox(3,4) is the same as gearbox("3:1","4:1") which corresponds to the gearbox attached to your motor.
  .withGearing(gearing)
  // Motor properties to prevent over currenting.
  .withMotorInverted(false)
  .withZeroPower(idle)
  .withStatorCurrentLimit(statorLimit)
  .withClosedLoopRampRate(closedLoopRampRate)
  .withOpenLoopRampRate(openLoopRampRate);
  
  private SmartMotorControllerConfig rightSmcConfig = new SmartMotorControllerConfig(this)
  .withControlMode(ControlMode.CLOSED_LOOP)
  // Elevator drum radius: derived from chain pitch and tooth count.
  .withDrumRadius(drumRadius)
  // Starting height of the elevator, used for simulation.
  .withStartingPosition(Meters.of(0.5))
  // Feedback Constants (PID Constants)
  .withClosedLoopController(kP, kI, kD)
  .withTrapezoidalProfile(maxVel, maxAccel)
  // Uncomment below to set sim only values.
  //.withSimClosedLoopController(4, 0, 0)
  // Feedforward Constants
  .withFeedforward(realFeedforward)
  .withSimFeedforward(simFeedforward)
  // Telemetry name and verbosity level
  .withTelemetry("RightElevatorMotor", TelemetryVerbosity.HIGH)
  // Gearing from the motor rotor to final shaft.
  // In this example gearbox(3,4) is the same as gearbox("3:1","4:1") which corresponds to the gearbox attached to your motor.
  .withGearing(gearing)
  // Motor properties to prevent over currenting.
  .withMotorInverted(false)
  .withZeroPower(idle)
  .withStatorCurrentLimit(statorLimit)
  .withClosedLoopRampRate(closedLoopRampRate)
  .withOpenLoopRampRate(openLoopRampRate);

  // Vendor motor controller object
  private SparkMax leftSpark = new SparkMax(CANPort.CAN_S0, 4, MotorType.kBrushless);
  private SparkMax rightSpark = new SparkMax(CANPort.CAN_S0, 6, MotorType.kBrushless);

  // Create our SmartMotorController from our Spark and config with the NEO.
  private SmartMotorController leftSparkSmartMotorController = new SparkWrapper(leftSpark, DCMotor.getNEO(1), leftSmcConfig);
  private SmartMotorController rightSparkSmartMotorController = new SparkWrapper(rightSpark, DCMotor.getNEO(1), rightSmcConfig);
  
  // Include this only if you want a pretty sim
  /*
  private ElevatorConfig elevconfig = new ElevatorConfig()
      .withHardLimits(Meters.of(0), Meters.of(3))
      .withTelemetry("Elevator", TelemetryVerbosity.HIGH)
      .withCarriageWeight(Pounds.of(16));

  // Elevator Mechanism
  private Elevator elevator = new Elevator(elevconfig, leftSparkSmartMotorController);
  */
  
  /**
   * Set the dutycycle of the SmartMotorController without interrupting 
   *
   * @param dutycycle Value between [-1, 1] which will control the speed of the motor.
   * @return Command which turns off the closed loop controller, sets the dutycycle of the SMC every loop, then turns the closed loop controller back on when it is canceled.
   */
  public Command setDutyCycle(double dutycycle)
  {
    return run(coroutine -> {
      // Stop the closed loop controller since the motors are being driven directly
      leftSparkSmartMotorController.stopClosedLoopController();
      rightSparkSmartMotorController.stopClosedLoopController();
      while (true) {
        // Apply the dutycycle given
        leftSparkSmartMotorController.setDutyCycle(dutycycle);
        rightSparkSmartMotorController.setDutyCycle(dutycycle);
        coroutine.yield();
      }
    }).whenCanceled(() -> {
      // Start the closed loop controller again when the command is interrupted
      leftSparkSmartMotorController.startClosedLoopController();
      rightSparkSmartMotorController.startClosedLoopController();
    }).named("Custom SetDutyCycle"); // Be nice, give your command name :)
  }
  
  /**
   * Set the position of the spark motor controller.
   * @param position Mechanism position to go to.
   * @return Command which will set the position of the mechanism until interrupted.
   */
  public Command setMechanismPosition(Angle position)
  {
    return run(coroutine -> {
      while (true) {
        leftSparkSmartMotorController.setPosition(position);
        rightSparkSmartMotorController.setPosition(position);
        coroutine.yield();
      }
    }).named("Custom SetPosition");
  }
  
  /**
   * Set the position of the spark motor controller.
   * @param height Measurement position to go to.
   * @return Command which will set the position of the measurement until interrupted.
   */
  public Command setMechanismPosition(Distance height)
  {
    return run(coroutine -> {
      while (true) {
        leftSparkSmartMotorController.setPosition(height);
        rightSparkSmartMotorController.setPosition(height);
        coroutine.yield();
      }
    }).named("Custom SetPosition with Height");
  }
  
  public void periodic() {
    // A Commands v3 Mechanism has no periodic hook: call this from Robot.robotPeriodic()
    leftSparkSmartMotorController.updateTelemetry();
    rightSparkSmartMotorController.updateTelemetry();
  }

  public void simulationPeriodic() {
    // A Commands v3 Mechanism has no periodic hook: call this from Robot.robotPeriodic() in simulation
    leftSparkSmartMotorController.simIterate();
    rightSparkSmartMotorController.simIterate();
  }
}
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
For Commands v2, `SmartMotorControllerConfig` and `Elevator` come from `yams.commands2.config` and `yams.commands2.mechanisms`. `SmartMotorController`, `SparkWrapper`, `ElevatorConfig`, the gearing classes and the enums come from `yams.core`. With Commands v3, use the `yams.commands3` classes and write the commands as coroutines on your `Mechanism` instead of `startRun(...)` and `run(...)`.

If both motors should simply chase the same position or velocity setpoint, make one a [loosely coupled follower](../details/editor/config/followers.md#withlooselycoupledfollowers) of the other instead. Keep at most one YAMS mechanism class per subsystem so live tuning keeps working.
{% endhint %}

{% hint style="success" %}
Remember that mechanisms will re-apply a modified `SmartMotorControllerConfig` with changes from their respective config classes when the mechanism is created!&#x20;

So even if you're not going to use the mechanism you should still create it!
{% endhint %}
