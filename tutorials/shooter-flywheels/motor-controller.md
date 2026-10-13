---
description: Configure the SparkMax and wrap it in a SmartMotorController.
---

# Step 1: The motor controller

{% hint style="info" %}
This page assumes a project with YAMS and REVLib installed. If you do not have one yet, start at [Create a project and install YAMS](../project-setup.md).
{% endhint %}

A `SmartMotorController` is how YAMS talks to a motor controller. It takes three things:

1. **The vendor's motor controller object**, here a `SparkMax`.
2. **The motor** that is plugged into it, here one NEO.
3. **A `SmartMotorControllerConfig`** that says what the motor controller should do: its control mode, its PID and feedforward gains, and the gearing between the motor and the flywheel.

YAMS applies the config to the `SparkMax` for you, and uses the same config to build the simulation.

<figure><img src="../../.gitbook/assets/tutorial-gearing.gif" alt="Animation of a motor rotor turning 12 times while the mechanism it drives through a 3 to 1 stage and a 4 to 1 stage, 12 to 1 in total, turns once. Counters show the motor rotations and the mechanism rotations and angle."><figcaption><p>With 12:1 gearing, twelve rotations of the motor are one rotation of the mechanism.</p></figcaption></figure>

## The code

{% tabs %}
{% tab title="Java, Commands v2" %}
Create `ShooterSubsystem.java` in the `subsystems` folder, next to the template's `ExampleSubsystem.java`.

{% code title="subsystems/ShooterSubsystem.java" lineNumbers="true" %}
```java
package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;

import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;
import org.wpilib.command2.SubsystemBase;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import org.wpilib.math.system.DCMotor;
import yams.commands2.config.SmartMotorControllerConfig;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.local.SparkWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class ShooterSubsystem extends SubsystemBase {

  // What the motor controller should do: its control mode, gains and gearing.
  private final SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      // Feedback constants (PID), on the real robot and in simulation.
      // A flywheel holds a speed, so it gets no motion profile.
      .withClosedLoopController(1, 0, 0)
      .withSimClosedLoopController(1, 0, 0)
      // Feedforward constants, on the real robot and in simulation.
      .withFeedforward(new SimpleMotorFeedforward(0, 0, 0))
      .withSimFeedforward(new SimpleMotorFeedforward(0, 0, 0))
      // Gearing from the motor rotor to the flywheel: a 3:1 stage and a 4:1 stage, 12:1 in total.
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
      .withMotorInverted(false)
      // Coast, so that the wheel spins down by itself instead of being braked to a stop.
      .withZeroPower(MotorMode.COAST)
      .withStatorCurrentLimit(Amps.of(40))
      // Telemetry name and how much to publish.
      .withTelemetry("ShooterMotor", TelemetryVerbosity.HIGH);

  // The vendor motor controller object: a SPARK MAX with CAN ID 4.
  private final SparkMax spark = new SparkMax(CANPort.CAN_S0, 4, MotorType.kBrushless);

  // The SmartMotorController applies the config to the SPARK MAX, which drives one NEO.
  private final SmartMotorController motor = new SparkWrapper(spark, DCMotor.getNEO(1), smcConfig);
}
```
{% endcode %}
{% endtab %}

{% tab title="Java, Commands v3" %}
Create `ShooterMechanism.java` in the `mechanisms` folder, next to the template's `ExampleMechanism.java`.

The config comes from `yams.commands3.config`, and it takes the `Mechanism` where Commands v2 passes the subsystem.

{% code title="mechanisms/ShooterMechanism.java" lineNumbers="true" %}
```java
package first.robot.mechanisms;

import static org.wpilib.units.Units.Amps;

import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.spark.SparkMax;
import org.wpilib.command3.Mechanism;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import org.wpilib.math.system.DCMotor;
import yams.commands3.config.SmartMotorControllerConfig;
import yams.core.gearing.GearBox;
import yams.core.gearing.MechanismGearing;
import yams.core.motorcontrollers.SmartMotorController;
import yams.core.motorcontrollers.enums.ControlMode;
import yams.core.motorcontrollers.enums.MotorMode;
import yams.core.motorcontrollers.local.SparkWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class ShooterMechanism implements Mechanism {

  // What the motor controller should do: its control mode, gains and gearing.
  private final SmartMotorControllerConfig smcConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      // Feedback constants (PID), on the real robot and in simulation.
      // A flywheel holds a speed, so it gets no motion profile.
      .withClosedLoopController(1, 0, 0)
      .withSimClosedLoopController(1, 0, 0)
      // Feedforward constants, on the real robot and in simulation.
      .withFeedforward(new SimpleMotorFeedforward(0, 0, 0))
      .withSimFeedforward(new SimpleMotorFeedforward(0, 0, 0))
      // Gearing from the motor rotor to the flywheel: a 3:1 stage and a 4:1 stage, 12:1 in total.
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
      .withMotorInverted(false)
      // Coast, so that the wheel spins down by itself instead of being braked to a stop.
      .withZeroPower(MotorMode.COAST)
      .withStatorCurrentLimit(Amps.of(40))
      // Telemetry name and how much to publish.
      .withTelemetry("ShooterMotor", TelemetryVerbosity.HIGH);

  // The vendor motor controller object: a SPARK MAX with CAN ID 4.
  private final SparkMax spark = new SparkMax(CANPort.CAN_S0, 4, MotorType.kBrushless);

  // The SmartMotorController applies the config to the SPARK MAX, which drives one NEO.
  private final SmartMotorController motor = new SparkWrapper(spark, DCMotor.getNEO(1), smcConfig);
}
```
{% endcode %}
{% endtab %}

