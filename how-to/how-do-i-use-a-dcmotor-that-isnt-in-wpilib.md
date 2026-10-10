# How do I use a DCMotor that isn't in WPILib?

WPILib ships physics models for common FRC motors (NEO, Falcon, CIM, etc.), but third-party or custom motors are not included. When your simulation needs accurate physics for an unlisted motor, you construct a `DCMotor` directly from its datasheet constants, stall torque, stall current, free speed, and free current, so the simulated plant behaves the same as your real hardware. Use this whenever you are simulating a mechanism driven by a motor that `DCMotor` does not already have a static factory for.

Sometimes the motor you use is not yet included in WPILib.

WPILib's `org.wpilib.math.system.DCMotor` has static factories for the motors it knows about (`getNEO`, `getNeo550`, `getNeoVortex`, `getFalcon500`, `getKrakenX60`, `getKrakenX44`, `getMinion`, and more). Each one is just a call to the `DCMotor` constructor, for example the Kraken X60:

```java
public static DCMotor getKrakenX60(int numMotors) {
  // From https://store.ctr-electronics.com/announcing-kraken-x60/
  return new DCMotor(
      12, 7.09, 366, 2, Units.rotationsPerMinuteToRadiansPerSecond(6000), numMotors);
}
```

Create a `DCMotor` from the datasheet values. Use ReCa.lc to calculate them.

{% embed url="https://www.reca.lc/motors" %}

```java
/**
 * Constructs a DC motor.
 *
 * @param nominalVoltage Voltage at which the motor constants were measured.
 * @param stallTorque Torque when stalled.
 * @param stallCurrent Current draw when stalled.
 * @param freeCurrent Current draw under no load.
 * @param freeSpeed Angular velocity under no load.
 * @param numMotors Number of motors in a gearbox.
 */
public DCMotor(
    double nominalVoltage,
    double stallTorque,
    double stallCurrent,
    double freeCurrent,
    double freeSpeed,
    int numMotors)
```

Torque is in newton-meters, current in amps and free speed in radians per second.

For example, before WPILib added `DCMotor.getMinion()`, a [Minion motor](https://store.ctr-electronics.com/products/minion-brushless-motor) could be created with:

```java
import static org.wpilib.units.Units.RPM;
import static org.wpilib.units.Units.RadiansPerSecond;
import org.wpilib.math.system.DCMotor;

int numMotors = 1;
new DCMotor(12, 3.1, 200.46, 1.43, RPM.of(7200).in(RadiansPerSecond), numMotors);
```
