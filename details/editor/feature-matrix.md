---
icon: table-cells
---

# Motor controller feature matrix

This page provides a comprehensive comparison of supported `SmartMotorControllerConfig` features across all motor controller wrappers in YAMS.

## Supported motor controllers

| Wrapper           | Vendor          | Hardware                                         | Java Package                        | C++ Header                                         |
| ----------------- | --------------- | ------------------------------------------------ | ----------------------------------- | -------------------------------------------------- |
| `TalonFXWrapper`  | CTRE            | TalonFX (Kraken X60, Falcon 500)                 | `yams.core.motorcontrollers.remote` | `yams/motorcontrollers/remote/TalonFXWrapper.hpp`  |
| `TalonFXSWrapper` | CTRE            | TalonFXS (controls NEO, NEO 550, Vortex, Minion) | `yams.core.motorcontrollers.remote` | `yams/motorcontrollers/remote/TalonFXSWrapper.hpp` |
| `SparkWrapper`    | REV             | SparkMax, SparkFlex                              | `yams.core.motorcontrollers.local`  | `yams/motorcontrollers/local/SparkWrapper.hpp`     |
| `NovaWrapper`     | The Thrifty Bot | Thrifty Nova (NEO, Minion, Pulsar, brushed)      | `yams.core.motorcontrollers.local`  | `yams/motorcontrollers/local/NovaWrapper.hpp`      |

{% hint style="info" %}
The wrappers live in `yams.core`, but you build them with the `SmartMotorControllerConfig` from your command framework's package: `yams.commands2.config.SmartMotorControllerConfig` (Commands v2) or `yams.commands3.config.SmartMotorControllerConfig` (Commands v3).
{% endhint %}

## Vendor configuration classes

Each wrapper accepts a specific vendor configuration class via `SmartMotorControllerConfig.withVendorConfig()`. This allows you to pass through vendor-specific settings that YAMS doesn't directly expose.

| Wrapper                    | Accepted Vendor Config Class | Package                        |
| -------------------------- | ---------------------------- | ------------------------------ |
| `TalonFXWrapper`           | `TalonFXConfiguration`       | `com.ctre.phoenix6.configs`    |
| `TalonFXSWrapper`          | `TalonFXSConfiguration`      | `com.ctre.phoenix6.configs`    |
| `SparkWrapper` (SparkMax)  | `SparkMaxConfig`             | `com.revrobotics.spark.config` |
| `SparkWrapper` (SparkFlex) | `SparkFlexConfig`            | `com.revrobotics.spark.config` |
| `NovaWrapper`              | `NovaConfig.Config`, such as `NovaConfigBatch` | `com.thrifty.nova`  |

{% hint style="warning" %}
**SparkWrapper Note**: The vendor config class must match the actual hardware. Use `SparkMaxConfig` for `SparkMax` controllers and `SparkFlexConfig` for `SparkFlex` controllers. Using the wrong config type will throw a `SmartMotorControllerConfigurationException`.
{% endhint %}

{% hint style="warning" %}
**NovaWrapper Note**: The vendor config must be a `NovaConfig.Config`, usually a `NovaConfigBatch` (in C++ it must be a `thrifty::NovaConfigBatch`). Anything else throws a `SmartMotorControllerConfigurationException`. The vendor config is sent to the Nova before the YAMS config, which overrides it.
{% endhint %}

### Vendor config examples

#### TalonFX with TalonFXConfiguration

```java
import com.ctre.phoenix6.configs.TalonFXConfiguration;
import com.ctre.phoenix6.signals.NeutralModeValue;

TalonFXConfiguration vendorConfig = new TalonFXConfiguration();
// Configure vendor-specific settings not exposed by YAMS
vendorConfig.Audio.BeepOnBoot = false;
vendorConfig.Audio.BeepOnConfig = false;

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withVendorConfig(vendorConfig)  // Must be TalonFXConfiguration
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(1, 0, 0)
    // ... other YAMS config
    ;

SmartMotorController smc = new TalonFXWrapper(talonFX, DCMotor.getKrakenX60(1), config);
```

