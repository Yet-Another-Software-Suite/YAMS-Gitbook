---
description: Aim a swerve drive and set a flywheel so that shots land, standing still or while driving.
---

# How do I shoot on the move?

Shooting on the move means scoring while the robot is still driving, instead of stopping to line up first. YAMS gives you the pieces: a swerve drive that knows its pose and velocity, a [Swerve Input Stream](../details/swerve-input-stream.md) that turns the robot to face a point while the driver keeps driving, and a flywheel that follows a changing speed. The part you write is a small calculator that decides where to aim and how fast to spin.

This guide builds that on top of the [Swerve Drive](../tutorials/swerve-drive.md) and [Shooter](../tutorials/shooter-flywheels.md) tutorials. It follows the three strategies of the fire control section being written for the WPILib documentation, and each has its own page here.

{% hint style="info" %}
The code is shown for Java with Commands v2 and Commands v3. The C++ `SwerveInputStream` has the same options, so the same approach works there. Every snippet compiles against YAMS, but none of it has been run on a robot: the numbers in the tables are placeholders for your own measurements.
{% endhint %}

## The idea

A game piece keeps the robot's velocity when it leaves the shooter. If the robot is moving sideways, the shot drifts sideways and misses. The fix is to **lead** the shot: aim at a **virtual target** that is moved against the robot's motion by exactly as far as the shot will drift.

<figure><img src="../.gitbook/assets/howto-sotm-virtual-target.gif" alt="Animation of two robots seen from above, both driving to the right at the same speed and shooting at a target. The first aims straight at the target, and every shot drifts to the right of it and misses. The second aims at a virtual target to the left of the real one, and every shot drifts into the real target."><figcaption><p>The shot keeps the robot's velocity. Aiming at the virtual target cancels the drift.</p></figcaption></figure>

The drift is the robot's velocity multiplied by the **time of flight**. The time of flight depends on the distance, and the distance changes once you aim at the virtual target, so the strategies differ in how they settle on it.

{% hint style="success" %}
**Read this too:** Eeshwar's blog series, [Shooting on the fly, part 1](https://blog.eeshwark.com/robotblog/shooting-on-the-fly) and [part 2](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2), is the best companion to these pages. Part 1 covers the vector math. Part 2 is the practical side: building the lookup table, getting the shot's speed from the time of flight, what to do with a hood, and how to tune the latency.
{% endhint %}

## The three strategies

| Strategy | What it does | Use it when |
| -------- | ------------ | ----------- |
| [1. A lookup table, standing still](shoot-on-the-move/lookup-table.md) | Reads the flywheel speed for the distance from a table, and aims at the real target. | You stop to shoot. Every other strategy is built on this table, so start here. |
| [2. Time of flight recursion](shoot-on-the-move/time-of-flight-recursion.md) | Leads the target by the time of flight, and repeats until the time stops changing. | You want to shoot while driving. This is what most teams use. |
| [3. Newton's method](shoot-on-the-move/newtons-method.md) | Finds the same time of flight with a smarter step, in fewer passes. | You shoot while driving fast toward or away from the target, where the recursion is slow to settle. |

These are the strategies of WPILib's [Fire Control](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/index.html) pages: [Static Shooting via Look-Up Table](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/static-shooting.html), [Dynamic Shooting via Time-of-Flight Recursion](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/dynamic-shooting.html) and [Newton's Method for Dynamic Shooting](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/newton-shooting.html). Each page here links to the one it follows.

