---
icon: chart-line
---

# Exponential Profiles

## What is a Motion Profile?

A motion profile is a trajectory that defines how a mechanism should move from one position to another over time. Instead of commanding a mechanism to instantly jump to a target position (which would require infinite acceleration), motion profiles generate a smooth path that respects the physical limitations of your system.

Motion profiles output a series of intermediate setpoints (position and velocity) that your closed-loop controller tracks. This results in:

* **Predictable motion** - You know exactly how long a movement will take
* **Smooth transitions** - No jerky movements that stress mechanical components
* **Controlled acceleration** - Prevents excessive current draw and brownouts

## Trapezoidal vs Exponential Profiles

<figure><img src="../../.gitbook/assets/exponential-profile-compare.png" alt="Two graphs. Setpoint velocity over time: a trapezoidal profile ramps in a straight line to a flat top, an exponential profile rises quickly and curves over near top speed. Acceleration against speed: the motor's available acceleration falls to zero at top speed, the exponential profile follows it, and the trapezoidal profile asks for a constant acceleration the motor cannot give at high speed"><figcaption><p>The two profiles, and why the motor can follow one more easily than the other.</p></figcaption></figure>

### Trapezoidal Profiles

Trapezoidal profiles are the most common type. The name comes from the trapezoidal shape of the velocity-time graph.

**Key characteristics:**

* Constant acceleration until max velocity is reached
* Cruise at max velocity (if distance is long enough)
* Constant deceleration to stop at target
* Assumes unlimited torque during acceleration phases

**When trapezoidal profiles struggle:**

* When a disturbance knocks the mechanism off the profile
* When the motor cannot actually provide the commanded acceleration
* When current limiting reduces available torque

### Exponential Profiles

Exponential profiles model how DC motors actually behave. Instead of assuming constant acceleration, they account for the motor's torque-speed curve and the system's physical dynamics.

**Key characteristics:**

* Acceleration naturally decreases as velocity increases (following motor physics)
* Uses motor kV (velocity constant) and kA (acceleration constant) to model behavior
* More accurately represents what the motor can actually achieve
* Better disturbance rejection

{% hint style="info" %}
Exponential profiles are named for the exponential approach to maximum velocity, which mirrors the natural response of a first-order system like a DC motor.
{% endhint %}

<figure><img src="../../.gitbook/assets/exponential-profile-tracking.gif" alt="Animation of a 12 volt motor following each profile. With the trapezoidal profile the measured velocity falls behind the setpoint as the speed rises and the motor saturates at 12 volts. With the exponential profile the measured velocity stays on the setpoint"><figcaption><p>A simulated 12 V motor following each profile. The trapezoidal profile asks for more than the motor has left at speed.</p></figcaption></figure>

## The Physics Behind Exponential Profiles

DC motors produce torque proportional to current. However, as the motor spins faster, back-EMF reduces the effective voltage across the windings, which reduces current and therefore torque.

The relationship is:

$$V = kS + kV \cdot \omega + kA \cdot \alpha$$

Where:

* **V** = Applied voltage
* **kS** = Static friction voltage (voltage to overcome stiction)
* **kV** = Velocity constant (volts per unit velocity)
* **kA** = Acceleration constant (volts per unit acceleration)
* **ω** = Angular velocity
* **α** = Angular acceleration

Rearranging for acceleration:

$$\alpha = \frac{V - kS - kV \cdot \omega}{kA}$$

This shows that as velocity (ω) increases, the available acceleration (α) decreases. This is the fundamental insight that exponential profiles capture.

## When to Use Each Profile Type

