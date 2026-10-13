---
description: A SmartMotorControllerConfig for the drive motors and one for the azimuth motors.
---

# Step 1: The module motors

{% hint style="info" %}
This page assumes a project with YAMS and REVLib and Phoenix 6 installed. If you do not have one yet, start at [Create a project and install YAMS](../project-setup.md).
{% endhint %}

Every swerve module has two motors, and they do different jobs:

<figure><img src="../../.gitbook/assets/mech-module-how.png" alt="A swerve module seen from above, with its speed along the wheel and its angle. The drive motor uses velocity control to set how fast the wheel rolls. The azimuth motor uses position control with wrapping to set which way the wheel points. An absolute encoder tells the azimuth where the wheel is at startup. The module's state is a SwerveModuleVelocity, a speed and an angle."><figcaption><p>One module is a drive motor, an azimuth motor and an absolute encoder.</p></figcaption></figure>

* The **drive** motor turns the wheel. It is **velocity** controlled, so it gets a PID controller and a feedforward, and it needs the wheel diameter to turn rotations into distance.
* The **azimuth** motor steers the module. It is **position** controlled.

All four modules are built the same, so you write each config once and reuse it.

## The code

{% tabs %}
{% tab title="Java, Commands v2" %}
Create `SwerveSubsystem.java` in the `subsystems` folder. Each method builds a fresh config, because every motor needs a config of its own.

{% code title="subsystems/SwerveSubsystem.java" lineNumbers="true" %}
```java
package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.MetersPerSecond;

import org.wpilib.command2.SubsystemBase;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.gearing.MechanismGearing;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class SwerveSubsystem extends SubsystemBase {

  /** The config of one drive motor. Drive motors are velocity controlled. */
  private SmartMotorControllerConfig buildDriveCfg() {
    return new SmartMotorControllerConfig(this)
        // The wheel diameter turns wheel rotations into distance driven.
        .withWheelDiameter(Inches.of(4))
        // kP trims the velocity error that is left after the feedforward.
        .withClosedLoopController(0.3, 0, 0)
        // Gearing from the motor rotor to the wheel.
        .withGearing(new MechanismGearing(12.75))
        // kV does most of the work of holding a speed.
        .withFeedforward(new SimpleMotorFeedforward(0, 12.0 / (MetersPerSecond.of(1).in(MetersPerSecond) / Inches.of(4).in(Meters)), 0.01))
        .withStatorCurrentLimit(Amps.of(40))
        .withTelemetry("driveMotor", TelemetryVerbosity.HIGH);
  }

  /** The config of one azimuth (steering) motor. Azimuth motors are position controlled. */
  private SmartMotorControllerConfig buildAzimuthCfg() {
    return new SmartMotorControllerConfig(this)
        .withClosedLoopController(1, 0, 0)
        .withFeedforward(new SimpleMotorFeedforward(0, 1))
        // Gearing from the motor rotor to the module's steering axis.
        .withGearing(new MechanismGearing(6.75))
        // The azimuth motor rarely works hard, so it gets a lower limit.
        .withStatorCurrentLimit(Amps.of(20))
        .withTelemetry("angleMotor", TelemetryVerbosity.HIGH);
  }
}
```
{% endcode %}
{% endtab %}

{% tab title="Java, Commands v3" %}
Create `SwerveMechanism.java` in the `mechanisms` folder. Each method builds a fresh config, because every motor needs a config of its own.

The config comes from `yams.commands3.config`, and it takes the `Mechanism` where Commands v2 passes the subsystem.

{% code title="mechanisms/SwerveMechanism.java" lineNumbers="true" %}
```java
package first.robot.mechanisms;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Meters;
import static org.wpilib.units.Units.MetersPerSecond;

import org.wpilib.command3.Mechanism;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import yams.commands3.config.SmartMotorControllerConfig;
import yams.core.gearing.MechanismGearing;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class SwerveMechanism implements Mechanism {

  /** The config of one drive motor. Drive motors are velocity controlled. */
  private SmartMotorControllerConfig buildDriveCfg() {
    return new SmartMotorControllerConfig(this)
        // The wheel diameter turns wheel rotations into distance driven.
        .withWheelDiameter(Inches.of(4))
        // kP trims the velocity error that is left after the feedforward.
        .withClosedLoopController(0.3, 0, 0)
        // Gearing from the motor rotor to the wheel.
        .withGearing(new MechanismGearing(12.75))
        // kV does most of the work of holding a speed.
        .withFeedforward(new SimpleMotorFeedforward(0, 12.0 / (MetersPerSecond.of(1).in(MetersPerSecond) / Inches.of(4).in(Meters)), 0.01))
        .withStatorCurrentLimit(Amps.of(40))
        .withTelemetry("driveMotor", TelemetryVerbosity.HIGH);
  }

  /** The config of one azimuth (steering) motor. Azimuth motors are position controlled. */
  private SmartMotorControllerConfig buildAzimuthCfg() {
    return new SmartMotorControllerConfig(this)
        .withClosedLoopController(1, 0, 0)
        .withFeedforward(new SimpleMotorFeedforward(0, 1))
        // Gearing from the motor rotor to the module's steering axis.
        .withGearing(new MechanismGearing(6.75))
        // The azimuth motor rarely works hard, so it gets a lower limit.
        .withStatorCurrentLimit(Amps.of(20))
        .withTelemetry("angleMotor", TelemetryVerbosity.HIGH);
  }
}
```
{% endcode %}
{% endtab %}