{% hint style="info" %}
The WPILib pages on fire control are a work in progress in [wpilib-docs #3242](https://github.com/wpilibsuite/wpilib-docs/pull/3242). They have not been merged, so the links go to the pull request's preview and may move.
{% endhint %}

### Another way to lead: subtract the robot's velocity

Eeshwar's [Shooting on the fly, part 1](https://blog.eeshwark.com/robotblog/shooting-on-the-fly) and [part 2](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2) reach the lead a different way, with no repeated passes:

1. The table gives the time of flight for the distance, so the shot's horizontal speed is simply the distance divided by the time of flight.
2. Point a velocity of that speed straight at the target. This is the velocity the game piece needs over the ground.
3. Subtract the robot's field relative velocity from it. What is left is the velocity the shooter has to give the game piece.
4. The direction of that vector is where to aim. Its length is the horizontal speed you need, and the table read in reverse turns that speed back into a flywheel speed.

It suits a turret well, because the result is an angle and a speed. It works in the **horizontal** speed of the shot, which matters for a lobbed shot: at a 60 degree exit angle the horizontal speed is only half of the total. With YAMS, give the direction to [`withHeading`](../details/swerve-input-stream.md#heading-snap-control) on the stream, or to a turret, and the speed to the flywheel.

## What `SwerveInputStream` does, and what it leaves to you

The stream's aim mode uses the simplest method there is, **point at the pose**. [`withAim`](../details/swerve-input-stream.md#aim-at-a-target) takes a target pose, works out the direction from the robot to it, and lets the drive's rotation PID controller turn the front of the robot that way while the left stick keeps translating.

<figure><img src="../.gitbook/assets/sis-aim.gif" alt="Animation of a robot driven left and right by the left stick while it stays facing a target on the field"><figcaption><p><code>withAim</code>: the left stick drives, and the robot keeps facing the target it is given.</p></figcaption></figure>

Four options refine it:

| Option | What it adds |
| ------ | ------------ |
| [`withAimLookahead(time)`](../details/swerve-input-stream.md#aim-lookahead) | Aims from where the robot will be after a fixed time, instead of where it is now. |
| [`withAimFeedforward(feedforward)`](../details/swerve-input-stream.md#aim-feedforward) | Adds a feedforward to the rotation, so the robot keeps up with a target whose direction keeps changing. |
| [`withAimHeadingOffset(angle)`](../details/swerve-input-stream.md#aim-heading-offset) | Rotates the aim heading, for a shooter that does not point out of the front of the robot. |
| [`aimLock(tolerance)`](../details/swerve-input-stream.md#aim-lock) | A trigger that is true while the robot is aiming and facing its aim heading. |

<figure><img src="../.gitbook/assets/sis-aim-lookahead.gif" alt="Animation of a robot driving to the right with a faded copy of itself a short way ahead along its velocity. A grey dashed line runs from the robot to the target, and an orange dashed line runs from the faded copy to the target. The robot turns to the direction of the orange line."><figcaption><p><code>withAimLookahead</code>: the robot turns to the direction it will need a moment from now.</p></figcaption></figure>

Even with all four, the stream on its own does **not** shoot on the move the way you might expect:

* **The lookahead is one fixed time.** Aiming from where the robot will be in 0.3 seconds is the same as leading the target by 0.3 seconds of travel. That is only the right lead when the time of flight is 0.3 seconds, which is true at one distance. A real time of flight changes with distance, so a fixed lookahead is right in one place and wrong everywhere else. Use it for what it is good at: the short, constant delay between the calculation and the game piece leaving the robot.
* **It only turns the robot.** A lead changes the distance of the shot as well as its direction. The stream knows nothing about your flywheel, your hood or your tables, so it cannot change the shot to match.
* **It assumes a straight line.** The lookahead takes the robot's velocity right now and assumes it stays the same. It is at its best while driving straight, and at its worst in a tight curve.
* **Without a feedforward, the rotation lags.** The PID controller only reacts to error, so while the robot drives past the target it trails a little behind. `withAimFeedforward` reduces this, and `aimLock` tells you when the robot has caught up.

So let the stream do the turning, and give it the right thing to turn to. In this guide the calculator works out the virtual target from the real time of flight, `withAim` faces it, `withAimLookahead` covers the release delay, and `aimLock` decides when to feed.

## What you need first

* A swerve drive that knows where it is. A moving shot is only as good as the pose it starts from. WPILib's [Pose Estimators](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/state-space/state-space-pose-estimators.html) page explains how vision corrects the pose.
* A rotation controller on the drive. `withAim` uses the one you gave [`SwerveDriveConfig`](../details/mechanism-configs/swerve-drive-config.md), which the swerve tutorial already sets.
* A flywheel that reaches its speed quickly and holds it. See [How to shoot with a Shooter?](how-to-shoot-with-a-shooter.md) and WPILib's [Tuning a Flywheel Velocity Controller](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/tuning-flywheel.html).

## 1. Let the calculator read the drive

Every strategy needs to know where the robot is, and the moving ones need to know how fast it is moving on the field. Add these to the drive class from the [Swerve Drive](../tutorials/swerve-drive.md) tutorial.

{% tabs %}
{% tab title="Java, Commands v2" %}
{% code title="subsystems/SwerveSubsystem.java" %}
```java
/** Where the robot is on the field. */
public Pose2d getPose() {
  return drive.getPose();
}

/** How fast the robot is moving on the field. */
public ChassisVelocities getFieldVelocity() {
  return drive.getFieldRelativeSpeed();
}
```
{% endcode %}
{% endtab %}

{% tab title="Java, Commands v3" %}
{% code title="mechanisms/SwerveMechanism.java" %}
```java
/** Where the robot is on the field. */
public Pose2d getPose() {
  return drive.getPose();
}

/** How fast the robot is moving on the field. */
public ChassisVelocities getFieldVelocity() {
  return drive.getFieldRelativeSpeed();
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

## 2. Let the flywheel follow a changing speed

The shot's flywheel speed changes as the robot moves, so the shooter needs a command that takes a supplier. `FlyWheel.run(Supplier)` reads it every loop. Add this to the shooter class from the [Shooter](../tutorials/shooter-flywheels.md) tutorial, along with an import of `java.util.function.Supplier`.

{% tabs %}
{% tab title="Java, Commands v2" %}
{% code title="subsystems/ShooterSubsystem.java" %}
```java
/** Follow a speed that changes every loop. This command does not end on its own. */
public Command track(Supplier<AngularVelocity> speed) {
  return shooter.run(speed);
}
```
{% endcode %}
{% endtab %}

{% tab title="Java, Commands v3" %}
{% code title="mechanisms/ShooterMechanism.java" %}
```java
/** Follow a speed that changes every loop. This command does not end on its own. */
public Command track(Supplier<AngularVelocity> speed) {
  return shooter.run(speed);
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

## 3. Pick a strategy and write the calculator

Each strategy page adds one method to a `ShotCalculator` class. Start with the first, because the other two reuse its tables.

* [Strategy 1: A lookup table, standing still](shoot-on-the-move/lookup-table.md)
* [Strategy 2: Time of flight recursion](shoot-on-the-move/time-of-flight-recursion.md)
* [Strategy 3: Newton's method](shoot-on-the-move/newtons-method.md)

## 4. Aim and spin up together

While the driver holds the right bumper, the stream turns the robot toward the shot's target and the shooter follows the shot's speed. `shot()` calls the strategy you picked: this one uses the recursion. Swapping strategies only changes that one line.

{% tabs %}
{% tab title="Java, Commands v2" %}
{% code title="RobotContainer.java" %}
```java
private final SwerveSubsystem drive = new SwerveSubsystem();
private final ShooterSubsystem shooter = new ShooterSubsystem();

/** The shot for where the robot is, and how it is moving, right now. */
private ShotCalculator.Shot shot() {
  return ShotCalculator.recursive(drive.getPose(), drive.getFieldVelocity());
}

// ...

  SwerveInputStream input = drive.getChassisSpeedsSupplier(
          () -> -driverController.getLeftY(),
          () -> -driverController.getLeftX(),
          () -> -driverController.getRightX())
      // While the right bumper is held, face the shot's target instead of
      // turning with the right stick. The left stick keeps driving.
      .withAim(() -> shot().target(), driverController.rightBumper())
      // Aim from where the robot will be when the game piece leaves it.
      .withAimLookahead(Seconds.of(0.1));
  drive.setDefaultCommand(drive.drive(input));

  // While the right bumper is held, keep the flywheel at the speed the shot needs.
  driverController.rightBumper().whileTrue(shooter.track(() -> shot().flywheelSpeed()));

  // True while the robot is aiming and within 2 degrees of the aim heading.
  Trigger aimed = input.aimLock(Degrees.of(2));
```
{% endcode %}
{% endtab %}

{% tab title="Java, Commands v3" %}
{% code title="Robot.java" %}
```java
final SwerveMechanism drive = new SwerveMechanism();
final ShooterMechanism shooter = new ShooterMechanism();

/** The shot for where the robot is, and how it is moving, right now. */
ShotCalculator.Shot shot() {
  return ShotCalculator.recursive(drive.getPose(), drive.getFieldVelocity());
}

// ...

@Override
public void robotPeriodic() {
  drive.periodic();
  shooter.periodic();
  scheduler.run();
  Telemetry.log("Scheduler", scheduler, Scheduler.proto);
}

@Override
public void simulationPeriodic() {
  drive.simulationPeriodic();
  shooter.simulationPeriodic();
}
```
{% endcode %}

{% code title="ExampleTeleop.java" %}
```java
SwerveInputStream input = robot.drive.getChassisSpeedsSupplier(
        () -> -robot.exampleController.getLeftY(),
        () -> -robot.exampleController.getLeftX(),
        () -> -robot.exampleController.getRightX())
    // While the right bumper is held, face the shot's target instead of
    // turning with the right stick. The left stick keeps driving.
    .withAim(() -> robot.shot().target(), robot.exampleController.rightBumper())
    // Aim from where the robot will be when the game piece leaves it.
    .withAimLookahead(Seconds.of(0.1));
robot.drive.setDefaultCommand(robot.drive.drive(input));

// While the right bumper is held, keep the flywheel at the speed the shot needs.
robot.exampleController.rightBumper()
    .whileTrue(robot.shooter.track(() -> robot.shot().flywheelSpeed()));

// True while the robot is aiming and within 2 degrees of the aim heading.
Trigger aimed = input.aimLock(Degrees.of(2));
```
{% endcode %}
{% endtab %}
{% endtabs %}

The new imports are `org.wpilib.units.Units.Degrees` (static), `org.wpilib.units.Units.Seconds` (static), `yams.commands2.swerve.SwerveInputStream` for Commands v2, and `org.wpilib.units.Units.Degrees` (static), `org.wpilib.units.Units.Seconds` (static), `org.wpilib.command3.Trigger`, `yams.commands3.swerve.SwerveInputStream` for Commands v3.

**For your robot:** start the lookahead at the delay you can measure, from the calculation to the game piece leaving the shooter, and tune from there. It is often 100 to 200 milliseconds. If your shooter does not point out of the front of the robot, add [`withAimHeadingOffset`](../details/swerve-input-stream.md#aim-heading-offset).

## 5. Feed only when the shot is ready

Do not feed a game piece until the robot faces the aim heading and the flywheel is at the shot's speed. `aimed` from the last step is the first half. [How to shoot with a Shooter?](how-to-shoot-with-a-shooter.md) shows how to gate a feeder on the flywheel's speed with a debounce, which is the second half. Combine the two with `aimed.and(...)`.

## Tuning it

* Get the standing shot right first. If shots miss while the robot is still, moving will not fix them.
* Then drive slowly sideways past the target. If shots land behind the robot's direction of travel, your times of flight are too short. If they land ahead of it, they are too long.
* Watch `aimed` while you drive past. If the robot trails the aim heading, raise the rotation controller's gains in [`SwerveDriveConfig`](../details/mechanism-configs/swerve-drive-config.md), or add a [`withAimFeedforward`](../details/swerve-input-stream.md#aim-feedforward).
* Driving straight at the target or straight away from it is the hardest case. WPILib's [Dynamic Shooting via Time-of-Flight Recursion](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/dynamic-shooting.html) page shows why, and how to tell a trustworthy shot from a fragile one.
* Tune the lookahead last, with the test from [part 2](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2) of Eeshwar's series: drive in a straight line across the front of the target at a steady speed. If the shots land behind where you are aiming, raise the lookahead. If they land ahead, lower it. Start at 0.1 seconds.

## Further reading

### WPILib documentation

{% hint style="info" %}
The WPILib pages on fire control are a work in progress in [wpilib-docs #3242](https://github.com/wpilibsuite/wpilib-docs/pull/3242). They have not been merged, so the links go to the pull request's preview and may move.
{% endhint %}

| Page | What it covers |
| ---- | -------------- |
| [Fire Control](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/index.html) | The start of the section, and the idea of leading and adjusting. |
| [Static Shooting via Look-Up Table](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/static-shooting.html) | Strategy 1: building and using a firing table. |
| [Dynamic Shooting via Time-of-Flight Recursion](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/dynamic-shooting.html) | Strategy 2: the recursion, with an interactive simulation of where it converges. |
| [Newton's Method for Dynamic Shooting](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/newton-shooting.html) | Strategy 3: Newton's method, with the math step by step. |
| [Linear Drag (First-Order Air Friction)](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/linear-drag.html) | A correction for air drag at high robot speeds, which works with strategies 2 and 3. |

The released WPILib documentation covers the building blocks. These pages are written for WPILib 2026. The ideas are the same in 2027, though some classes were renamed: `ChassisSpeeds`, for example, is `ChassisVelocities`.

| Page | Why it matters here |
| ---- | ------------------- |
| [Coordinate System](https://docs.wpilib.org/en/stable/docs/software/basic-programming/coordinate-system.html) | Which way X, Y and a positive rotation point. Get this wrong and the virtual target moves the wrong way. |
| [Translation, Rotation, and Pose](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/geometry/pose.html) | The classes that the distance and direction to the target are calculated with. |
| [Transformations](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/geometry/transformations.html) | Getting from the center of the robot to where the shooter really is. |
| [Introduction to Kinematics and The ChassisSpeeds Class](https://docs.wpilib.org/en/stable/docs/software/kinematics-and-odometry/intro-and-chassis-speeds.html) | Robot relative and field relative velocity. |
| [Pose Estimators](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/state-space/state-space-pose-estimators.html) | Combining odometry with vision for a pose you can trust. |
| [Tuning a Flywheel Velocity Controller](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/tuning-flywheel.html) | Getting the flywheel to its speed quickly. |
| [`InterpolatingDoubleTreeMap`](https://github.wpilib.org/allwpilib/docs/release/java/edu/wpi/first/math/interpolation/InterpolatingDoubleTreeMap.html) (API) | The lookup table used by `ShotCalculator`. |

### FRC write-ups and code

| Resource | What you get from it |
| -------- | -------------------- |
| [Shoot on the move from the code perspective](https://www.chiefdelphi.com/t/shoot-on-the-move-from-the-code-perspective/511815) (Chief Delphi) | A good place to start. It explains the virtual target and links to most of the resources below. |
| [Shooting on the fly, part 1](https://blog.eeshwark.com/robotblog/shooting-on-the-fly) and [part 2](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2) (Eeshwar) | Part 1 is the vector math. Part 2 builds the lookup table, gets the horizontal speed of the shot from the time of flight, covers shooters with a hood, and tunes the latency. |
| [Time of Flight Determination](https://www.chiefdelphi.com/t/time-of-flight-determination/512542) (Chief Delphi) | The different ways teams find the time of flight. |
| [Shoot While Move Code (1706)](https://www.chiefdelphi.com/t/shoot-while-move-code-1706/410494) (Chief Delphi) | Team 1706 walks through its 2022 code: a table of time of flight against distance, a virtual goal, and a few repeats of the calculation. |
| [Orbit 1690 Online Software Sessions](https://www.chiefdelphi.com/t/orbit-1690-online-software-sessions/467221) (Chief Delphi) | Recorded sessions from Team 1690 on their swerve and system control software, which other teams point to for the shoot on the move math. |
| [Mechanical Advantage 6328's `LaunchCalculator.java`](https://github.com/Mechanical-Advantage/RobotCode2026Public/blob/main/src/main/java/org/littletonrobotics/frc2026/subsystems/launcher/LaunchCalculator.java) | The launch calculation from their 2026 robot code. |
| [Hammerheads 5000's `TurretCalculator.java`](https://github.com/hammerheads5000/2026Rebuilt/blob/main/src/main/java/frc/robot/subsystems/turret/TurretCalculator.java) | The shot calculation for a turret from their 2026 robot code, with a [build thread post](https://www.chiefdelphi.com/t/frc-5000-hammerheads-2026-build-thread-open-alliance/507502/129) that walks through it. |

None of the team code uses YAMS, so the class and method names differ from yours. Read it for the math, and use the steps above to find the YAMS call that supplies each input.
