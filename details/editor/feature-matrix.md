---
icon: table-cells
---

# Motor Controller Feature Matrix

This page provides a comprehensive comparison of supported `SmartMotorControllerConfig` features across all motor controller wrappers in YAMS.

## Supported Motor Controllers

| Wrapper           | Vendor | Hardware                                         | Package Location               |
| ----------------- | ------ | ------------------------------------------------ | ------------------------------ |
| `TalonFXWrapper`  | CTRE   | TalonFX (Kraken X60, Falcon 500)                 | `yams.motorcontrollers.remote` |
| `TalonFXSWrapper` | CTRE   | TalonFXS (controls NEO, NEO 550, Vortex, Minion) | `yams.motorcontrollers.remote` |
| `SparkWrapper`    | REV    | SparkMax, SparkFlex                              | `yams.motorcontrollers.local`  |

## Vendor Configuration Classes

Each wrapper accepts a specific vendor configuration class via `SmartMotorControllerConfig.withVendorConfig()`. This allows you to pass through vendor-specific settings that YAMS doesn't directly expose.

| Wrapper                    | Accepted Vendor Config Class | Package                         |
| -------------------------- | ---------------------------- | -------------------------------- |
| `TalonFXWrapper`           | `TalonFXConfiguration`       | `com.ctre.phoenix6.configs`     |
| `TalonFXSWrapper`          | `TalonFXSConfiguration`      | `com.ctre.phoenix6.configs`     |
| `SparkWrapper` (SparkMax)  | `SparkMaxConfig`             | `com.revrobotics.spark.config`  |
| `SparkWrapper` (SparkFlex) | `SparkFlexConfig`            | `com.revrobotics.spark.config`  |

{% hint style="warning" %}
**SparkWrapper Note**: The vendor config class must match the actual hardware. Use `SparkMaxConfig` for `SparkMax` controllers and `SparkFlexConfig` for `SparkFlex` controllers. Using the wrong config type will throw a `SmartMotorControllerConfigurationException`.
{% endhint %}

### Vendor Config Examples

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

SparkMax sparkMax = new SparkMax(1, MotorType.kBrushless);
SmartMotorController smc = new SparkWrapper(sparkMax, DCMotor.getNEO(1), config);
```

#### SparkFlex with SparkFlexConfig

```java
import com.revrobotics.spark.config.SparkFlexConfig;
import com.revrobotics.spark.SparkFlex;
import com.revrobotics.spark.SparkLowLevel.MotorType;

SparkFlexConfig vendorConfig = new SparkFlexConfig();
// Configure vendor-specific settings not exposed by YAMS

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withVendorConfig(vendorConfig)  // Must be SparkFlexConfig for SparkFlex
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(1, 0, 0)
    // ... other YAMS config
    ;

