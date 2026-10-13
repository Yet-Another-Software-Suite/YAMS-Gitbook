# How do I use exponential profiles?

Trapezoidal motion profiles assume constant maximum acceleration throughout a move, which can cause jerky starts on high-inertia mechanisms like heavy elevators or long arms. Exponential profiles let the acceleration vary continuously so the motor builds speed smoothly, following the physics of what the motor can actually deliver at each moment, this reduces overshoot and makes tuning more forgiving. Reach for exponential profiles when your trapezoidal profile produces oscillation, mechanical banging at the start of a move, or is difficult to tune around mechanical disturbances.

## What are exponential profiles?

Exponential profiles are best described by CTRE, however with YAMS they are supported by all motor controllers.

{% embed url="https://v6.docs.ctr-electronics.com/en/latest/docs/api-reference/device-specific/talonfx/motion-magic.html#motion-magic-expo" %}

## How can I use exponential profiles?

All `SmartMotorController`s support exponential profiles. Set your PID gains with `withClosedLoopController(kP, kI, kD)` and add `withExponentialProfile(...)` instead of `withTrapezoidalProfile(...)`. The static helpers on `yams.core.math.ExponentialProfilePIDController` build the profile constraints from your mechanism's physical properties.

```java
private final Distance chainPitch = Inches.of(0.25);
private final int toothCount = 22;
private final Mass     weight = Pounds.of(16);
private final DCMotor  motors = DCMotor.getNEO(1);
private final MechanismGearing gearing = new MechanismGearing(GearBox.fromReductionStages(3, 4));

private final SmartMotorControllerConfig motorConfig        = new SmartMotorControllerConfig(this)
      .withDrumRadius(chainPitch, toothCount)
      .withClosedLoopController(30, 0, 0)
      .withExponentialProfile(
      ExponentialProfilePIDController.createElevatorConstraints(
                                     Volts.of(12), // Maximum voltage during profile
                                     motors, // Motors
                                     weight, // Carriage weight
                                     chainPitch.times(toothCount).div(2 * Math.PI), // Drum radius
                                     gearing)) // Gearing
...
```

There are multiple ways to create constraints, like [`createArmConstraints`](https://yet-another-software-suite.github.io/YAMS/javadocs/yams/core/math/ExponentialProfilePIDController.html) or [`createFlywheelConstraints`](https://yet-another-software-suite.github.io/YAMS/javadocs/yams/core/math/ExponentialProfilePIDController.html) or even [`createConstraints`](https://yet-another-software-suite.github.io/YAMS/javadocs/yams/core/math/ExponentialProfilePIDController.html) from a maximum voltage, velocity and acceleration. `SmartMotorControllerConfig` also has `withExponentialProfile(...)` overloads that take the maximum voltage, `DCMotor` and moment of inertia (or carriage mass and drum radius) directly.

## When should I use exponential profiles?

Exponential Profiles are easier to tune than Trapezoidal Profiles (from `ProfiledPIDController`) and are very good at overcoming mechanical failures to a point. It CANNOT speed up a mechanism that is massively under-powered and slow to move. Trapezoidal Profiles often fail miserably when a mechanism experiences disturbances along its movement, aside from fixing those disturbances an Exponential Profile is probably your best chance at getting it "working" quickly.

<figure><img src="../.gitbook/assets/exponential-profile-tracking.gif" alt="Animation of a 12 volt motor following each profile. With the trapezoidal profile the measured velocity falls behind the setpoint as the speed rises and the motor saturates at 12 volts. With the exponential profile the measured velocity stays on the setpoint"><figcaption><p>A simulated 12 V motor following each profile. The trapezoidal profile asks for more than the motor has left at speed.</p></figcaption></figure>

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/ExponentiallyProfiledArmSubsystem.java" %}
