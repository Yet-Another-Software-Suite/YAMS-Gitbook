# How do I enable battery discharge simulation?

By default, `BatterySim` only models instantaneous voltage sag from combined current draw, and the simulated battery itself never changes, so it sags from the same fully-charged 12V/20mΩ baseline whether it's the first cycle of the match or the last. Enabling discharge simulation layers state-of-charge modeling on top: as your simulated robot draws amp-hours, the battery's open-circuit voltage droops and its internal resistance rises, so a long autonomous-plus-teleop simulation run actually sags harder near the end, the way a real battery does late in a match. This page is the practical walkthrough; for the full explanation of why YAMS models one shared battery at all, see [Battery Simulation](../understanding/battery-simulation.md).

## Turn on discharge modeling

Call `BatterySim.enableDischarge(...)` once, early on, with the capacity, nominal voltage, and nominal resistance of the battery you're modeling. `robotInit()` is a good place for it. A fresh 18 Ah competition battery is a reasonable default if you don't have measured numbers for your own.

```java
import static edu.wpi.first.units.Units.Volts;
import static edu.wpi.first.units.Units.Milliohms;
import yams.motorcontrollers.simulation.BatterySim;

@Override
public void robotInit() {
  BatterySim.enableDischarge(18.0, Volts.of(12.9), Milliohms.of(20));
}
```

{% hint style="warning" %}
Discharge modeling only affects simulation. On a real robot `BatterySim` is never consulted, the RIO reports the actual battery voltage from hardware, so this call is a no-op there.
{% endhint %}

## Make sure your mechanisms report accurate current draw

Discharge modeling can only drain the simulated battery as realistically as the current draw it's fed, and that comes straight out of each mechanism's physics simulation. If a mechanism's moment of inertia is a default/guessed value rather than a real one, its simulated current draw, and therefore how fast it drains the modeled battery, will be too low. Set a real MOI on each `SmartMotorControllerConfig` with `.withMomentOfInertia(...)` before relying on discharge numbers; see [MOI](../details/turrets-wrists.md#moi) for how.

## Reset between runs (optional)

`BatterySim` carries discharge state across simulation runs the same way a real battery carries charge across matches. If you want every simulated match or unit test to start from a full charge instead of inheriting drain from the previous run, call `BatterySim.resetDischarge()` at the start of it, in `testInit()` for unit tests, or wherever you reset robot state between simulated matches.

```java
@Override
public void testInit() {
  BatterySim.resetDischarge();
}
```

Call `BatterySim.disableDischarge()` instead if you want to turn discharge modeling back off entirely and revert to a constant nominal voltage/resistance.

## Read the state of charge for telemetry (optional)

`BatterySim.getStateOfCharge()` returns the modeled charge from `0` (empty) to `1` (full), so you can publish it to a dashboard or log it alongside your other telemetry to watch the battery drain over a simulated match.

```java
SmartDashboard.putNumber("Battery/StateOfCharge", BatterySim.getStateOfCharge());
```

## Replace the state-of-charge interpolation curve (optional)

`enableDischarge(...)` sags voltage along a curve tuned for a typical FRC sealed lead-acid battery, flat through most of the charge, then dropping off quickly near depletion. If that doesn't match the specific battery you're trying to model, call `BatterySim.replaceSOCInterpolation(...)` with your own curve **before** calling `enableDischarge(...)`.

Reach for this when you're modeling a well-used competition battery that sags earlier than a fresh one, or when you have real measured voltage-vs-state-of-charge data from a load tester.

{% tabs %}
{% tab title="Java" %}
```java
import edu.wpi.first.math.interpolation.InterpolatingDoubleTreeMap;
import yams.motorcontrollers.simulation.BatterySim;

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
// Call enableDischarge(...) AFTER replaceSOCInterpolation(...), or it uses the default curve.
BatterySim.enableDischarge(15.0, Volts.of(12.6), Milliohms.of(28));
```
{% endtab %}

{% tab title="C++" %}
```cpp
#include <map>
#include <yams/motorcontrollers/simulation/BatterySim.hpp>

// Turn on discharge modeling.
yams::motorcontrollers::simulation::BatterySim::EnableDischarge(
    18.0, units::volt_t{12.9}, units::ohm_t{0.020});

// Reset between runs (optional), e.g. in TestInit().
yams::motorcontrollers::simulation::BatterySim::ResetDischarge();

// Read the state of charge for telemetry (optional).
double soc = yams::motorcontrollers::simulation::BatterySim::GetStateOfCharge();

// Replace the state-of-charge interpolation curve (optional), before EnableDischarge(...).
std::map<double, double> wornBatteryCurve{
    {0.00, 8.0},  {0.05, 9.5},  {0.10, 10.5}, {0.20, 11.2}, {0.40, 11.6},
    {0.60, 11.9}, {0.80, 12.2}, {0.90, 12.4}, {1.00, 12.6},
};
yams::motorcontrollers::simulation::BatterySim::ReplaceSOCInterpolation(wornBatteryCurve);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Keys and values should span the full `[0, 1]` state-of-charge range. Querying outside the range you defined just returns the nearest endpoint's voltage instead of extrapolating, so a table missing the low or high end won't sag realistically there. See [Battery Simulation → Custom discharge curves](../understanding/battery-simulation.md#custom-discharge-curves) for more on choosing curve points.
{% endhint %}

## Related pages

* [Battery Simulation](../understanding/battery-simulation.md), the full explanation of the shared-battery model and why it matters
* [MOI](../details/turrets-wrists.md#moi), accurate current draw starts with an accurate moment of inertia
* [Limiting Power Consumption](../details/editor/limiting-power-consumption.md)