SparkFlex sparkFlex = new SparkFlex(1, MotorType.kBrushless);
SmartMotorController smc = new SparkWrapper(sparkFlex, DCMotor.getNeoVortex(1), config);
```

{% hint style="info" %}
**Configuration Priority**: Settings in `SmartMotorControllerConfig` always take precedence over vendor config settings. The vendor config is applied first as a base, then YAMS overwrites any settings it manages directly.
{% endhint %}

## Feature Support Matrix

### Legend

| Symbol | Meaning                         |
| ------ | ------------------------------- |
| ✅      | Fully Supported                 |
| ⚠️     | Partially Supported (see notes) |
| ❌      | Not Supported                   |

***

### Basic Configuration

| Feature                   | TalonFX | TalonFXS | SparkMax/Flex |
| ------------------------- | ------- | -------- | ------------- |
| Motor Inversion           | ✅       | ✅        | ✅             |
| Encoder Inversion         | ✅       | ✅        | ✅             |
| Idle Mode (Brake/Coast)   | ✅       | ✅        | ✅             |
| Reset Previous Config     | ✅       | ✅        | ✅             |
| Vendor Config Passthrough | ✅       | ✅        | ✅             |
| Starting Position         | ✅       | ✅        | ✅             |

### Current Limits

| Feature              | TalonFX | TalonFXS | SparkMax/Flex |
| --------------------- | ------- | -------- | -------------- |
| Stator Current Limit | ✅       | ✅        | ✅              |
| Supply Current Limit | ✅       | ✅        | ❌              |

### Voltage Configuration

| Feature                 | TalonFX | TalonFXS | SparkMax/Flex |
| ------------------------ | ------- | -------- | -------------- |
| Voltage Compensation    | ❌       | ❌        | ✅              |
| Closed Loop Max Voltage | ✅       | ✅        | ✅              |

{% hint style="warning" %}
**Voltage Compensation**: `TalonFXWrapper` and `TalonFXSWrapper` throw an exception if voltage compensation is configured, it's only supported on `SparkWrapper`.
{% endhint %}

### Ramp Rates

| Feature               | TalonFX | TalonFXS | SparkMax/Flex |
| ---------------------- | ------- | -------- | -------------- |
| Open Loop Ramp Rate   | ✅       | ✅        | ✅              |
| Closed Loop Ramp Rate | ✅       | ✅        | ✅              |

### Gearing

| Feature                          | TalonFX | TalonFXS | SparkMax/Flex |
| ---------------------------------- | ------- | -------- | -------------- |
| Mechanism Gearing                | ✅       | ✅        | ✅              |
| External Encoder Gearing         | ✅       | ✅        | ✅              |
| Mechanism Circumference (Linear) | ✅       | ✅        | ✅              |

### Closed Loop Control

| Feature                 | TalonFX      | TalonFXS     | SparkMax/Flex                        |
| ------------------------ | ------------ | ------------ | ------------------------------------- |
| PID Controller          | ✅            | ✅            | ✅                                     |
| LQR Controller          | ⚠️ (RoboRIO) | ⚠️ (RoboRIO) | ⚠️ (RoboRIO)                          |
| Closed Loop Tolerance   | ✅            | ✅            | ✅                                     |
| Control Period Override | ❌            | ❌            | ⚠️ (requires an Exponential Profile) |

{% hint style="warning" %}
**LQR Controller Execution**: LQR is **not natively supported** by any motor controller hardware. When configured, the closed loop control runs on the **RoboRIO** via `SmartMotorController.iterateClosedLoop()`. This means you must call `iterateClosedLoop()` in your periodic method (mechanism classes handle this automatically). See [LQR Controllers](lqr-controllers.md) for details.
{% endhint %}

### Motion Profiles

| Feature             | TalonFX               | TalonFXS              | SparkMax/Flex |
| -------------------- | --------------------- | --------------------- | -------------- |
| Trapezoidal Profile | ✅ (Motion Magic)      | ✅ (Motion Magic)      | ✅ (MAXMotion) |
| Exponential Profile | ✅ (Motion Magic Expo) | ✅ (Motion Magic Expo) | ⚠️ (RoboRIO)   |

{% hint style="info" %}
**Profile Execution Location**:

* **TalonFX/TalonFXS**: Profiles run on the motor controller (Motion Magic)
* **SparkMax/SparkFlex**: Trapezoidal profiles run on motor controller (MAXMotion), exponential profiles run on RoboRIO
{% endhint %}

### Feedforward

| Feature                | TalonFX | TalonFXS | SparkMax/Flex |
| ----------------------- | ------- | -------- | -------------- |
| SimpleMotorFeedforward | ✅       | ✅        | ✅              |
| ArmFeedforward         | ✅       | ✅        | ✅              |
| ElevatorFeedforward    | ✅       | ✅        | ✅              |
| Simulation Feedforward | ✅       | ✅        | ✅              |

### Soft Limits

| Feature              | TalonFX | TalonFXS | SparkMax/Flex |
| --------------------- | ------- | -------- | -------------- |
| Angle Soft Limits    | ✅       | ✅        | ✅              |
| Distance Soft Limits | ✅       | ✅        | ✅              |
| Continuous Wrapping  | ✅       | ✅        | ✅              |

### External Encoders

| Feature                      | TalonFX | TalonFXS | SparkMax/Flex                        |
| ----------------------------- | ------- | -------- | -------------------------------------- |
| External Encoder Support     | ✅       | ✅        | ✅                                      |
| External Encoder Inversion   | ✅       | ✅        | ✅                                      |
| External Encoder Zero Offset | ✅       | ✅        | ✅                                      |
| Use External for Feedback    | ✅       | ✅        | ✅                                      |
| Feedback Synchronization     | ❌       | ❌        | ⚠️ (requires an Exponential Profile or LQR controller) |

**Supported External Encoder Types:**

| Wrapper       | Supported Encoders                             |
| ------------- | ----------------------------------------------- |
| TalonFX       | `CANcoder`, `CANdi` (PWM1/PWM2)                |
| TalonFXS      | `CANcoder`, `CANdi` (PWM1/PWM2)                |
| SparkMax/Flex | `SparkAbsoluteEncoder` (Through Bore, REV Hex) |

### Follower Motors

| Feature                   | TalonFX | TalonFXS | SparkMax/Flex |
| --------------------------- | ------- | -------- | -------------- |
| Hardware Followers        | ✅       | ✅        | ✅              |
| Follower Inversion        | ✅       | ✅        | ✅              |
| Loosely Coupled Followers | ✅       | ✅        | ✅              |

{% hint style="warning" %}
**Follower Compatibility**: Followers must be the same brand as the leader motor controller. You cannot mix vendors for hardware followers.
{% endhint %}

### Telemetry

| Feature                 | TalonFX | TalonFXS | SparkMax/Flex |
| ------------------------- | ------- | -------- | -------------- |
| Telemetry Name          | ✅       | ✅        | ✅              |
| Telemetry Verbosity     | ✅       | ✅        | ✅              |
| Custom Telemetry Config | ✅       | ✅        | ✅              |

### Simulation

| Feature              | TalonFX | TalonFXS | SparkMax/Flex |
| --------------------- | ------- | -------- | -------------- |
| DCMotorSim           | ✅       | ✅        | ✅              |
| Moment of Inertia    | ✅       | ✅        | ✅              |
| Custom Sim Supplier  | ✅       | ✅        | ✅              |
| External Encoder Sim | ✅       | ✅        | ✅              |

### Safety Features

| Feature            | TalonFX | TalonFXS | SparkMax/Flex                        |
| -------------------- | ------- | -------- | -------------------------------------- |
| Temperature Cutoff | ❌       | ❌        | ⚠️ (requires no motion profile configured) |

{% hint style="info" %}
YAMS doesn't provide a built-in SysId routine. Use `SmartMechanism.setVoltage(Voltage)` (available on every mechanism) as the voltage-consumer function for WPILib's own `SysIdRoutine`.
{% endhint %}

***

## Motor Type Support by Wrapper

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

***

## Configuration Examples

### TalonFX with CANcoder

```java
TalonFX motor = new TalonFX(1);
CANcoder encoder = new CANcoder(2);

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(1, 0, 0)
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(5)))
    .withExternalEncoder(encoder)
    .withExternalEncoderZeroOffset(Degrees.of(90))
    .withUseExternalFeedbackEncoder(true)
    .withIdleMode(MotorMode.BRAKE)
    .withStatorCurrentLimit(Amps.of(40))
    .withTelemetry("ArmMotor", TelemetryVerbosity.HIGH);

