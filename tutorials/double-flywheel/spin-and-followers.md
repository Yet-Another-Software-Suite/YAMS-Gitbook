---
description: A spin ratio between the two flywheels, and what to do when they share a shaft.
---

# Step 3: Spin control and followers

## A spin ratio

Three presets are enough to start with. For finer control, let the caller choose a base speed and how much spin to put on it:

* A ratio of **0** runs both wheels at the base speed: no spin.
* A ratio of **1** runs the lower wheel 25% faster and the upper wheel 25% slower: the most backspin.
* A ratio of **-1** does the opposite: the most topspin.

The average of the two speeds stays at the base speed, so changing the spin does not change how fast the ball leaves.

{% tabs %}
{% tab title="Java, Commands v2" %}
{% code title="subsystems/DoubleFlywheelSubsystem.java" %}
```java
/**
 * Spin both flywheels around a base speed, with a spin ratio between them.
 *
 * @param baseSpeed The average speed of the two flywheels.
 * @param spinRatio From -1 (the most topspin) to 1 (the most backspin). 0 is no spin.
 */
public Command setSpinShot(AngularVelocity baseSpeed, double spinRatio) {
  double ratio = Math.max(-1.0, Math.min(1.0, spinRatio));
  // At a ratio of 1 the lower flywheel runs 25% faster and the upper one 25% slower.
  double upperMultiplier = 1.0 - ratio * 0.25;
  double lowerMultiplier = 1.0 + ratio * 0.25;
  return setFlywheelSpeeds(baseSpeed.times(upperMultiplier), baseSpeed.times(lowerMultiplier));
}
```
{% endcode %}
{% endtab %}

{% tab title="Java, Commands v3" %}
{% code title="mechanisms/DoubleFlywheelMechanism.java" %}
```java
/**
 * Spin both flywheels around a base speed, with a spin ratio between them.
 *
 * @param baseSpeed The average speed of the two flywheels.
 * @param spinRatio From -1 (the most topspin) to 1 (the most backspin). 0 is no spin.
 */
public Command setSpinShot(AngularVelocity baseSpeed, double spinRatio) {
  double ratio = Math.max(-1.0, Math.min(1.0, spinRatio));
  // At a ratio of 1 the lower flywheel runs 25% faster and the upper one 25% slower.
  double upperMultiplier = 1.0 - ratio * 0.25;
  double lowerMultiplier = 1.0 + ratio * 0.25;
  return setFlywheelSpeeds(baseSpeed.times(upperMultiplier), baseSpeed.times(lowerMultiplier));
}
```
{% endcode %}
{% endtab %}

{% tab title="C++" %}
{% code title="include/subsystems/DoubleFlywheelSubsystem.hpp" %}
```cpp
/**
 * Spin both flywheels around a base speed, with a spin ratio between them.
 *
 * @param baseSpeed The average speed of the two flywheels.
 * @param spinRatio From -1 (the most topspin) to 1 (the most backspin). 0 is no spin.
 */
wpi::cmd::CommandPtr SetSpinShot(wpi::units::revolutions_per_minute_t baseSpeed, double spinRatio);
```
{% endcode %}

