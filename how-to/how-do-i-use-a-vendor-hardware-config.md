# How do I use a vendor hardware config?

Some motor controllers ship with on-board configurations stored in flash, and vendor APIs expose options that YAMS does not surface directly, things like CANivore bus selection, custom status frame rates, or audio feedback settings. YAMS integrates with these through a hardware config layer: you pre-build a vendor config object and hand it to `SmartMotorControllerConfig.withVendorConfig()`, and YAMS merges its own settings on top without discarding the rest. Understanding when to use a vendor hardware config (versus configuring purely through YAMS) helps you avoid conflicting settings between the vendor's stored config and YAMS's runtime config.

YAMS configures your motor controller automatically based on your `SmartMotorControllerConfig`. But sometimes you need to reach features that YAMS doesn't expose directly, custom voltage limits, CANivore bus names, status frame rates, or device-specific options only available through the vendor's own configuration object.

`SmartMotorControllerConfig.withVendorConfig(Object config)` lets you supply a pre-built vendor config as a baseline. YAMS merges its own settings on top of it, so anything you set in `SmartMotorControllerConfig` always wins, but everything else you configured in the vendor object is preserved.

## CTRE TalonFX

Pass a `TalonFXConfiguration` object:

```java
TalonFXConfiguration talonConfig = new TalonFXConfiguration()
    .withAudio(new AudioConfigs().withBeepOnBoot(false).withBeepOnConfig(false))
    .withVoltage(new VoltageConfigs().withPeakReverseVoltage(-10).withPeakForwardVoltage(10));

SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
    .withVendorConfig(talonConfig)
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(4, 0, 0)
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
    .withZeroPower(MotorMode.BRAKE);

TalonFX talonFX = new TalonFX(1, new CANBus(CANPort.CAN_S0));
SmartMotorController smc = new TalonFXWrapper(talonFX, DCMotor.getKrakenX60(1), motorConfig);
```

## CTRE TalonFXS

Pass a `TalonFXSConfiguration` object:

```java
TalonFXSConfiguration talonConfig = new TalonFXSConfiguration()
    .withMotorOutput(new MotorOutputConfigs().withNeutralMode(NeutralModeValue.Brake))
    .withCommutation(new CommutationConfigs().withMotorArrangement(MotorArrangementValue.Minion_JST));

SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
    .withVendorConfig(talonConfig)
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(4, 0, 0)
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)));

TalonFXS talonFXS = new TalonFXS(1, new CANBus(CANPort.CAN_S0));
SmartMotorController smc = new TalonFXSWrapper(talonFXS, DCMotor.getKrakenX60(1), motorConfig);
```

## REV SparkMax

Pass a `SparkMaxConfig` object:

```java
SparkMaxConfig sparkConfig = new SparkMaxConfig();
sparkConfig.signals
    .primaryEncoderPositionPeriodMs(10)
    .primaryEncoderVelocityPeriodMs(20);

SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
    .withVendorConfig(sparkConfig)
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(4, 0, 0)
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
    .withZeroPower(MotorMode.BRAKE);

SparkMax spark = new SparkMax(CANPort.CAN_S0, 1, MotorType.kBrushless);
SmartMotorController smc = new SparkWrapper(spark, DCMotor.getNEO(1), motorConfig);
```

## REV SparkFlex

Pass a `SparkFlexConfig` object:

```java
SparkFlexConfig sparkConfig = new SparkFlexConfig();
sparkConfig.signals
    .primaryEncoderPositionPeriodMs(10)
    .primaryEncoderVelocityPeriodMs(20);

SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
    .withVendorConfig(sparkConfig)
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(4, 0, 0)
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
    .withZeroPower(MotorMode.BRAKE);

SparkFlex spark = new SparkFlex(CANPort.CAN_S0, 1, MotorType.kBrushless);
SmartMotorController smc = new SparkWrapper(spark, DCMotor.getNeoVortex(1), motorConfig);
```

## Thrifty Nova

Pass a ThriftyLib `NovaConfig.Config`, usually a `NovaConfigBatch` holding one or more `NovaConfig` options. The vendor config is applied to the Nova before the YAMS config:

```java
SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
    .withVendorConfig(new NovaConfigBatch(NovaConfig.tempThrottleEnable(true)))
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(4, 0, 0)
    .withGearing(new MechanismGearing(GearBox.fromReductionStages(3, 4)))
    .withZeroPower(MotorMode.BRAKE);

Nova nova = new Nova(0, 1, MotorType.NEO); // CAN bus 0, CAN ID 1
SmartMotorController smc = new NovaWrapper(nova, DCMotor.getNEO(1), motorConfig);
```

Any other vendor config type throws a `SmartMotorControllerConfigurationException`. See [How do I use a Thrifty Nova?](how-do-i-use-a-thrifty-nova.md) for more.

## How precedence works

YAMS uses your vendor config as a starting point and then writes its own settings on top of it. This means:

* Any field you set in `SmartMotorControllerConfig` (PID gains, gearing, zero power mode, current limits, etc.) **will overwrite** the corresponding field in your vendor config.
* Any field you set **only** in the vendor config and that YAMS doesn't touch will be preserved exactly as you set it.

If you set the same option in both places, `SmartMotorControllerConfig` wins. Think of the vendor config as the floor, not the ceiling.

{% hint style="warning" %}
Always verify the final applied configuration on your hardware. Use your vendor's tuning tool (such as Phoenix Tuner X or REV Hardware Client) to confirm that YAMS didn't silently overwrite a setting you intended to keep from your vendor config.
{% endhint %}