SmartMotorController smc = new TalonFXWrapper(motor, DCMotor.getKrakenX60(1), config);
```

### TalonFXS with NEO

```java
TalonFXS motor = new TalonFXS(1);

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(0.5, 0, 0)
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(4)))
    .withFeedforward(new SimpleMotorFeedforward(0.1, 0.12, 0))
    .withIdleMode(MotorMode.COAST)
    .withStatorCurrentLimit(Amps.of(40))
    .withTelemetry("FlywheelMotor", TelemetryVerbosity.HIGH);

SmartMotorController smc = new TalonFXSWrapper(motor, DCMotor.getNEO(1), config);
```

### SparkMax with Absolute Encoder

```java
SparkMax spark = new SparkMax(1, MotorType.kBrushless);

SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(1, 0, 0, DegreesPerSecond.of(180), DegreesPerSecondPerSecond.of(360))
    .withExternalEncoder(spark.getAbsoluteEncoder())
    .withExternalEncoderInverted(true)
    .withUseExternalFeedbackEncoder(true)
    .withExternalEncoderZeroOffset(Degrees.of(45))
    .withFeedforward(new ArmFeedforward(0, 0.5, 0, 0))
    .withIdleMode(MotorMode.BRAKE)
    .withTelemetry("WristMotor", TelemetryVerbosity.HIGH);

SmartMotorController smc = new SparkWrapper(spark, DCMotor.getNEO(1), config);
```

***

## Known Limitations

### SparkMax/SparkFlex (SparkWrapper)

* Exponential profiles run on RoboRIO, not on motor controller
* External encoders must be REV-compatible (Through Bore, Hex)

### TalonFX/TalonFXS

* Requires Phoenix 6 library
* CANcoder/CANdi must be on same CAN bus for best performance
