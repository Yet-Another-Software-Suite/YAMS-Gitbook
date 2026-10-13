---
description: One config and one SmartMotorController for each flywheel.
---

# Step 1: Two motor controllers

{% hint style="info" %}
This page assumes a project with YAMS and Phoenix 6 installed. If you do not have one yet, start at [Create a project and install YAMS](../project-setup.md).
{% endhint %}

Each flywheel gets its own `SmartMotorControllerConfig` and its own `SmartMotorController`, so that each can be given its own speed. The two configs are the same except for the telemetry name and the direction of the motor.

There is no `FlyWheel` mechanism in this tutorial, so three jobs a mechanism class would do fall to the subsystem:

* **Describe the wheel.** `withMomentOfInertia(diameter, weight)` on the config gives the simulation the wheel it needs.
* **Set up the telemetry.** `setupTelemetry()` on each `SmartMotorController`, once, in the constructor. This is also what makes live tuning work.
* **Update every loop.** `updateTelemetry()` on each, and `simIterate()` on each in simulation.

<figure><img src="../../.gitbook/assets/smc-followers.png" alt="Two panels. Tightly coupled, withFollowers: a leader and a follower are on one shaft or gearbox and the follower copies the leader's output. The motors are mechanically locked, there is one closed loop and one config, the followers are the same vendor as the leader, and voltage and duty cycle reach every motor. Loosely coupled, withLooselyCoupledFollowers: two separate motors joined only by the mechanism get the same setpoint and each runs its own PID. They are not mechanically locked, each has its own closed loop and config, vendors can be mixed, and voltage and duty cycle are not forwarded, so the leader moves alone."><figcaption><p>Tightly coupled followers share a shaft. Loosely coupled followers are separate motors.</p></figcaption></figure>

The two wheels here are separate motors on separate shafts, each given its own setpoint. If yours share a shaft, see [step 3](spin-and-followers.md).

## The code

{% tabs %}
{% tab title="Java, Commands v2" %}
Create `DoubleFlywheelSubsystem.java` in the `subsystems` folder.

{% code title="subsystems/DoubleFlywheelSubsystem.java" lineNumbers="true" %}
```java
package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Pounds;

import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.TalonFX;
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
import yams.core.motorcontrollers.remote.TalonFXWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class DoubleFlywheelSubsystem extends SubsystemBase {

  // Each flywheel gets its own config, so that each can be given its own speed.
  private final SmartMotorControllerConfig upperConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      // Feedback and feedforward constants, on the real robot and in simulation.
      .withClosedLoopController(0.1, 0, 0)
      .withSimClosedLoopController(0.1, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withSimFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      // Gearing from the motor rotor to the flywheel: 3:1.
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3)))
      // There is no FlyWheel mechanism to describe the wheel, so the simulation
      // gets the wheel's diameter and weight from here.
      .withMomentOfInertia(Inches.of(4), Pounds.of(1))
      .withMotorInverted(false)
      .withZeroPower(MotorMode.COAST)
      .withStatorCurrentLimit(Amps.of(40))
      .withTelemetry("UpperFlyWheel", TelemetryVerbosity.HIGH);

  // The lower flywheel is the same, except that its motor faces the other way.
  private final SmartMotorControllerConfig lowerConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withClosedLoopController(0.1, 0, 0)
      .withSimClosedLoopController(0.1, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withSimFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3)))
      .withMomentOfInertia(Inches.of(4), Pounds.of(1))
      .withMotorInverted(true)
      .withZeroPower(MotorMode.COAST)
      .withStatorCurrentLimit(Amps.of(40))
      .withTelemetry("LowerFlyWheel", TelemetryVerbosity.HIGH);

  // The vendor motor controller objects: two TalonFX, with CAN IDs 1 and 2.
  private final TalonFX upperTalon = new TalonFX(1, new CANBus(CANPort.CAN_S0));
  private final TalonFX lowerTalon = new TalonFX(2, new CANBus(CANPort.CAN_S0));

  // One SmartMotorController for each flywheel, each driving one Kraken X60.
  private final SmartMotorController upperMotor = new TalonFXWrapper(upperTalon, DCMotor.getKrakenX60(1), upperConfig);
  private final SmartMotorController lowerMotor = new TalonFXWrapper(lowerTalon, DCMotor.getKrakenX60(1), lowerConfig);

  public DoubleFlywheelSubsystem() {
    // A mechanism class would do this for us. Without one, set up the telemetry
    // (and live tuning) of each motor controller ourselves.
    upperMotor.setupTelemetry();
    lowerMotor.setupTelemetry();
  }

  @Override
  public void periodic() {
    // Publishes each flywheel's telemetry, and applies anything changed through live tuning.
    upperMotor.updateTelemetry();
    lowerMotor.updateTelemetry();
  }

  @Override
  public void simulationPeriodic() {
    // Runs each flywheel's physics simulation.
    upperMotor.simIterate();
    lowerMotor.simIterate();
  }
}
```
{% endcode %}
{% endtab %}

