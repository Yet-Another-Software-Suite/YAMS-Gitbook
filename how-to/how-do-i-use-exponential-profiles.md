# How do I use exponential profiles?

An exponential profile plans a move the way a DC motor actually moves: hard acceleration at first, then less as it nears top speed. Every `SmartMotorController` in YAMS supports them. This is the short version: [Exponential profiles](../details/editor/exponential-profiles.md) has the full explanation and the tuning steps.

<figure><img src="../.gitbook/assets/exponential-profile-compare.png" alt="Two graphs. Setpoint velocity over time: a trapezoidal profile ramps in a straight line to a flat top, an exponential profile rises quickly and curves over near top speed. Acceleration against speed: the motor's available acceleration falls to zero at top speed, the exponential profile follows it, and the trapezoidal profile asks for a constant acceleration the motor cannot give at high speed"><figcaption><p>The two profiles, and why the motor can follow one more easily than the other.</p></figcaption></figure>

## Use one

Add `withExponentialProfile(...)` instead of `withTrapezoidalProfile(...)`. The helpers on `yams.core.math.ExponentialProfilePIDController` work out starting values from the mechanism:

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

There are helpers for arms, flywheels and custom limits too: see [creating constraints](../details/editor/exponential-profiles.md#creating-constraints).

## When should I use one?

<figure><img src="../.gitbook/assets/exponential-profile-tracking.gif" alt="Animation of a 12 volt motor following each profile. With the trapezoidal profile the measured velocity falls behind the setpoint as the speed rises and the motor saturates at 12 volts. With the exponential profile the measured velocity stays on the setpoint"><figcaption><p>A simulated 12 V motor following each profile. The trapezoidal profile asks for more than the motor has left at speed.</p></figcaption></figure>

* **Use one** when a trapezoidal profile is hard to tune, or the mechanism falls behind it and then overshoots.
* **Keep the trapezoidal profile** if it already works well, or the motor is always current limited.
* **It cannot fix an underpowered mechanism.**

See [when to use each profile type](../details/editor/exponential-profiles.md#when-to-use-each-profile-type).

## Tune it

Start from the helper's values and [refine the profile live](../details/editor/exponential-profiles.md#refining-the-profile-live). The profile only plans the move, so it still needs a feedforward, and the profile's `kV` and `kA` are [not the feedforward's](../details/editor/config/feedforward.md#these-are-not-an-exponential-profiles-kv-and-ka).

## Code reference

{% @github-files/github-code-block url="https://github.com/Yet-Another-Software-Suite/YAMS/blob/master/examples/commands2/simple_robot-2027/src/main/java/first/robot/subsystems/ExponentiallyProfiledArmSubsystem.java" %}