#### TalonFXS with TalonFXSConfiguration

```java
import com.ctre.phoenix6.configs.TalonFXSConfiguration;
import com.ctre.phoenix6.signals.MotorArrangementValue;

TalonFXSConfiguration vendorConfig = new TalonFXSConfiguration();
// Note: Motor arrangement is auto-detected by YAMS based on DCMotor type
// but you can configure other vendor-specific settings here

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withVendorConfig(vendorConfig)  // Must be TalonFXSConfiguration
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(0.5, 0, 0)
    // ... other YAMS config
    ;

SmartMotorController smc = new TalonFXSWrapper(talonFXS, DCMotor.getNEO(1), config);
```

#### SparkMax with SparkMaxConfig

```java
import com.revrobotics.spark.config.SparkMaxConfig;
import com.revrobotics.spark.SparkMax;
import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.util.CANPorts;

SparkMaxConfig vendorConfig = new SparkMaxConfig();
// Configure vendor-specific settings not exposed by YAMS
vendorConfig.signals.primaryEncoderPositionPeriodMs(10);
vendorConfig.signals.primaryEncoderVelocityPeriodMs(20);

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withVendorConfig(vendorConfig)  // Must be SparkMaxConfig for SparkMax
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(1, 0, 0)
    // ... other YAMS config
    ;

SparkMax sparkMax = new SparkMax(CANPorts.fromBusId(1), 1, MotorType.kBrushless);
SmartMotorController smc = new SparkWrapper(sparkMax, DCMotor.getNEO(1), config);
```

#### SparkFlex with SparkFlexConfig

```java
import com.revrobotics.spark.config.SparkFlexConfig;
import com.revrobotics.spark.SparkFlex;
import com.revrobotics.spark.SparkLowLevel.MotorType;
import com.revrobotics.util.CANPorts;

SparkFlexConfig vendorConfig = new SparkFlexConfig();
// Configure vendor-specific settings not exposed by YAMS

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withVendorConfig(vendorConfig)  // Must be SparkFlexConfig for SparkFlex
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(1, 0, 0)
    // ... other YAMS config
    ;

SparkFlex sparkFlex = new SparkFlex(CANPorts.fromBusId(1), 1, MotorType.kBrushless);
SmartMotorController smc = new SparkWrapper(sparkFlex, DCMotor.getNeoVortex(1), config);
```

#### Thrifty Nova with NovaConfigBatch

```java
import com.thrifty.core.Motor.MotorType;
import com.thrifty.nova.Nova;
import com.thrifty.nova.NovaConfig;
import com.thrifty.nova.NovaConfigBatch;

// Configure vendor-specific settings not exposed by YAMS
NovaConfigBatch vendorConfig = new NovaConfigBatch(NovaConfig.tempThrottleEnable(true));

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withVendorConfig(vendorConfig)  // Must be a NovaConfig.Config, such as NovaConfigBatch
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(0.2, 0, 0)
    // ... other YAMS config
    ;

Nova nova = new Nova(0, 3, MotorType.NEO); // CAN bus 0, CAN ID 3
SmartMotorController smc = new NovaWrapper(nova, DCMotor.getNEO(1), config);
```

{% hint style="info" %}
**Configuration Priority**: Settings in `SmartMotorControllerConfig` always take precedence over vendor config settings. The vendor config is applied first as a base, then YAMS overwrites any settings it manages directly.
{% endhint %}

## Feature support matrix

This is every option against every motor controller. What each option does, with a graphic and an example, is on the [SmartMotorControllerConfig](config/README.md) pages.

### Legend

| Symbol | Meaning                         |
| ------ | ------------------------------- |
| ✅      | Fully Supported                 |
| ⚠️     | Partially Supported (see notes) |
| ❌      | Not Supported                   |

***

### Basic configuration