{% tab title="Java, Commands v3" %}
Create `DoubleFlywheelMechanism.java` in the `mechanisms` folder. Nothing calls a periodic method on a Commands v3 `Mechanism`, so create the mechanism in `Robot.java` and call its `periodic()` and `simulationPeriodic()` from `robotPeriodic()`. The lines added to `Robot.java` are in **bold**.

{% code title="mechanisms/DoubleFlywheelMechanism.java" lineNumbers="true" %}
```java
package first.robot.mechanisms;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Pounds;

import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.TalonFX;
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
import yams.core.motorcontrollers.remote.TalonFXWrapper;
import yams.core.telemetry.enums.TelemetryVerbosity;

public class DoubleFlywheelMechanism implements Mechanism {

  // Each flywheel gets its own config, so that each can be given its own speed.
  private final SmartMotorControllerConfig upperConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      // Feedback and feedforward constants, on the real robot and in simulation.
      .withClosedLoopController(0.1, 0, 0)
      .withSimClosedLoopController(0.1, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withSimFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      // Gearing from the motor rotor to the flywheel: 3:1.
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3)))
      // There is no FlyWheel mechanism to describe the wheel, so the simulation
      // gets the wheel's diameter and weight from here.
      .withMomentOfInertia(Inches.of(4), Pounds.of(1))
      .withMotorInverted(false)
      .withZeroPower(MotorMode.COAST)
      .withStatorCurrentLimit(Amps.of(40))
      .withTelemetry("UpperFlyWheel", TelemetryVerbosity.HIGH);

  // The lower flywheel is the same, except that its motor faces the other way.
  private final SmartMotorControllerConfig lowerConfig = new SmartMotorControllerConfig(this)
      .withControlMode(ControlMode.CLOSED_LOOP)
      .withClosedLoopController(0.1, 0, 0)
      .withSimClosedLoopController(0.1, 0, 0)
      .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withSimFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
      .withGearing(new MechanismGearing(GearBox.fromReductionStages(3)))
      .withMomentOfInertia(Inches.of(4), Pounds.of(1))
      .withMotorInverted(true)
      .withZeroPower(MotorMode.COAST)
      .withStatorCurrentLimit(Amps.of(40))
      .withTelemetry("LowerFlyWheel", TelemetryVerbosity.HIGH);

  // The vendor motor controller objects: two TalonFX, with CAN IDs 1 and 2.
  private final TalonFX upperTalon = new TalonFX(1, new CANBus(CANPort.CAN_S0));
  private final TalonFX lowerTalon = new TalonFX(2, new CANBus(CANPort.CAN_S0));

  // One SmartMotorController for each flywheel, each driving one Kraken X60.
  private final SmartMotorController upperMotor = new TalonFXWrapper(upperTalon, DCMotor.getKrakenX60(1), upperConfig);
  private final SmartMotorController lowerMotor = new TalonFXWrapper(lowerTalon, DCMotor.getKrakenX60(1), lowerConfig);

  public DoubleFlywheelMechanism() {
    // A mechanism class would do this for us. Without one, set up the telemetry
    // (and live tuning) of each motor controller ourselves.
    upperMotor.setupTelemetry();
    lowerMotor.setupTelemetry();
  }

  /** Call this from Robot.robotPeriodic(), before the scheduler runs. */
  public void periodic() {
    // Publishes each flywheel's telemetry, and applies anything changed through live tuning.
    upperMotor.updateTelemetry();
    lowerMotor.updateTelemetry();
  }

  /** Call this from Robot.robotPeriodic() in simulation. */
  public void simulationPeriodic() {
    // Runs each flywheel's physics simulation.
    upperMotor.simIterate();
    lowerMotor.simIterate();
  }
}
```
{% endcode %}