{% code title="cpp/subsystems/DoubleFlywheelSubsystem.cpp" %}
```cpp
wpi::cmd::CommandPtr DoubleFlywheelSubsystem::SetSpinShot(wpi::units::revolutions_per_minute_t baseSpeed,
                                                          double spinRatio) {
  double ratio = std::clamp(spinRatio, -1.0, 1.0);
  // At a ratio of 1 the lower flywheel runs 25% faster and the upper one 25% slower.
  double upperMultiplier = 1.0 - ratio * 0.25;
  double lowerMultiplier = 1.0 + ratio * 0.25;
  return SetFlywheelSpeeds(baseSpeed * upperMultiplier, baseSpeed * lowerMultiplier);
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

**For your robot:** the 25% is how much the two wheels may differ at most. Raise it for more spin.

### The complete file

{% tabs %}
{% tab title="Java, Commands v2" %}
The highlighted lines are new in this step.

<pre class="language-java" data-title="subsystems/DoubleFlywheelSubsystem.java" data-line-numbers><code class="lang-java">package first.robot.subsystems;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Pounds;
import static org.wpilib.units.Units.RPM;

import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.TalonFX;
import org.wpilib.command2.Command;
import org.wpilib.command2.SubsystemBase;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import org.wpilib.math.system.DCMotor;
import org.wpilib.units.measure.AngularVelocity;
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

  /** Spin both flywheels. One command sets both speeds, so the two always change together. */
  public Command setFlywheelSpeeds(AngularVelocity upperSpeed, AngularVelocity lowerSpeed) {
    return run(() -> {
      upperMotor.setVelocity(upperSpeed);
      lowerMotor.setVelocity(lowerSpeed);
    }).withName("Set FlyWheel Speeds");
  }

  /** Both flywheels at the same speed: no spin. */
  public Command neutralSpinShot() {
    return setFlywheelSpeeds(RPM.of(3000), RPM.of(3000));
  }

  /** The lower flywheel faster: backspin. */
  public Command backspinShot() {
    return setFlywheelSpeeds(RPM.of(2500), RPM.of(3500));
  }

  /** The upper flywheel faster: topspin. */
  public Command topspinShot() {
    return setFlywheelSpeeds(RPM.of(3500), RPM.of(2500));
  }

  /** Bring both flywheels to a stop. */
  public Command stopFlywheels() {
    return setFlywheelSpeeds(RPM.of(0), RPM.of(0));
  }

  /** True when both flywheels are within the tolerance of their target speeds. */
  public boolean atTargetVelocity(AngularVelocity upperTarget, AngularVelocity lowerTarget, AngularVelocity tolerance) {
    return upperMotor.getMechanismVelocity().isNear(upperTarget, tolerance)
        &#x26;&#x26; lowerMotor.getMechanismVelocity().isNear(lowerTarget, tolerance);
  }

<strong>  /**
</strong><strong>   * Spin both flywheels around a base speed, with a spin ratio between them.
</strong><strong>   *
</strong><strong>   * @param baseSpeed The average speed of the two flywheels.
</strong><strong>   * @param spinRatio From -1 (the most topspin) to 1 (the most backspin). 0 is no spin.
</strong><strong>   */
</strong><strong>  public Command setSpinShot(AngularVelocity baseSpeed, double spinRatio) {
</strong><strong>    double ratio = Math.max(-1.0, Math.min(1.0, spinRatio));
</strong><strong>    // At a ratio of 1 the lower flywheel runs 25% faster and the upper one 25% slower.
</strong><strong>    double upperMultiplier = 1.0 - ratio * 0.25;
</strong><strong>    double lowerMultiplier = 1.0 + ratio * 0.25;
</strong><strong>    return setFlywheelSpeeds(baseSpeed.times(upperMultiplier), baseSpeed.times(lowerMultiplier));
</strong><strong>  }
</strong>
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
</code></pre>
{% endtab %}

{% tab title="Java, Commands v3" %}
The highlighted lines are new in this step.

<pre class="language-java" data-title="mechanisms/DoubleFlywheelMechanism.java" data-line-numbers><code class="lang-java">package first.robot.mechanisms;

import static org.wpilib.units.Units.Amps;
import static org.wpilib.units.Units.Inches;
import static org.wpilib.units.Units.Pounds;
import static org.wpilib.units.Units.RPM;

import com.ctre.phoenix6.CANBus;
import com.ctre.phoenix6.hardware.TalonFX;
import org.wpilib.command3.Command;
import org.wpilib.command3.Mechanism;
import org.wpilib.hardware.bus.CANPort;
import org.wpilib.math.controller.SimpleMotorFeedforward;
import org.wpilib.math.system.DCMotor;
import org.wpilib.units.measure.AngularVelocity;
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

  /** Spin both flywheels. One command sets both speeds, so the two always change together. */
  public Command setFlywheelSpeeds(AngularVelocity upperSpeed, AngularVelocity lowerSpeed) {
    return run(coroutine -> {
      while (true) {
        upperMotor.setVelocity(upperSpeed);
        lowerMotor.setVelocity(lowerSpeed);
        coroutine.yield();
      }
    }).named("Set FlyWheel Speeds");
  }

  /** Both flywheels at the same speed: no spin. */
  public Command neutralSpinShot() {
    return setFlywheelSpeeds(RPM.of(3000), RPM.of(3000));
  }

  /** The lower flywheel faster: backspin. */
  public Command backspinShot() {
    return setFlywheelSpeeds(RPM.of(2500), RPM.of(3500));
  }

  /** The upper flywheel faster: topspin. */
  public Command topspinShot() {
    return setFlywheelSpeeds(RPM.of(3500), RPM.of(2500));
  }

  /** Bring both flywheels to a stop. */
  public Command stopFlywheels() {
    return setFlywheelSpeeds(RPM.of(0), RPM.of(0));
  }

  /** True when both flywheels are within the tolerance of their target speeds. */
  public boolean atTargetVelocity(AngularVelocity upperTarget, AngularVelocity lowerTarget, AngularVelocity tolerance) {
    return upperMotor.getMechanismVelocity().isNear(upperTarget, tolerance)
        &#x26;&#x26; lowerMotor.getMechanismVelocity().isNear(lowerTarget, tolerance);
  }

<strong>  /**
</strong><strong>   * Spin both flywheels around a base speed, with a spin ratio between them.
</strong><strong>   *
</strong><strong>   * @param baseSpeed The average speed of the two flywheels.
</strong><strong>   * @param spinRatio From -1 (the most topspin) to 1 (the most backspin). 0 is no spin.
</strong><strong>   */
</strong><strong>  public Command setSpinShot(AngularVelocity baseSpeed, double spinRatio) {
</strong><strong>    double ratio = Math.max(-1.0, Math.min(1.0, spinRatio));
</strong><strong>    // At a ratio of 1 the lower flywheel runs 25% faster and the upper one 25% slower.
</strong><strong>    double upperMultiplier = 1.0 - ratio * 0.25;
</strong><strong>    double lowerMultiplier = 1.0 + ratio * 0.25;
</strong><strong>    return setFlywheelSpeeds(baseSpeed.times(upperMultiplier), baseSpeed.times(lowerMultiplier));
</strong><strong>  }
</strong>
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
</code></pre>
{% endtab %}

{% tab title="C++" %}
The highlighted lines are new in this step.

<pre class="language-cpp" data-title="include/subsystems/DoubleFlywheelSubsystem.hpp" data-line-numbers><code class="lang-cpp">#pragma once

#include &#x3C;optional>

#include &#x3C;ctre/phoenix6/TalonFX.hpp>
#include &#x3C;wpi/commands2/CommandPtr.hpp>
#include &#x3C;wpi/commands2/SubsystemBase.hpp>
#include &#x3C;wpi/units/angular_velocity.hpp>

#include "yams/motorcontrollers/SmartMotorControllerConfig.hpp"
#include "yams/motorcontrollers/remote/TalonFXWrapper.hpp"

class DoubleFlywheelSubsystem : public wpi::cmd::SubsystemBase {
 public:
  DoubleFlywheelSubsystem();

  /** Spin both flywheels. One command sets both speeds, so the two always change together. */
  wpi::cmd::CommandPtr SetFlywheelSpeeds(wpi::units::revolutions_per_minute_t upperSpeed,
                                         wpi::units::revolutions_per_minute_t lowerSpeed);

  /** Both flywheels at the same speed: no spin. */
  wpi::cmd::CommandPtr NeutralSpinShot();

  /** The lower flywheel faster: backspin. */
  wpi::cmd::CommandPtr BackspinShot();

  /** The upper flywheel faster: topspin. */
  wpi::cmd::CommandPtr TopspinShot();

  /** Bring both flywheels to a stop. */
  wpi::cmd::CommandPtr StopFlywheels();

  /** True when both flywheels are within the tolerance of their target speeds. */
  bool AtTargetVelocity(wpi::units::revolutions_per_minute_t upperTarget,
                        wpi::units::revolutions_per_minute_t lowerTarget,
                        wpi::units::revolutions_per_minute_t tolerance);

<strong>  /**
</strong><strong>   * Spin both flywheels around a base speed, with a spin ratio between them.
</strong><strong>   *
</strong><strong>   * @param baseSpeed The average speed of the two flywheels.
</strong><strong>   * @param spinRatio From -1 (the most topspin) to 1 (the most backspin). 0 is no spin.
</strong><strong>   */
</strong><strong>  wpi::cmd::CommandPtr SetSpinShot(wpi::units::revolutions_per_minute_t baseSpeed, double spinRatio);
</strong>
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
  std::optional&#x3C;yams::motorcontrollers::remote::TalonFXWrapper> m_upperMotor;
  std::optional&#x3C;yams::motorcontrollers::remote::TalonFXWrapper> m_lowerMotor;
};
</code></pre>

<pre class="language-cpp" data-title="cpp/subsystems/DoubleFlywheelSubsystem.cpp" data-line-numbers><code class="lang-cpp">#include "subsystems/DoubleFlywheelSubsystem.hpp"

<strong>#include &#x3C;algorithm>
</strong>#include &#x3C;wpi/math/controller/SimpleMotorFeedforward.hpp>
#include &#x3C;wpi/math/system/DCMotor.hpp>
#include &#x3C;wpi/units/angle.hpp>
#include &#x3C;wpi/units/current.hpp>
#include &#x3C;wpi/units/length.hpp>
#include &#x3C;wpi/units/mass.hpp>
#include &#x3C;wpi/units/math.hpp>
#include &#x3C;wpi/units/voltage.hpp>

#include "yams/gearing/GearBox.hpp"
#include "yams/gearing/MechanismGearing.hpp"

using namespace yams::gearing;
using Cfg = yams::motorcontrollers::SmartMotorControllerConfig;
using Feedforward = wpi::math::SimpleMotorFeedforward&#x3C;wpi::units::turns>;

DoubleFlywheelSubsystem::DoubleFlywheelSubsystem() {
  // The feedforward constants of both flywheels: kS and kV.
  const Feedforward feedforward{0.1_V, wpi::units::unit_t&#x3C;Feedforward::kv_unit>{0.12}};

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
  m_upperMotor.emplace(&#x26;m_upperTalon, wpi::math::DCMotor::KrakenX60(1), &#x26;m_upperConfig);
  m_lowerMotor.emplace(&#x26;m_lowerTalon, wpi::math::DCMotor::KrakenX60(1), &#x26;m_lowerConfig);

  // A mechanism class would do this for us. Without one, set up the telemetry
  // (and live tuning) of each motor controller ourselves.
  m_upperMotor->SetupTelemetry();
  m_lowerMotor->SetupTelemetry();
}

wpi::cmd::CommandPtr DoubleFlywheelSubsystem::SetFlywheelSpeeds(
    wpi::units::revolutions_per_minute_t upperSpeed, wpi::units::revolutions_per_minute_t lowerSpeed) {
  return Run([this, upperSpeed, lowerSpeed] {
           m_upperMotor->SetVelocity(wpi::units::turns_per_second_t{upperSpeed});
           m_lowerMotor->SetVelocity(wpi::units::turns_per_second_t{lowerSpeed});
         })
      .WithName("Set FlyWheel Speeds");
}

wpi::cmd::CommandPtr DoubleFlywheelSubsystem::NeutralSpinShot() {
  return SetFlywheelSpeeds(3000_rpm, 3000_rpm);
}

wpi::cmd::CommandPtr DoubleFlywheelSubsystem::BackspinShot() {
  return SetFlywheelSpeeds(2500_rpm, 3500_rpm);
}

wpi::cmd::CommandPtr DoubleFlywheelSubsystem::TopspinShot() {
  return SetFlywheelSpeeds(3500_rpm, 2500_rpm);
}

wpi::cmd::CommandPtr DoubleFlywheelSubsystem::StopFlywheels() {
  return SetFlywheelSpeeds(0_rpm, 0_rpm);
}

bool DoubleFlywheelSubsystem::AtTargetVelocity(wpi::units::revolutions_per_minute_t upperTarget,
                                               wpi::units::revolutions_per_minute_t lowerTarget,
                                               wpi::units::revolutions_per_minute_t tolerance) {
  return wpi::units::math::abs(m_upperMotor->GetMechanismVelocity() - upperTarget) &#x3C;= tolerance &#x26;&#x26;
         wpi::units::math::abs(m_lowerMotor->GetMechanismVelocity() - lowerTarget) &#x3C;= tolerance;
}

<strong>wpi::cmd::CommandPtr DoubleFlywheelSubsystem::SetSpinShot(wpi::units::revolutions_per_minute_t baseSpeed,
</strong><strong>                                                          double spinRatio) {
</strong><strong>  double ratio = std::clamp(spinRatio, -1.0, 1.0);
</strong><strong>  // At a ratio of 1 the lower flywheel runs 25% faster and the upper one 25% slower.
</strong><strong>  double upperMultiplier = 1.0 - ratio * 0.25;
</strong><strong>  double lowerMultiplier = 1.0 + ratio * 0.25;
</strong><strong>  return SetFlywheelSpeeds(baseSpeed * upperMultiplier, baseSpeed * lowerMultiplier);
</strong><strong>}
</strong>
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
</code></pre>
{% endtab %}
{% endtabs %}

## Loosely coupled followers

If you want both wheels to always get the **same** setpoint, you do not need to command each one yourself. Make one a loosely coupled follower of the other, and command only the leader:

{% tabs %}
{% tab title="Java, Commands v2" %}
```java
SmartMotorControllerConfig lowerConfig = new SmartMotorControllerConfig(this)
    // ... config options
    ;