| Feature                   | TalonFX | TalonFXS | SparkMax/Flex | Nova |
| ------------------------- | ------- | -------- | ------------- | ---- |
| Motor Inversion           | ✅       | ✅        | ✅             | ✅ |
| Encoder Inversion         | ❌       | ✅        | ❌             | ❌ |
| Zero Power Mode (Brake/Coast)   | ✅       | ✅        | ✅             | ✅ |
| Reset Previous Config     | ✅       | ✅        | ✅             | ✅ |
| Vendor Config Passthrough | ✅       | ✅        | ✅             | ✅ |
| Starting Position         | ✅       | ✅        | ✅             | ✅ |

### Current limits

| Feature              | TalonFX | TalonFXS | SparkMax/Flex | Nova |
| -------------------- | ------- | -------- | ------------- | ---- |
| Stator Current Limit | ✅       | ✅        | ✅             | ✅ |
| Supply Current Limit | ✅       | ✅        | ❌             | ✅ |

### Voltage configuration

| Feature                 | TalonFX | TalonFXS | SparkMax/Flex | Nova |
| ----------------------- | ------- | -------- | ------------- | ---- |
| Voltage Compensation    | ❌       | ❌        | ✅             | ✅ |
| Closed Loop Max Voltage | ✅       | ✅        | ⚠️ (requires an Exponential Profile or LQR controller) | ✅ |

{% hint style="warning" %}
**Voltage Compensation**: `TalonFXWrapper` and `TalonFXSWrapper` throw an exception if voltage compensation is configured; it's supported on `SparkWrapper` and `NovaWrapper`.
{% endhint %}

{% hint style="warning" %}
**Closed Loop Max Voltage on Sparks**: `SparkWrapper` only supports `withClosedLoopControllerMaximumVoltage` when the closed loop controller runs on the SystemCore (an Exponential Profile or LQR controller); otherwise it throws a `SmartMotorControllerConfigurationException`.
{% endhint %}

### Ramp rates

| Feature               | TalonFX | TalonFXS | SparkMax/Flex | Nova |
| --------------------- | ------- | -------- | ------------- | ---- |
| Open Loop Ramp Rate   | ✅       | ✅        | ✅             | ⚠️ (one ramp rate) |
| Closed Loop Ramp Rate | ✅       | ✅        | ✅             | ⚠️ (one ramp rate) |

{% hint style="warning" %}
**Nova Ramp Rate**: The Thrifty Nova has one ramp rate for both open and closed loop control. `NovaWrapper` throws a `SmartMotorControllerConfigurationException` if `withOpenLoopRampRate` and `withClosedLoopRampRate` are given different values; set one of them, or both to the same value.
{% endhint %}

### Gearing

| Feature                          | TalonFX | TalonFXS | SparkMax/Flex | Nova |
| -------------------------------- | ------- | -------- | ------------- | ---- |
| Mechanism Gearing                | ✅       | ✅        | ✅             | ✅ |
| External Encoder Gearing         | ✅       | ✅        | ✅             | ✅ |
| Mechanism Circumference (Linear) | ✅       | ✅        | ✅             | ✅ |

### Closed-loop control

| Feature                 | TalonFX      | TalonFXS     | SparkMax/Flex                        | Nova |
| ----------------------- | ------------ | ------------ | ------------------------------------ | ---- |
| PID Controller          | ✅               | ✅               | ✅                                                      | ⚠️ (SystemCore)                 |
| LQR Controller          | ⚠️ (SystemCore) | ⚠️ (SystemCore) | ⚠️ (SystemCore)                                        | ⚠️ (SystemCore)                 |
| Closed Loop Tolerance   | ❌               | ❌               | ✅ (not with LQR)                                       | ✅ (not with LQR)                |
| Control Period Override | ❌               | ❌               | ⚠️ (requires an Exponential Profile or LQR controller) | ⚠️ (requires closed loop mode) |
| Closed Loop Slots       | ⚠️ (`SLOT_0` to `SLOT_2`) | ⚠️ (`SLOT_0` to `SLOT_2`) | ✅ | ✅ |

