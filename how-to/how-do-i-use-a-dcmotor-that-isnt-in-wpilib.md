# How do I use a DCMotor that isn't in WPILib?

WPILib ships physics models for common FRC motors (NEO, Falcon, CIM, etc.), but third-party or custom motors are not included. When your simulation needs accurate physics for an unlisted motor, you construct a `DCMotor` directly from its datasheet constants, stall torque, stall current, free speed, and free current, so the simulated plant behaves the same as your real hardware. Use this whenever you are simulating a mechanism driven by a motor that `DCMotor` does not already have a static factory for.

Sometimes the motor you use is not yet included in WPILib.

{% @github-files/github-code-block url="https://github.com/wpilibsuite/allwpilib/blob/main/wpimath/src/main/java/edu/wpi/first/math/system/plant/DCMotor.java#L166-L175" %}

Create a `DCMotor` from the datasheet values. Use ReCa.lc to calculate them.

{% embed url="https://www.reca.lc/motors" %}

{% @github-files/github-code-block url="https://github.com/wpilibsuite/allwpilib/blob/main/wpimath/src/main/java/edu/wpi/first/math/system/plant/DCMotor.java#L45-L61" %}

For example, a [Minion motor](https://store.ctr-electronics.com/products/minion-brushless-motor) uses:

```java
int numMotors = 1;
new DCMotor(12, 3.1, 200.46, 1.43, RPM.of(7200).in(RadiansPerSecond), numMotors);
```
