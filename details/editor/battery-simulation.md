---
icon: car-battery
---

# Battery Simulation

YAMS models a single, shared robot battery in simulation instead of letting each mechanism compute its own voltage sag in isolation. Every simulated `SmartMotorController` — whether driven by an `ArmSimSupplier`, `ElevatorSimSupplier`, `DCMotorSimSupplier`, or a hardware wrapper (`SparkWrapper`, `TalonFXWrapper`, `TalonFXSWrapper`) — reports its current draw into [`BatterySim`](https://yet-another-software-suite.github.io/YAMS/javadocs/yams/motorcontrollers/simulation/BatterySim.html), which combines every mechanism's draw into one realistic loaded-voltage calculation and writes it to `RoboRioSim`.

{% hint style="info" %}
This is automatic. You do not need to call anything for basic voltage sag under combined load — it happens as soon as a mechanism has a sim supplier attached and `simIterate()`/`SimIterate()` is called each loop, same as [Simulation without a mechanism](README.md#simulation-without-a-mechanism).
{% endhint %}

## Why this matters

Before, if two mechanisms each computed `RoboRioSim.setVInVoltage(...)` from only their own current draw, whichever mechanism updated last would overwrite the other's contribution — the simulated battery would never actually see the *combined* draw of the whole robot. With a shared `BatterySim`, a flywheel spinning up while an elevator is climbing will sag the simulated bus voltage the same way a real battery would under both loads at once, which is exactly the scenario that trips brownouts on a real robot. See [Limiting Power Consumption](limiting-power-consumption.md) for how to guard against that on top of realistic simulation.

## Realistic discharge over a match

By default the simulated battery holds a constant nominal voltage and resistance (12V / 20 mΩ) — enough to model voltage sag under instantaneous load, but not a battery getting weaker over the course of a match. Call `BatterySim.enableDischarge(...)` to layer state-of-charge modeling on top: as amp-hours are drawn from the battery, its open-circuit voltage droops along a discharge curve and its internal resistance rises, both becoming more pronounced as the battery empties — the same way a real lead-acid FRC battery behaves late in a match.

```java
import yams.motorcontrollers.simulation.BatterySim;

// Somewhere in robotInit(), with a fresh, fully-charged 18Ah battery model:
BatterySim.enableDischarge(18.0, Volts.of(12.9), Milliohms.of(20));
```

| Method                                                       | Description                                                                                       |
| ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `enableDischarge(double capacityAh, Voltage, Resistance)`    | Turns on discharge modeling with the given capacity and nominal (fully-charged) voltage/resistance. |
| `disableDischarge()`                                         | Reverts to a constant nominal voltage/resistance.                                                    |
| `resetDischarge()`                                            | Resets the simulated battery back to a full charge, e.g. between test runs.                          |
| `getStateOfCharge()`                                          | Returns the current state of charge from `0` (empty) to `1` (full), for telemetry or dashboards.     |

{% hint style="warning" %}
Discharge only affects simulation. On a real robot `BatterySim` is never consulted — the RIO reports the actual battery voltage from hardware.
{% endhint %}

{% hint style="info" %}
Call `BatterySim.resetDischarge()` in `testInit()` or at the start of a unit test if you want every simulated match to start from a full charge rather than carrying over drain from a previous run.
{% endhint %}

## Custom discharge curves

`enableDischarge(...)` sags voltage along a curve tuned for a typical FRC sealed lead-acid battery — flat through most of the charge, then dropping off quickly near depletion. Not every battery you want to model behaves that way, so call `BatterySim.replaceSOCInterpolation(...)` **before** `enableDischarge(...)` to swap in your own curve.

Reach for this when:

* **You're simulating a well-used competition battery**, which sags earlier and harder than a fresh one — a flatter, lower curve reproduces "that one tired battery" instead of assuming every match starts fresh.
* **You're modeling a different chemistry**, like LiFePO4, which holds a much flatter voltage curve than lead-acid until it's nearly empty, then falls off a cliff — a shape the default curve doesn't capture.
* **You have measured data**, e.g. from putting a real battery on a load tester — feeding in the actual voltage-vs-state-of-charge points gives the most accurate brownout predictions for that specific battery.

```java
import edu.wpi.first.math.interpolation.InterpolatingDoubleTreeMap;
import yams.motorcontrollers.simulation.BatterySim;

// Model a well-used competition battery that sags earlier and more severely than a new one.
InterpolatingDoubleTreeMap wornBatteryCurve = new InterpolatingDoubleTreeMap();
wornBatteryCurve.put(0.00, 8.0);
wornBatteryCurve.put(0.05, 9.5);
wornBatteryCurve.put(0.10, 10.5);
wornBatteryCurve.put(0.20, 11.2);
wornBatteryCurve.put(0.40, 11.6);
wornBatteryCurve.put(0.60, 11.9);
wornBatteryCurve.put(0.80, 12.2);
wornBatteryCurve.put(0.90, 12.4);
wornBatteryCurve.put(1.00, 12.6);

BatterySim.replaceSOCInterpolation(wornBatteryCurve);
// Pair with a reduced usable capacity and higher resistance to match a worn battery.
BatterySim.enableDischarge(15.0, Volts.of(12.6), Milliohms.of(28));
```

{% hint style="info" %}
Keys and values should span the full `[0, 1]` state-of-charge range. Querying outside the range you defined just returns the nearest endpoint's voltage instead of extrapolating, so a table missing the low or high end will not sag realistically there.
{% endhint %}

### C++

The same model is available as `yams::motorcontrollers::simulation::BatterySim`:

```cpp
#include <yams/motorcontrollers/simulation/BatterySim.hpp>

// Somewhere during robot setup:
yams::motorcontrollers::simulation::BatterySim::EnableDischarge(
    18.0, units::volt_t{12.9}, units::ohm_t{0.020});
```

| Method                                                                              | Description                                                             |
| -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `EnableDischarge(double capacityAh, units::volt_t, units::ohm_t)`                  | Turns on discharge modeling with the given capacity and nominal voltage/resistance. |
| `DisableDischarge()`                                                                | Reverts to a constant nominal voltage/resistance.                            |
| `ResetDischarge()`                                                                  | Resets the simulated battery back to a full charge.                          |
| `GetStateOfCharge()`                                                                | Returns the current state of charge from `0` (empty) to `1` (full).          |
| `ReplaceSOCInterpolation(const std::map<double, double>&)`                         | Swaps in a custom state-of-charge → voltage discharge curve.                |

```cpp
#include <map>
#include <yams/motorcontrollers/simulation/BatterySim.hpp>

// Model a well-used competition battery that sags earlier and more severely than a new one.
std::map<double, double> wornBatteryCurve{
    {0.00, 8.0},  {0.05, 9.5},  {0.10, 10.5}, {0.20, 11.2}, {0.40, 11.6},
    {0.60, 11.9}, {0.80, 12.2}, {0.90, 12.4}, {1.00, 12.6},
};

yams::motorcontrollers::simulation::BatterySim::ReplaceSOCInterpolation(wornBatteryCurve);
// Pair with a reduced usable capacity and higher resistance to match a worn battery.
yams::motorcontrollers::simulation::BatterySim::EnableDischarge(
    15.0, units::volt_t{12.6}, units::ohm_t{0.028});
```

## Related pages

* [Limiting Power Consumption](limiting-power-consumption.md)
* [Simulation without a mechanism](README.md#simulation-without-a-mechanism)
* [Simulation Only PID + FeedForward](simulation-only-pid-+-feedforward.md)