<pre class="language-java" data-title="Robot.java" data-line-numbers><code class="lang-java">package first.robot;

import org.wpilib.command3.Scheduler;
import org.wpilib.command3.button.CommandGamepad;
import org.wpilib.framework.OpModeRobot;
<strong>import org.wpilib.framework.RobotBase;
</strong>import first.robot.constants.DriverConstants;
<strong>import first.robot.mechanisms.DoubleFlywheelMechanism;
</strong>import first.robot.mechanisms.ExampleMechanism;

public class Robot extends OpModeRobot {
  final ExampleMechanism exampleMechanism = new ExampleMechanism();
<strong>  final DoubleFlywheelMechanism doubleFlywheel = new DoubleFlywheelMechanism();
</strong>  final CommandGamepad exampleController =
      new CommandGamepad(DriverConstants.DRIVER_CONTROLLER_PORT);

  public Robot() {}

  @Override
  public void robotPeriodic() {
<strong>    // Update the mechanism before the scheduler runs its commands.
</strong><strong>    doubleFlywheel.periodic();
</strong><strong>    if (RobotBase.isSimulation()) {
</strong><strong>      doubleFlywheel.simulationPeriodic();
</strong><strong>    }
</strong>    Scheduler.getDefault().run();
  }
}
</code></pre>
{% endtab %}

{% tab title="C++" %}
Create `DoubleFlywheelSubsystem.hpp` in `src/main/include/subsystems` and `DoubleFlywheelSubsystem.cpp` in `src/main/cpp/subsystems`.

YAMS keeps a pointer to each config and each `TalonFX`, so in C++ they are members of the subsystem, and each `SmartMotorController` is a `std::optional` that the constructor fills in. `WithMOI` is the C++ name of `withMomentOfInertia`.

{% code title="include/subsystems/DoubleFlywheelSubsystem.hpp" lineNumbers="true" %}
```cpp
#pragma once

#include <optional>

#include <ctre/phoenix6/TalonFX.hpp>
#include <wpi/commands2/SubsystemBase.hpp>

#include "yams/motorcontrollers/SmartMotorControllerConfig.hpp"
#include "yams/motorcontrollers/remote/TalonFXWrapper.hpp"

class DoubleFlywheelSubsystem : public wpi::cmd::SubsystemBase {
 public:
  DoubleFlywheelSubsystem();

  void Periodic() override;
  void SimulationPeriodic() override;

 private:
  // The vendor motor controller objects: two TalonFX, with CAN IDs 1 and 2.
  ctre::phoenix6::hardware::TalonFX m_upperTalon{1, ctre::phoenix6::CANBus{}};
  ctre::phoenix6::hardware::TalonFX m_lowerTalon{2, ctre::phoenix6::CANBus{}};

  // Each flywheel gets its own config, so that each can be given its own speed.
  yams::motorcontrollers::SmartMotorControllerConfig m_upperConfig;
  yams::motorcontrollers::SmartMotorControllerConfig m_lowerConfig;

  // One SmartMotorController for each flywheel, created in the constructor.
  std::optional<yams::motorcontrollers::remote::TalonFXWrapper> m_upperMotor;
  std::optional<yams::motorcontrollers::remote::TalonFXWrapper> m_lowerMotor;
};
```
{% endcode %}