{% hint style="warning" %}
**LQR Controller Execution**: LQR is **not natively supported** by any motor controller hardware. When configured, the closed loop control runs on the **SystemCore**: the wrapper starts a `Notifier` that calls `SmartMotorController.iterateClosedLoopController()` every closed loop control period (20 ms unless you set `withClosedLoopControlPeriod`), so there is nothing to call yourself. See [LQR Controllers](lqr-controllers.md) for details.
{% endhint %}

{% hint style="info" %}
**Nova Closed Loop Execution**: `NovaWrapper` always runs its closed loop controller (PID or LQR, motion profiles and feedforwards) on the SystemCore and commands the Nova with voltage. ThriftyLib's simulation ignores the Nova's onboard PID gains, so this keeps simulation and the real robot the same. A YAMS alert reports that the closed loop controller is running on the SystemCore.
{% endhint %}

### Motion profiles

| Feature             | TalonFX               | TalonFXS              | SparkMax/Flex | Nova |
| ------------------- | --------------------- | --------------------- | ------------- | ---- |
| Trapezoidal Profile | ✅ (Motion Magic)      | ✅ (Motion Magic)      | ✅ (MAXMotion) | ⚠️ (SystemCore) |
| Exponential Profile | ✅ (Motion Magic Expo) | ✅ (Motion Magic Expo) | ⚠️ (SystemCore) | ⚠️ (SystemCore) |

{% hint style="info" %}
**Profile Execution Location**:

* **TalonFX/TalonFXS**: Profiles run on the motor controller (Motion Magic)
* **SparkMax/SparkFlex**: Trapezoidal profiles run on motor controller (MAXMotion), exponential profiles run on the SystemCore
* **Thrifty Nova**: Both profiles run on the SystemCore
{% endhint %}

### Feedforward

| Feature                | TalonFX | TalonFXS | SparkMax/Flex | Nova |
| ---------------------- | ------- | -------- | ------------- | ---- |
| SimpleMotorFeedforward | ✅       | ✅        | ✅             | ✅ |
| ArmFeedforward         | ✅       | ✅        | ✅             | ✅ |
| ElevatorFeedforward    | ✅       | ✅        | ✅             | ✅ |
| Simulation Feedforward | ✅       | ✅        | ✅             | ✅ |

### Soft Limits

| Feature              | TalonFX | TalonFXS | SparkMax/Flex | Nova |
| -------------------- | ------- | -------- | ------------- | ---- |
| Angle Soft Limits    | ✅       | ✅        | ✅             | ✅ |
| Distance Soft Limits | ✅       | ✅        | ✅             | ✅ |
| Continuous Wrapping  | ✅       | ✅        | ✅             | ✅ |

### External encoders

| Feature                      | TalonFX | TalonFXS | SparkMax/Flex                                          | Nova |
| ---------------------------- | ------- | -------- | ------------------------------------------------------ | ---- |
| External Encoder Support     | ✅       | ✅        | ✅                                                      | ✅ |
| External Encoder Inversion   | ✅       | ✅        | ✅                                                      | ✅ |
| External Encoder Zero Offset | ✅       | ✅        | ✅                                                      | ⚠️ (absolute and CAN encoders only) |
| Use External for Feedback    | ✅       | ✅        | ✅                                                      | ✅ |
| Feedback Synchronization     | ❌       | ❌        | ⚠️ (requires an Exponential Profile or LQR controller) | ⚠️ (requires closed loop mode) |

**Supported External Encoder Types:**