SmartMotorController lowerMotor = new TalonFXWrapper(lowerTalon, DCMotor.getKrakenX60(1), lowerConfig);

SmartMotorControllerConfig upperConfig = new SmartMotorControllerConfig(this)
    // ... config options
    .withLooselyCoupledFollowers(lowerMotor); // the lower motor is given the upper motor's setpoints

SmartMotorController upperMotor = new TalonFXWrapper(upperTalon, DCMotor.getKrakenX60(1), upperConfig);
```
{% endtab %}

{% tab title="Java, Commands v3" %}
The same as Commands v2, with `SmartMotorControllerConfig` from `yams.commands3.config`.

```java
SmartMotorControllerConfig lowerConfig = new SmartMotorControllerConfig(this)
    // ... config options
    ;

SmartMotorController lowerMotor = new TalonFXWrapper(lowerTalon, DCMotor.getKrakenX60(1), lowerConfig);

SmartMotorControllerConfig upperConfig = new SmartMotorControllerConfig(this)
    // ... config options
    .withLooselyCoupledFollowers(lowerMotor); // the lower motor is given the upper motor's setpoints

SmartMotorController upperMotor = new TalonFXWrapper(upperTalon, DCMotor.getKrakenX60(1), upperConfig);
```
{% endtab %}

{% tab title="C++" %}
In the constructor, create the lower motor first, then name it in the upper motor's config:

```cpp
m_lowerMotor.emplace(&m_lowerTalon, wpi::math::DCMotor::KrakenX60(1), &m_lowerConfig);

