---
icon: download
---

# Step 0: Install AdvantageKit and set up Robot

Before the IO classes can log anything, the project needs the AdvantageKit library, its code generator in `build.gradle`, and a `Robot` class that starts the AdvantageKit logger. This page sets up a Commands v2 project that already builds with YAMS, the same way the [YAMS AdvantageKit example](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands2/advantage_kit) does.

{% hint style="info" %}
AdvantageKit is a separate library from Team 6328, and its 2027 releases are still alphas. If anything here does not match what you see, the [AdvantageKit installation docs for existing projects](https://docs.advantagekit.org/getting-started/installation/existing-projects) are the source of truth.
{% endhint %}

{% stepper %}
{% step %}
### Install the vendor library

In VS Code, open the WPILib command palette, choose **WPILib: Manage Vendor Libraries**, then **Install new library (online)**, and paste:

```
https://github.com/Mechanical-Advantage/AdvantageKit/releases/latest/download/AdvantageKit.json
```

This adds `vendordeps/AdvantageKit.json` to your project. The YAMS 2027 examples were built against AdvantageKit `27.0.0-alpha-5`.

**Check:** `vendordeps/AdvantageKit.json` exists in your project.
{% endstep %}

{% step %}
### Add the AutoLog code generator to `build.gradle`

The `@AutoLog` annotation used in [Step 1](io-interface.md) generates a class for each inputs class when the code builds. That needs an annotation processor line in the `dependencies` block of `build.gradle`:

```groovy
dependencies {
    annotationProcessor wpi.java.deps.wpilibAnnotations()
    // AdvantageKit's @AutoLog code generator, at the same version as the vendor library.
    def akitJson = new groovy.json.JsonSlurper().parseText(
        new File(projectDir.getAbsolutePath() + "/vendordeps/AdvantageKit.json").text)
    annotationProcessor "org.littletonrobotics.akit:akit-autolog:$akitJson.version"

    // ... the rest of your dependencies stay as they are
}
```

This is the setup used by the YAMS AdvantageKit examples. AdvantageKit's own docs add more to `build.gradle` (such as recording git metadata); see the [AdvantageKit installation docs](https://docs.advantagekit.org/getting-started/installation/existing-projects) if you want those.

**Check:** the project still builds.
{% endstep %}

{% step %}
### Make `Robot` a `LoggedRobot`

`Robot` extends AdvantageKit's `LoggedRobot` instead of WPILib's `TimedRobot`, and starts the logger in its constructor before anything else is created. It also defines `currentMode`, which [Log replay](log-replay.md) uses to choose the IO class.

```java
package first.robot;

import org.littletonrobotics.junction.LogFileUtil;
import org.littletonrobotics.junction.LoggedRobot;
import org.littletonrobotics.junction.Logger;
import org.littletonrobotics.junction.networktables.NT4Publisher;
import org.littletonrobotics.junction.wpilog.WPILOGReader;
import org.littletonrobotics.junction.wpilog.WPILOGWriter;
import org.wpilib.command2.CommandScheduler;
import org.wpilib.framework.RobotBase;

public class Robot extends LoggedRobot {
  public static enum Mode {
    /** Running on a real robot. */
    REAL,
    /** Running a physics simulator. */
    SIM,
    /** Replaying from a log file. */
    REPLAY
  }

  /** Change to Mode.REPLAY to replay a log in simulation. */
  public static final Mode simMode = Mode.SIM;
  public static final Mode currentMode = RobotBase.isReal() ? Mode.REAL : simMode;

  private final RobotContainer robotContainer;

  public Robot() {
    switch (currentMode) {
      case REAL:
        // On the SystemCore: write a log file and publish to NetworkTables.
        Logger.addDataReceiver(new WPILOGWriter());
        Logger.addDataReceiver(new NT4Publisher());
        break;

      case SIM:
        // In simulation: publish to NetworkTables.
        Logger.addDataReceiver(new NT4Publisher());
        break;

      case REPLAY:
        // Replaying a log: read it, and write the replayed values to a new log next to it.
        setUseTiming(false); // Run as fast as possible
        String logPath = LogFileUtil.findReplayLog();
        Logger.setReplaySource(new WPILOGReader(logPath));
        Logger.addDataReceiver(new WPILOGWriter(LogFileUtil.addPathSuffix(logPath, "_sim")));
        break;
    }

    // Start the AdvantageKit logger before creating the subsystems.
    Logger.start();

    robotContainer = new RobotContainer();
  }

  @Override
  public void robotPeriodic() {
    CommandScheduler.getInstance().run();
  }
}
```

Keep the rest of your `Robot` methods (autonomous, teleop and so on) as they are. `first/Main.java` does not change: it still starts `first.robot.Robot`.

{% hint style="info" %}
**Commands v3:** `Robot` extends `LoggedRobot` in the same way, with the same `Mode` and logger setup, and `robotPeriodic()` runs `Scheduler.getDefault().run()`. See the [Commands v3 AdvantageKit example](https://github.com/Yet-Another-Software-Suite/YAMS/tree/master/examples/commands3/advantage_kit).
{% endhint %}

**Check:** run the code in simulation and open AdvantageScope. An `AdvantageKit` table appears in NetworkTables next to the YAMS `Mechanisms` table.
{% endstep %}
{% endstepper %}

## If it did not work

* **`org.littletonrobotics.junction` cannot be found:** the vendor library is not installed. Repeat [1. Install the vendor library](#install-the-vendor-library), then build again.
* **`ArmIOInputsAutoLogged` cannot be found later:** the annotation processor line is missing or wrong. Check step 2 above, clean the project with `gradlew clean` (Windows) or `./gradlew clean` (Linux/macOS) and build again.

Next: [Step 1: Define the IO interface](io-interface.md)