| Wrapper       | Supported Encoders                             |
| ------------- | ---------------------------------------------- |
| TalonFX       | `CANcoder`, `CANdi` (PWM1/PWM2)                |
| TalonFXS      | `CANcoder`, `CANdi` (PWM1/PWM2)                |
| SparkMax/Flex | `SparkAbsoluteEncoder` (Through Bore, REV Hex) |
| Nova          | `FeedbackSensorType.ABS` (absolute encoder on the data port), `FeedbackSensorType.QUAD` (quadrature encoder on the data port), `CanEncoder` (Thrifty CAN Encoder) |

{% hint style="info" %}
**Nova external encoders**: pass `FeedbackSensorType.ABS` or `FeedbackSensorType.QUAD` (from `com.thrifty.core.Motor`) to `withExternalEncoder`. The encoder's type comes from the vendor config, for example `NovaConfig.absoluteEncoderType(Motor.AbsoluteEncoderType.REV_ENCODER)` or `NovaConfig.quadratureEncoderType(Motor.QuadratureEncoderType.custom(2048))`. Zero offset and discontinuity point only apply to absolute and CAN encoders.
{% endhint %}

### Follower motors

| Feature                   | TalonFX | TalonFXS | SparkMax/Flex | Nova |
| ------------------------- | ------- | -------- | ------------- | ---- |
| Hardware Followers        | ✅       | ✅        | ✅             | ✅ |
| Follower Inversion        | ✅       | ✅        | ✅             | ✅ |
| Loosely Coupled Followers | ✅       | ✅        | ✅             | ✅ |

{% hint style="warning" %}
**Follower Compatibility**: Followers must be the same brand as the leader motor controller. You cannot mix vendors for hardware followers.
{% endhint %}

### Telemetry

| Feature                 | TalonFX | TalonFXS | SparkMax/Flex | Nova |
| ----------------------- | ------- | -------- | ------------- | ---- |
| Telemetry Name          | ✅       | ✅        | ✅             | ✅ |
| Telemetry Verbosity     | ✅       | ✅        | ✅             | ✅ |
| Custom Telemetry Config | ✅       | ✅        | ✅             | ✅ |

### Simulation

| Feature              | TalonFX | TalonFXS | SparkMax/Flex | Nova |
| -------------------- | ------- | -------- | ------------- | ---- |
| DCMotorSim           | ✅       | ✅        | ✅             | ✅ |
| Moment of Inertia    | ✅       | ✅        | ✅             | ✅ |
| Custom Sim Supplier  | ✅       | ✅        | ✅             | ✅ |
| External Encoder Sim | ✅       | ✅        | ✅             | ✅ |

### Safety Features

| Feature            | TalonFX | TalonFXS | SparkMax/Flex                              | Nova |
| ------------------ | ------- | -------- | ------------------------------------------ | ---- |
| Temperature Cutoff | ❌       | ❌        | ⚠️ (requires an Exponential Profile or LQR controller) | ⚠️ (requires closed loop mode) |

{% hint style="warning" %}
**Temperature Cutoff on Sparks**: the cutoff is checked by the closed loop running on the SystemCore, so on a `SparkWrapper` it only works with an Exponential Profile or an LQR controller. With a trapezoidal profile, which runs on the SPARK, it does not work.
{% endhint %}

{% hint style="info" %}
YAMS doesn't provide a built-in SysId routine. Use `SmartMechanism.setVoltage(Voltage)` (available on every mechanism) as the voltage-consumer function for WPILib's own `SysIdRoutine`.
{% endhint %}

***

## Motor type support by wrapper

### TalonFXWrapper

* Kraken X60 / Kraken X60 FOC
* Falcon 500 / Falcon 500 FOC

### TalonFXSWrapper

* NEO (via JST connector)
* NEO 550 (via JST connector)
* NEO Vortex (via JST connector)
* Minion (via JST connector, with Advanced Hall Support)

### SparkWrapper

* NEO (SparkMax with brushless mode)
* NEO 550 (SparkMax with brushless mode)
* NEO Vortex (SparkFlex)
* Any brushless motor (SparkMax/SparkFlex)

### NovaWrapper

Set with the `MotorType` given to the `Nova` constructor (`com.thrifty.core.Motor.MotorType`):

