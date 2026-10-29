# Step 1: Define the IO interface

Create an interface that defines what inputs your subsystem reads and what outputs it sends. Following AdvantageKit conventions, inputs are grouped in an `Inputs` class. This needs AdvantageKit installed, from [Step 0: Install AdvantageKit and set up Robot](setup.md).

The inputs are plain numbers with the unit in the name, as AdvantageKit's own examples do. The methods that command the mechanism take WPILib units (`Angle`, `Distance`, `AngularVelocity`), so a caller cannot mix up degrees and rotations.

## Arm IO interface

```java
package first.robot.subsystems.akit;

import org.littletonrobotics.junction.AutoLog;
import org.wpilib.units.measure.Angle;

public interface ArmIO {

  /**
   * Inputs that will be logged and replayed.
   * The @AutoLog annotation generates ArmIOInputsAutoLogged class.
   */
  @AutoLog
  public static class ArmIOInputs {
    public double positionRotations = 0.0;
    public double velocityRotationsPerSec = 0.0;
    public double appliedVolts = 0.0;
    public double supplyCurrentAmps = 0.0;
    public double statorCurrentAmps = 0.0;
    public double temperatureCelsius = 0.0;
    public double targetPositionRotations = 0.0;
  }

  /** Update the inputs from hardware. Called every loop cycle. */
  default void updateInputs(ArmIOInputs inputs) {}

  /** Set the target angle for the arm. */
  default void setTargetAngle(Angle angle) {}

  /** Stop the arm motor. */
  default void stop() {}

  /** Update the sim telemetry */
  default void simIterate() {}

  /** Update the telemetry */
  default void updateTelemetry() {}
}
```

## Elevator IO interface

```java
package first.robot.subsystems.akit;

import org.littletonrobotics.junction.AutoLog;
import org.wpilib.units.measure.Distance;

public interface ElevatorIO {

  @AutoLog
  public static class ElevatorIOInputs {
    public double positionMeters = 0.0;
    public double velocityMetersPerSec = 0.0;
    public double appliedVolts = 0.0;
    public double supplyCurrentAmps = 0.0;
    public double statorCurrentAmps = 0.0;
    public double temperatureCelsius = 0.0;
    public double targetPositionMeters = 0.0;
  }

  default void updateInputs(ElevatorIOInputs inputs) {}

  default void setTargetHeight(Distance height) {}

  default void stop() {}

  /** Update the sim telemetry */
  default void simIterate() {}

  /** Update the telemetry */
  default void updateTelemetry() {}
}
```

## Shooter IO interface

```java
package first.robot.subsystems.akit;

import org.littletonrobotics.junction.AutoLog;
import org.wpilib.units.measure.AngularVelocity;

public interface ShooterIO {

  @AutoLog
  public static class ShooterIOInputs {
    public double velocityRotationsPerSec = 0.0;
    public double appliedVolts = 0.0;
    public double supplyCurrentAmps = 0.0;
    public double statorCurrentAmps = 0.0;
    public double temperatureCelsius = 0.0;
    public double targetVelocityRotationsPerSec = 0.0;
  }

  default void updateInputs(ShooterIOInputs inputs) {}

  default void setTargetVelocity(AngularVelocity velocity) {}

  default void stop() {}

  /** Update the sim telemetry */
  default void simIterate() {}

  /** Update the telemetry */
  default void updateTelemetry() {}
}
```

{% hint style="info" %}
The `@AutoLog` annotation from AdvantageKit generates a class (e.g., `ArmIOInputsAutoLogged`) that handles serialization for log replay. See [AdvantageKit AutoLog documentation](https://docs.advantagekit.org/data-flow/recording-inputs/io-interfaces#autolog) for details.

If the `AutoLogged` class is not generated, check the `annotationProcessor` lines from [Step 0](setup.md), then clean the project with `gradlew clean` (Windows) or `./gradlew clean` (Linux/macOS) and build again. If the class still does not generate, try commenting out the `annotationProcessor wpi.java.deps.wpilibAnnotations()` line and clean again.
{% endhint %}

**Check:** build the project. It succeeds, and `ArmIOInputsAutoLogged`, `ElevatorIOInputsAutoLogged` and `ShooterIOInputsAutoLogged` can be imported (they are generated under `build/generated`).

Next: [Step 2: Implement the IO class](io-implementation.md)