| Situation                               | Recommended Profile      | Why                                                                                                                          |
| --------------------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| Motor is always current-limited         | Trapezoidal              | Current limiting makes torque roughly constant regardless of speed                                                           |
| Motor is never current-limited          | Exponential              | Motor follows natural torque-speed curve                                                                                     |
| Sometimes current-limited               | Hybrid (not yet in YAMS) | See [this whitepaper](https://www.chiefdelphi.com/t/whitepaper-trapezoidal-exponential-motion-profiling/443468/12?u=nstrike) |
| Heavy mechanism with adequate motor     | Exponential              | Gravity/friction dominate, exponential handles disturbances better                                                           |
| Lightweight mechanism                   | Either                   | Both work well when inertia is low                                                                                           |
| Mechanism with significant disturbances | Exponential              | Better recovery when knocked off profile                                                                                     |
| Precise timing required                 | Trapezoidal              | Deterministic duration calculation                                                                                           |

{% hint style="success" %}
**Rule of thumb**: If your mechanism works well with trapezoidal profiles, keep using them. Switch to exponential profiles when you encounter disturbance rejection issues or when you want the profile to better match your motor's capabilities.
{% endhint %}

## Exponential Profiles in YAMS

YAMS provides `ExponentialProfilePIDController` (`yams.core.math.ExponentialProfilePIDController`), which combines an exponential motion profile with a PID controller and has factory methods for the profile's constraints. In a `SmartMotorControllerConfig` you set the PID gains with `withClosedLoopController(kP, kI, kD)` and the profile with `withExponentialProfile(constraints)`, instead of a trapezoidal profile (`withTrapezoidalProfile`) or no profile at all.

### Creating Constraints

YAMS provides factory methods to create exponential profile constraints for common mechanism types:

#### For Elevators

```java
ExponentialProfile.Constraints constraints = ExponentialProfilePIDController.createElevatorConstraints(
    Volts.of(12),        // Maximum voltage during profile
    DCMotor.getNEO(2),   // Motor type and quantity
    Pounds.of(15),       // Carriage mass
    Inches.of(1),        // Drum radius (or sprocket pitch radius)
    gearing              // MechanismGearing object
);
```

#### For Arms

```java
ExponentialProfile.Constraints constraints = ExponentialProfilePIDController.createArmConstraints(
    Volts.of(12),        // Maximum voltage during profile
    DCMotor.getKrakenX60(1), // Motor type and quantity
    Pounds.of(8),        // Arm mass (at center of mass)
    Inches.of(18),       // Arm length (to center of mass)
    gearing              // MechanismGearing object
);
```

#### For Flywheels

```java
ExponentialProfile.Constraints constraints = ExponentialProfilePIDController.createFlywheelConstraints(
    Volts.of(12),        // Maximum voltage during profile
    DCMotor.getFalcon500(1), // Motor type and quantity
    Pounds.of(2),        // Flywheel mass
    Inches.of(2),        // Flywheel radius
    gearing              // MechanismGearing object
);
```

#### Custom Constraints

If the factory methods don't fit your use case, you can create constraints directly:

```java
ExponentialProfile.Constraints constraints = ExponentialProfilePIDController.createConstraints(
    Volts.of(12),                    // Maximum voltage
    RotationsPerSecond.of(100),      // Maximum velocity (kV derived)
    RotationsPerSecondPerSecond.of(200) // Maximum acceleration (kA derived)
);
```

### Using with SmartMotorControllerConfig

```java
SmartMotorControllerConfig config = new SmartMotorControllerConfig(this)
    .withControlMode(ControlMode.CLOSED_LOOP)
    .withClosedLoopController(30, 0, 0) // kP, kI, kD
    .withExponentialProfile(ExponentialProfilePIDController.createElevatorConstraints(
        Volts.of(12),
        DCMotor.getNEO(2),
        Pounds.of(15),
        Inches.of(1),
        gearing
    ))
    .withGearing(gearing)
    .withFeedforward(new ElevatorFeedforward(0, 0.5, 0, 0))
    .withTelemetry("Elevator", TelemetryVerbosity.HIGH);
```

## Tuning Exponential Profiles

### Step 1: Find kV and kA

The exponential profile needs accurate kV and kA values. You have two options:

| Option | How | Good for |
| --- | --- | --- |
| Factory methods | YAMS calculates kV and kA from the motor, gearing and mass | A first version, and simulation |
| Measure them yourself | One constant voltage test and a velocity graph, see [below](#measuring-kv-and-ka) | Real mechanisms, with no extra tools |

{% hint style="warning" %}
Factory methods use theoretical values. Real mechanisms have friction, cable drag, and other losses, so measured values are more accurate.
{% endhint %}

### Measuring kV and kA

It needs only a button binding and a graph, and it gives you both profile constants from one test.

<figure><img src="../../.gitbook/assets/exponential-profile-step-test.png" alt="A graph of velocity after a constant 6 volts is applied. It levels off at 2.9 rotations per second, so kV is 6.0 minus kS, divided by 2.9, which is 2.0. It reaches 63 percent of that speed after 0.15 seconds, so kA is kV times 0.15, which is 0.30"><figcaption><p>One voltage step gives both constants. Example values.</p></figcaption></figure>

{% stepper %}
{% step %}
#### Find kS

Raise the voltage slowly from zero until the mechanism just starts to move. That voltage is kS.
{% endstep %}

{% step %}
#### Apply a constant voltage

Bind a button to a fixed voltage, well below 12 V so the mechanism does not hit a hard stop before it reaches a steady speed:

```java
driver.a().whileTrue(flywheel.setVoltage(Volts.of(6)));
```

Graph `NT:/Mechanisms/<Mechanism>/<Motor>/mechanism/velocity` in AdvantageScope while you hold the button. Leave it in rotations per second: the profile's kV and kA are per rotation of the mechanism.
{% endstep %}

{% step %}
#### Read the steady speed: that gives kV

Once the line goes flat, read the speed.

`kV = (voltage - kS) / steady speed`
{% endstep %}

{% step %}
#### Read the time to 63%: that gives kA

Find the moment the velocity reached 63% of the steady speed, and how long after the voltage was applied that was.

`kA = kV x that time`
{% endstep %}

{% step %}
#### Put them in the config

```java
.withExponentialProfile(ExponentialProfile.Constraints.fromCharacteristics(
    10.0, // maximum voltage the profile may ask for
    2.0,  // kV, volts per rotation per second
    0.30  // kA, volts per rotation per second squared
))
```
{% endstep %}

{% step %}
#### Refine them live

At `TelemetryVerbosity.HIGH` the profile's constants are tunable under `NT:/Tuning/.../closedloop/motionprofile/` as `kV`, `kA` and `maxInput`. Run a move and compare `mechanism/velocity` with `closedloop/setpoint/velocity`:

* Falls behind while speeding up: raise `kA`.
* Falls behind at top speed, or never reaches it: raise `kV`.
* Follows easily with voltage to spare: lower them a little, or raise `maxInput`.

See [Live tuning](../live-tuning/README.md) for how to edit these.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
**Gravity gets in the way of this test.** On an elevator, subtract kG as well: `kV = (voltage - kS - kG) / steady speed`. On an arm the pull of gravity changes with the angle, so the speed never settles: start from `createArmConstraints(...)` and use the live refinement step instead.
{% endhint %}

### Step 2: Set Maximum Voltage

The maximum voltage determines how aggressively the profile will command the mechanism. Consider:

* **12V** - Full battery voltage, most aggressive
* **10V** - Leaves headroom for PID corrections
* **8V** - Conservative, good for testing

### Step 3: Tune PID Gains

With exponential profiles, the feedforward does most of the work. The PID handles:

* **kP** - Corrects position error. Start low (1-5) and increase until responsive
* **kI** - Usually not needed with good feedforward. Use only if steady-state error persists
* **kD** - Dampens oscillations. Add if mechanism oscillates around target

### Step 4: Add Feedforward

Even with exponential profiles, you still need feedforward for:

* **Gravity compensation** (arms, elevators) - The profile doesn't know about gravity
* **Friction compensation** (kS) - Helps overcome stiction

```java
// For an arm
.withFeedforward(new ArmFeedforward(
    0.1,  // kS - voltage to overcome friction
    0.3,  // kG - voltage to hold horizontal
    0,    // kV - usually 0 since profile handles this
    0     // kA - usually 0 since profile handles this
))

// For an elevator
.withFeedforward(new ElevatorFeedforward(
    0.1,  // kS - voltage to overcome friction
    0.2,  // kG - voltage to counteract gravity
    0,    // kV - usually 0 since profile handles this
    0     // kA - usually 0 since profile handles this
))
```

{% hint style="info" %}
When using exponential profiles, the profile itself handles the velocity and acceleration feedforward. You typically only need kS (static friction) and kG (gravity) in your feedforward object.
{% endhint %}

## Motor Controller Support

| Motor Controller         | Trapezoidal Profile      | Exponential Profile           | Notes                       |
| ------------------------ | ------------------------ | ----------------------------- | --------------------------- |
| TalonFX (Kraken, Falcon) | On-device (Motion Magic) | On-device (Motion Magic Expo) | Best performance            |
| TalonFXS                 | On-device (Motion Magic) | On-device (Motion Magic Expo) | Best performance            |
| SparkMax/SparkFlex       | On-device (MAXMotion)    | SystemCore                    | Exponential runs on the SystemCore |
| Thrifty Nova             | SystemCore               | SystemCore                    | All closed loop control runs on the SystemCore |

{% hint style="info" %}
When exponential profiles run on the SystemCore, there's slightly more latency compared to on-device execution. However, the profile calculation is still very fast and suitable for most mechanisms.
{% endhint %}

## Common Issues and Solutions

### Profile is too slow

* Increase maximum voltage
* Check that your mechanism parameters (mass, length, gearing) are accurate
* Verify motor type matches actual hardware

### Profile overshoots target

* Decrease kP
* Add kD for damping
* Check that kV and kA values aren't underestimated

### Mechanism can't keep up with profile

* Your motor may be undersized for the mechanism
* Reduce maximum voltage
* Check for mechanical binding or excessive friction

### Profile works in sim but not on real robot

* Measure kV and kA on the real mechanism, see [Measuring kV and kA](#measuring-kv-and-ka)
* Account for friction with kS in feedforward
* Check current limits aren't restricting torque

## Example: Complete Elevator with Exponential Profile

```java
public class ElevatorSubsystem extends SubsystemBase {

    private final Distance chainPitch = Inches.of(0.25);
    private final int toothCount = 22;
    private final Mass carriageMass = Pounds.of(15);
    private final DCMotor motors = DCMotor.getNEO(2);
    private final MechanismGearing gearing = new MechanismGearing(GearBox.fromReductionStages(5, 4));

    private final SmartMotorControllerConfig motorConfig = new SmartMotorControllerConfig(this)
        .withControlMode(ControlMode.CLOSED_LOOP)
        .withDrumRadius(chainPitch, toothCount)
        .withClosedLoopController(15, 0, 0)
        .withExponentialProfile(ExponentialProfilePIDController.createElevatorConstraints(
            Volts.of(10), // Conservative voltage for testing
            motors,
            carriageMass,
            chainPitch.times(toothCount).div(2 * Math.PI),
            gearing
        ))
        .withSimClosedLoopController(15, 0, 0)
        .withFeedforward(new ElevatorFeedforward(0.1, 0.25, 0, 0))
        .withSimFeedforward(new ElevatorFeedforward(0, 0.2, 0, 0))
        .withGearing(gearing)
        .withZeroPower(MotorMode.BRAKE)
        .withStatorCurrentLimit(Amps.of(40))
        .withSoftLimits(Meters.of(0), Meters.of(1.2))
        .withStartingPosition(Meters.of(0))
        .withTelemetry("ElevatorMotor", TelemetryVerbosity.HIGH);

    private final SparkMax leftSpark = new SparkMax(CANPorts.fromBusId(1), 1, MotorType.kBrushless);
    private final SparkMax rightSpark = new SparkMax(CANPorts.fromBusId(1), 2, MotorType.kBrushless);
    
    private final SmartMotorController motor = new SparkWrapper(
        leftSpark, 
        motors, 
        motorConfig.withFollowers(Pair.of(rightSpark, true))
    );

    private final Elevator elevator = new Elevator(
        new ElevatorConfig()
            .withHardLimits(Meters.of(-0.05), Meters.of(1.3))
            .withCarriageWeight(carriageMass)
            .withTelemetry("Elevator", TelemetryVerbosity.HIGH),
        motor
    );

    public Command goToHeight(Distance height) {
        return elevator.setHeight(height);
    }

    @Override
    public void periodic() {
        elevator.updateTelemetry();
    }

    @Override
    public void simulationPeriodic() {
        elevator.simIterate();
    }
}
```

## Further Reading

* [CTRE Motion Magic Expo Documentation](https://v6.docs.ctr-electronics.com/en/latest/docs/api-reference/device-specific/talonfx/motion-magic.html#motion-magic-expo) - CTRE's explanation of exponential profiles
* [How do I use Exponential Profiles?](../../how-to/how-do-i-use-exponential-profiles.md) - Quick how-to guide
* [Trapezoidal/Exponential Motion Profiling Whitepaper](https://www.chiefdelphi.com/t/whitepaper-trapezoidal-exponential-motion-profiling/443468/12?u=nstrike) - Advanced hybrid profiling techniques
* [WPILib Tuning Guide](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/tuning-turret.html) - General PID and feedforward tuning