* NEO (`MotorType.NEO`)
* Minion (`MotorType.MINION`)
* Pulsar (`MotorType.PULSAR`)
* Brushed DC motors (`MotorType.BRUSHED`)

***

## Configuration examples

### TalonFX with CANcoder

```java
TalonFX motor = new TalonFX(1, new CANBus(CANPort.CAN_S0));
CANcoder encoder = new CANcoder(2, new CANBus(CANPort.CAN_S0));

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(1, 0, 0)
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(5)))
    .withExternalEncoder(encoder)
    .withExternalEncoderZeroOffset(Degrees.of(90))
    .withUseExternalFeedbackEncoder(true)
    .withZeroPower(MotorMode.BRAKE)
    .withStatorCurrentLimit(Amps.of(40))
    .withTelemetry("ArmMotor", TelemetryVerbosity.HIGH);

SmartMotorController smc = new TalonFXWrapper(motor, DCMotor.getKrakenX60(1), config);
```

### TalonFXS with NEO

```java
TalonFXS motor = new TalonFXS(1, new CANBus(CANPort.CAN_S0));

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(0.5, 0, 0)
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(4)))
    .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
    .withZeroPower(MotorMode.COAST)
    .withStatorCurrentLimit(Amps.of(40))
    .withTelemetry("FlywheelMotor", TelemetryVerbosity.HIGH);

SmartMotorController smc = new TalonFXSWrapper(motor, DCMotor.getNEO(1), config);
```

### SparkMax with absolute encoder

```java
SparkMax spark = new SparkMax(CANPorts.fromBusId(1), 1, MotorType.kBrushless);

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(1, 0, 0)
    .withTrapezoidalProfile(DegreesPerSecond.of(180), DegreesPerSecondPerSecond.of(360))
    .withExternalEncoder(spark.getAbsoluteEncoder())
    .withExternalEncoderInverted(true)
    .withUseExternalFeedbackEncoder(true)
    .withExternalEncoderZeroOffset(Degrees.of(45))
    .withFeedforward(new ArmFeedforward(0, 0.5, 0, 0))
    .withZeroPower(MotorMode.BRAKE)
    .withTelemetry("WristMotor", TelemetryVerbosity.HIGH);

SmartMotorController smc = new SparkWrapper(spark, DCMotor.getNEO(1), config);
```

### Thrifty Nova with a CAN Encoder

```java
Nova nova = new Nova(0, 3, MotorType.NEO);   // CAN bus 0, CAN ID 3
CanEncoder encoder = new CanEncoder(0, 4);   // CAN bus 0, CAN ID 4

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(0.2, 0, 0)
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(5)))
    .withExternalEncoder(encoder)
    .withExternalEncoderZeroOffset(Degrees.of(90))
    .withUseExternalFeedbackEncoder(true)
    .withZeroPower(MotorMode.BRAKE)
    .withStatorCurrentLimit(Amps.of(40))
    .withTelemetry("ArmMotor", TelemetryVerbosity.HIGH);

SmartMotorController smc = new NovaWrapper(nova, DCMotor.getNEO(1), config);
```

***

## Known limitations

### SparkMax/SparkFlex (SparkWrapper)

* Exponential profiles run on the SystemCore, not on the motor controller
* Supply current limits are not supported
* External encoders must be REV-compatible (Through Bore, Hex)

### TalonFX/TalonFXS

* Requires Phoenix 6 library
* CANcoder/CANdi must be on same CAN bus for best performance

### Thrifty Nova (NovaWrapper)

* Requires ThriftyLib 2027
* All closed loop control, motion profiles and feedforwards run on the SystemCore
* One ramp rate for open and closed loop control
* The internal encoder cannot be inverted (`withEncoderInverted` throws)
* Vendor control requests (`withVendorControlRequest`) are not supported
* Hardware followers must be `Nova` controllers
* Setting the encoder velocity is only supported in simulation