{% tab title="C++" %}
Create `SwerveSubsystem.hpp` in `src/main/include/subsystems` and `SwerveSubsystem.cpp` in `src/main/cpp/subsystems`.

YAMS keeps a pointer to each config, so in C++ a config cannot be a temporary. These two methods fill in a config that someone else owns; step 2 gives every module its own.

`WithFeedback` is the C++ name of `withClosedLoopController`, `WithMotorGearing` of `withGearing`, and `WithMechanismDiameter` of `withWheelDiameter`.

{% code title="include/subsystems/SwerveSubsystem.hpp" lineNumbers="true" %}
```cpp
#pragma once

#include <wpi/commands2/SubsystemBase.hpp>

#include "yams/motorcontrollers/SmartMotorControllerConfig.hpp"

class SwerveSubsystem : public wpi::cmd::SubsystemBase {
 public:
  SwerveSubsystem();

 private:
  /** Fill in the config of one drive motor. Drive motors are velocity controlled. */
  void ConfigureDrive(yams::motorcontrollers::SmartMotorControllerConfig& config);

  /** Fill in the config of one azimuth (steering) motor. Azimuth motors are position controlled. */
  void ConfigureAzimuth(yams::motorcontrollers::SmartMotorControllerConfig& config);
};
```
{% endcode %}

{% code title="cpp/subsystems/SwerveSubsystem.cpp" lineNumbers="true" %}
```cpp
#include "subsystems/SwerveSubsystem.hpp"

#include <wpi/math/controller/SimpleMotorFeedforward.hpp>
#include <wpi/units/angle.hpp>
#include <wpi/units/current.hpp>
#include <wpi/units/length.hpp>
#include <wpi/units/voltage.hpp>

using Cfg = yams::motorcontrollers::SmartMotorControllerConfig;
using Feedforward = wpi::math::SimpleMotorFeedforward<wpi::units::turns>;

void SwerveSubsystem::ConfigureDrive(Cfg& config) {
  config
      .WithSubsystem(this)
      // The wheel diameter turns wheel rotations into distance driven.
      .WithMechanismDiameter(4_in)
      // kP trims the velocity error that is left after the feedforward.
      .WithFeedback(0.3, 0, 0)
      // Gearing from the motor rotor to the wheel.
      .WithMotorGearing(12.75)
      // kV does most of the work of holding a speed.
      .WithFeedforward(Feedforward{0_V, wpi::units::unit_t<Feedforward::kv_unit>{12.0 / (1.0 / 0.1016)},
                                   wpi::units::unit_t<Feedforward::ka_unit>{0.01}})
      .WithStatorCurrentLimit(40_A)
      .WithTelemetry("driveMotor", Cfg::TelemetryVerbosity::HIGH);
}

void SwerveSubsystem::ConfigureAzimuth(Cfg& config) {
  config
      .WithSubsystem(this)
      .WithFeedback(1, 0, 0)
      .WithFeedforward(Feedforward{0_V, wpi::units::unit_t<Feedforward::kv_unit>{1}})
      // Gearing from the motor rotor to the module's steering axis.
      .WithMotorGearing(6.75)
      // The azimuth motor rarely works hard, so it gets a lower limit.
      .WithStatorCurrentLimit(20_A)
      .WithTelemetry("angleMotor", Cfg::TelemetryVerbosity::HIGH);
}

SwerveSubsystem::SwerveSubsystem() {
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

The drive feedforward's `kV` is worked out from a top speed and the wheel diameter, so that the feedforward alone does most of the work of holding a speed, and the PID only has to correct what is left.

{% hint style="info" %}
The gains here are a place to start. [Live tuning](../../details/live-tuning/README.md) has a section on tuning the drive and azimuth motors of a swerve module.
{% endhint %}

Previous: [Swerve Drive](../swerve-drive.md)

Next: [Step 2: The modules and the drive](modules-and-drive.md)