{% tab title="C++" %}
Create `ShooterSubsystem.hpp` in `src/main/include/subsystems` and `ShooterSubsystem.cpp` in `src/main/cpp/subsystems`.

YAMS keeps a pointer to the config and to the `SparkMax`, so in C++ both are members of the subsystem. The constructor fills in the config and then creates the `SmartMotorController`, which is why that one is a `std::optional`.

`WithFeedback` is the C++ name of `withClosedLoopController`, and `WithMotorGearing` of `withGearing`.

{% code title="include/subsystems/ShooterSubsystem.hpp" lineNumbers="true" %}
```cpp
#pragma once

#include <optional>

#include <rev/SparkMax.h>
#include <wpi/commands2/SubsystemBase.hpp>
#include <wpi/hardware/bus/CANPort.hpp>

#include "yams/motorcontrollers/SmartMotorControllerConfig.hpp"
#include "yams/motorcontrollers/local/SparkWrapper.hpp"

class ShooterSubsystem : public wpi::cmd::SubsystemBase {
 public:
  ShooterSubsystem();

 private:
  // The vendor motor controller object: a SPARK MAX with CAN ID 4.
  rev::spark::SparkMax m_spark{wpi::CANPort::CAN_S0, 4, rev::spark::SparkMax::MotorType::kBrushless};

  // What the motor controller should do. It is filled in by the constructor.
  yams::motorcontrollers::SmartMotorControllerConfig m_smcConfig;

  // The SmartMotorController, created in the constructor once the config is ready.
  std::optional<yams::motorcontrollers::local::SparkWrapper> m_motor;
};
```
{% endcode %}

{% code title="cpp/subsystems/ShooterSubsystem.cpp" lineNumbers="true" %}
```cpp
#include "subsystems/ShooterSubsystem.hpp"

#include <wpi/math/controller/SimpleMotorFeedforward.hpp>
#include <wpi/math/system/DCMotor.hpp>
#include <wpi/units/angle.hpp>
#include <wpi/units/current.hpp>
#include <wpi/units/voltage.hpp>

#include "yams/gearing/GearBox.hpp"
#include "yams/gearing/MechanismGearing.hpp"

using namespace yams::gearing;
using Cfg = yams::motorcontrollers::SmartMotorControllerConfig;
using Feedforward = wpi::math::SimpleMotorFeedforward<wpi::units::turns>;

ShooterSubsystem::ShooterSubsystem() {
  // What the motor controller should do: its control mode, gains and gearing.
  m_smcConfig
      .WithSubsystem(this)
      .WithClosedLoopMode()
      // Feedback constants (PID), on the real robot and in simulation.
      // A flywheel holds a speed, so it gets no motion profile.
      .WithFeedback(1, 0, 0)
      .WithSimClosedLoopController(1, 0, 0)
      // Feedforward constants: kS and kV.
      .WithFeedforward(Feedforward{0_V, wpi::units::unit_t<Feedforward::kv_unit>{0}})
      // Gearing from the motor rotor to the flywheel: a 3:1 stage and a 4:1 stage, 12:1 in total.
      .WithMotorGearing(MechanismGearing{GearBox::FromReductionStages({3.0, 4.0})})
      .WithMotorInverted(false)
      // Coast, so that the wheel spins down by itself instead of being braked to a stop.
      .WithZeroPower(Cfg::MotorMode::COAST)
      .WithStatorCurrentLimit(40_A)
      // Telemetry name and how much to publish.
      .WithTelemetry("ShooterMotor", Cfg::TelemetryVerbosity::HIGH);

  // The SmartMotorController applies the config to the SPARK MAX, which drives one NEO.
  m_motor.emplace(&m_spark, wpi::math::DCMotor::NEO(1), &m_smcConfig);
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

## What a flywheel leaves out

A flywheel holds a speed and has no position to protect, so this config has no motion profile, no starting position and no soft limits. It coasts at zero power, so that the wheel is not braked to a stop every time a command ends.

## Read more

Every option used here has its own section in the `SmartMotorControllerConfig` pages:

* [Motor basics](../../details/editor/config/motor-basics.md): control mode, inversion, what the motor does at zero power
* [Gearing and units](../../details/editor/config/gearing-and-units.md): gearing, and how rotations become distance
* [Closed loop control](../../details/editor/config/closed-loop-control.md), [Feedforward](../../details/editor/config/feedforward.md) and [Motion profiles](../../details/editor/config/motion-profiles.md)
* [Limits and protection](../../details/editor/config/limits-and-protection.md): soft limits and current limits
* [Simulation](../../details/editor/config/simulation.md): the simulation only gains

The PID and feedforward numbers on this page are a place to start, not tuned values. Tune them on your own mechanism with [Live tuning](../../details/live-tuning/README.md).

Previous: [Shooter/FlyWheels](../shooter-flywheels.md)

Next: [Step 2: The flywheel](mechanism.md)