{% code title="cpp/subsystems/DoubleFlywheelSubsystem.cpp" lineNumbers="true" %}
```cpp
#include "subsystems/DoubleFlywheelSubsystem.hpp"

#include <wpi/math/controller/SimpleMotorFeedforward.hpp>
#include <wpi/math/system/DCMotor.hpp>
#include <wpi/units/angle.hpp>
#include <wpi/units/current.hpp>
#include <wpi/units/length.hpp>
#include <wpi/units/mass.hpp>
#include <wpi/units/voltage.hpp>

#include "yams/gearing/GearBox.hpp"
#include "yams/gearing/MechanismGearing.hpp"

using namespace yams::gearing;
using Cfg = yams::motorcontrollers::SmartMotorControllerConfig;
using Feedforward = wpi::math::SimpleMotorFeedforward<wpi::units::turns>;

DoubleFlywheelSubsystem::DoubleFlywheelSubsystem() {
  // The feedforward constants of both flywheels: kS and kV.
  const Feedforward feedforward{0.1_V, wpi::units::unit_t<Feedforward::kv_unit>{0.12}};

  m_upperConfig
      .WithSubsystem(this)
      .WithClosedLoopMode()
      // Feedback and feedforward constants, on the real robot and in simulation.
      .WithFeedback(0.1, 0, 0)
      .WithSimClosedLoopController(0.1, 0, 0)
      .WithFeedforward(feedforward)
      // Gearing from the motor rotor to the flywheel: 3:1.
      .WithMotorGearing(MechanismGearing{GearBox::FromReductionStages({3.0})})
      // There is no FlyWheel mechanism to describe the wheel, so the simulation
      // gets the wheel's diameter and weight from here.
      .WithMOI(4_in, 1_lb)
      .WithMotorInverted(false)
      .WithZeroPower(Cfg::MotorMode::COAST)
      .WithStatorCurrentLimit(40_A)
      .WithTelemetry("UpperFlyWheel", Cfg::TelemetryVerbosity::HIGH);

  // The lower flywheel is the same, except that its motor faces the other way.
  m_lowerConfig
      .WithSubsystem(this)
      .WithClosedLoopMode()
      .WithFeedback(0.1, 0, 0)
      .WithSimClosedLoopController(0.1, 0, 0)
      .WithFeedforward(feedforward)
      .WithMotorGearing(MechanismGearing{GearBox::FromReductionStages({3.0})})
      .WithMOI(4_in, 1_lb)
      .WithMotorInverted(true)
      .WithZeroPower(Cfg::MotorMode::COAST)
      .WithStatorCurrentLimit(40_A)
      .WithTelemetry("LowerFlyWheel", Cfg::TelemetryVerbosity::HIGH);

  // One SmartMotorController for each flywheel, each driving one Kraken X60.
  m_upperMotor.emplace(&m_upperTalon, wpi::math::DCMotor::KrakenX60(1), &m_upperConfig);
  m_lowerMotor.emplace(&m_lowerTalon, wpi::math::DCMotor::KrakenX60(1), &m_lowerConfig);

  // A mechanism class would do this for us. Without one, set up the telemetry
  // (and live tuning) of each motor controller ourselves.
  m_upperMotor->SetupTelemetry();
  m_lowerMotor->SetupTelemetry();
}

void DoubleFlywheelSubsystem::Periodic() {
  // Publishes each flywheel's telemetry, and applies anything changed through live tuning.
  m_upperMotor->UpdateTelemetry();
  m_lowerMotor->UpdateTelemetry();
}

void DoubleFlywheelSubsystem::SimulationPeriodic() {
  // Runs each flywheel's physics simulation.
  m_upperMotor->SimIterate();
  m_lowerMotor->SimIterate();
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

## Read more

Every option used here has its own section in the `SmartMotorControllerConfig` pages:

* [Motor basics](../../details/editor/config/motor-basics.md): control mode, inversion, what the motor does at zero power
* [Gearing and units](../../details/editor/config/gearing-and-units.md): gearing, and how rotations become distance
* [Closed loop control](../../details/editor/config/closed-loop-control.md), [Feedforward](../../details/editor/config/feedforward.md) and [Motion profiles](../../details/editor/config/motion-profiles.md)
* [Limits and protection](../../details/editor/config/limits-and-protection.md): soft limits and current limits
* [Simulation](../../details/editor/config/simulation.md): the simulation only gains

The PID and feedforward numbers on this page are a place to start, not tuned values. Tune them on your own mechanism with [Live tuning](../../details/live-tuning/README.md).

Previous: [Double FlyWheel](../double-flywheel.md)

Next: [Step 2: Commands and buttons](commands.md)