m_upperConfig
    // ... config options
    .WithLooselyCoupledFollowers({&m_lowerMotor.value()});  // the lower motor is given the upper motor's setpoints

m_upperMotor.emplace(&m_upperTalon, wpi::math::DCMotor::KrakenX60(1), &m_upperConfig);
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
Loosely coupled followers are only given velocity and position requests, **NOT** duty cycle or voltage requests. Command each `SmartMotorController` yourself when you need fully independent control, as the rest of this tutorial does.
{% endhint %}

[Followers](../../details/editor/config/followers.md) explains the difference between loosely and tightly coupled followers.

## Summary

You now have a double flywheel shooter with:

* Independent velocity control of the upper and lower flywheels
* Preset shots for no spin, backspin and topspin, and a spin ratio for anything in between
* Full simulation support
* Telemetry and live tuning for both flywheels

The differences from a single flywheel are:

1. Two `SmartMotorControllerConfig`s and two `SmartMotorController`s, controlled directly instead of through a `FlyWheel` mechanism
2. The subsystem calls `setupTelemetry()`, `updateTelemetry()` and `simIterate()` on each `SmartMotorController` itself
3. One command sets both velocity setpoints

Previous: [Step 2: Commands and buttons](commands.md)

Next: [Double FlyWheel](../double-flywheel.md)
