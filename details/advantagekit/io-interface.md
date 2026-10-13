# Step 1: Define the IO interface

Create an interface that defines what inputs your subsystem reads and what outputs it sends. Following AdvantageKit conventions, inputs are grouped in an `Inputs` class.

## Arm IO Interface

```java
package first.robot.subsystems.akit;

import org.littletonrobotics.junction.AutoLog;

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
  default void setTargetAngle(double rotations) {}

  /** Stop the arm motor. */
  default void stop() {}

  /** Update the sim telemetry */
  default void simIterate() {}

  /** Update the telemtry */
  default void updateTelemetry() {}
}
```

## Elevator IO Interface

```java
package first.robot.subsystems.akit;

import org.littletonrobotics.junction.AutoLog;

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

  default void setTargetHeight(double meters) {}

  default void stop() {}

  /** Update the sim telemetry */
  default void simIterate() {}

  /** Update the telemtry */
  default void updateTelemetry() {}
}
```

## Shooter IO Interface

```java
package first.robot.subsystems.akit;

import org.littletonrobotics.junction.AutoLog;

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

  default void setTargetVelocity(double rotationsPerSec) {}

  default void stop() {}

  /** Update the sim telemetry */
  default void simIterate() {}

  /** Update the telemtry */
  default void updateTelemetry() {}
}
```

{% hint style="info" %}
The `@AutoLog` annotation from AdvantageKit generates a class (e.g., `ArmIOInputsAutoLogged`) that handles serialization for log replay. See [AdvantageKit AutoLog documentation](https://docs.advantagekit.org/data-flow/recording-inputs/io-interfaces#autolog) for details.

If the @AutoLog fails to generate the coresponding AutoLogged class, check your build.gradle. In the dependencies section you should have

## Build.gradle

```groovy
dependencies {
    annotationProcessor wpi.java.deps.wpilibAnnotations()
    def akitJson = new groovy.json.JsonSlurper().parseText(
        new File(projectDir.getAbsolutePath() + "/vendordeps/AdvantageKit.json").text)
    annotationProcessor "org.littletonrobotics.akit:akit-autolog:$akitJson.version"
    // ...
}
```

This is the setup used by the YAMS AdvantageKit examples. After changing it, clean the project with `gradlew clean` (Windows) or `./gradlew clean` (Linux/macOS) and then build. If the class still does not generate, try commenting out the `annotationProcessor wpi.java.deps.wpilibAnnotations()` line and clean again.
{% endhint %}

Next: [Step 2: Implement the IO class](io-implementation.md)
