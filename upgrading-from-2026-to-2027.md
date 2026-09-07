---
icon: arrow-up-right-dots
---

# Upgrading from 2026 to 2027

Between the 2026 and 2027 releases, `ArmConfig`, `ElevatorConfig`, `PivotConfig`, and `FlyWheelConfig` were trimmed down: anything that duplicated a `SmartMotorControllerConfig` setting was removed from the mechanism config and now lives on the motor config instead. This mostly affects how you wire a `SmartMotorController` into a mechanism, plus two setters that moved.

{% hint style="warning" %}
These are breaking changes. Code written against 2026 YAMS will fail to compile until it's updated using the patterns below.
{% endhint %}

## Mechanism constructors now take the `SmartMotorController` directly

Every positional/velocity mechanism (`Arm`, `Elevator`, `Pivot`, `FlyWheel`, …) used to take a single-argument constructor, with the `SmartMotorController` attached to the config via `.withSmartMotorController(...)`. That setter is gone — pass the `SmartMotorController` as a second constructor argument instead.

```java
// Before (2026)
SmartMotorController smc = new TalonFXWrapper(new TalonFX(1), DCMotor.getKrakenX60(1), motorConfig);
FlyWheelConfig config = new FlyWheelConfig()
    .withDiameter(Inches.of(4))
    .withSmartMotorController(smc);
FlyWheel flywheel = new FlyWheel(config);
```

```java
// After (2027)
SmartMotorController smc = new TalonFXWrapper(new TalonFX(1), DCMotor.getKrakenX60(1), motorConfig);
FlyWheelConfig config = new FlyWheelConfig()
    .withDiameter(Inches.of(4));
FlyWheel flywheel = new FlyWheel(config, smc);
```

This applies identically to `Arm`, `Elevator`, and `Pivot` — swap `new Mechanism(config.withSmartMotorController(smc))` for `new Mechanism(config, smc)` everywhere.

## Moment of inertia moved to `SmartMotorControllerConfig`

`withMass(Mass)` was removed from `ArmConfig`, `ElevatorConfig`, and `FlyWheelConfig`. Configure the simulated moment of inertia on the `SmartMotorControllerConfig` instead, either from a length + mass estimate (the same `SingleJointedArmSim.estimateMOI(...)` calculation used previously) or a known MOI value straight from CAD.

```java
// Before (2026)
FlyWheelConfig config = new FlyWheelConfig()
    .withDiameter(Inches.of(4))
    .withMass(Pounds.of(2));
```

```java
// After (2027)
SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig()
    // ...
    .withMomentOfInertia(Inches.of(4), Pounds.of(2)); // length, weight — same estimateMOI() as before

FlyWheelConfig config = new FlyWheelConfig()
    .withDiameter(Inches.of(4));
```

For an `Arm`, use the arm's length in place of diameter: `.withMomentOfInertia(Inches.of(30), Pounds.of(8))`. If you already know the MOI (e.g. from CAD), skip the estimate entirely with `.withMomentOfInertia(MomentOfInertia)`.

## Starting position moved to `SmartMotorControllerConfig`

`withStartingPosition(Angle)` (and the `Distance` overload on `ElevatorConfig`) was removed from the mechanism configs. Set it on `SmartMotorControllerConfig` instead:

```java
// Before (2026)
ArmConfig config = new ArmConfig()
    .withHardLimits(Degrees.of(-180), Degrees.of(180))
    .withStartingPosition(Degrees.of(-180));
```

```java
// After (2027)
SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig()
    // ...
    .withStartingPosition(Degrees.of(-180));

ArmConfig config = new ArmConfig()
    .withHardLimits(Degrees.of(-180), Degrees.of(180));
```

## Quick checklist

* [ ] Replace `config.withSmartMotorController(smc)` + `new Mechanism(config)` with `new Mechanism(config, smc)`.
* [ ] Move every `.withMass(Mass)` call from a mechanism config to `.withMomentOfInertia(Distance, Mass)` (or `.withMomentOfInertia(MomentOfInertia)`) on that mechanism's `SmartMotorControllerConfig`.
* [ ] Move every `.withStartingPosition(...)` call from a mechanism config to the corresponding `SmartMotorControllerConfig`.
* [ ] Double check any hard-limit setter names — `withHardLimits(...)` (plural) is correct on `ArmConfig`/`ElevatorConfig`/`PivotConfig`; there has never been a singular `withHardLimit(...)`.
