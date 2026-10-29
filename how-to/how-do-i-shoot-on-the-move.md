---
description: Where to learn shooting on the move, and how the ideas in those sources apply to YAMS.
---

# How do I shoot on the move?

Shooting on the move means scoring while the robot is still driving, instead of stopping to line up first.

We covered this in a YASS Weekly Workshop, **How To Program Target Tracking And Shooting On The Fly for FRC**. Watch it first:

{% embed url="https://www.youtube.com/watch?v=pa-HMLTHYvo" %}

People in the FRC community have already explained the math well, so this page sends you to them for it and keeps to the YAMS side: which YAMS call supplies each thing those sources ask for, and what the [Swerve Input Stream](../details/swerve-input-stream.md) does for you and what it does not. [Shoot on the move: worked example](shoot-on-the-move-example.md) then builds a complete robot that shoots on the move.

## Read these first

| Source | Read it for |
| ------ | ----------- |
| [Shooting on the Fly](https://blog.eeshwark.com/robotblog/shooting-on-the-fly) and [Shooting on the Fly, Part 2: The LUT Approach](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2), by Eeshwar | The best place to start. Part 1 explains the idea as plain vector math, then covers what goes wrong on a real robot: latency, a flywheel that slows and slips, spin and drag, lobbed shots, and a shooter that is itself moving. Part 2 is the practical side: building the lookup table, getting the shot's speed from the time of flight, shooters with a hood, and tuning the latency. |
| WPILib's [Fire Control](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/index.html) pages: [Static Shooting via Look-Up Table](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/static-shooting.html), [Dynamic Shooting via Time-of-Flight Recursion](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/dynamic-shooting.html), [Newton's Method for Dynamic Shooting](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/newton-shooting.html) and [Linear Drag (First-Order Air Friction)](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/linear-drag.html) | Three firing strategies and a drag correction, with interactive simulations that show where each one converges and where it does not. |
| [Shoot on the move from the code perspective](https://www.chiefdelphi.com/t/shoot-on-the-move-from-the-code-perspective/511815) (Chief Delphi) | A short explanation of the virtual target, with links to much of the rest. |
| [Time of Flight Determination](https://www.chiefdelphi.com/t/time-of-flight-determination/512542) (Chief Delphi) | The different ways teams measure or calculate the time of flight. |
| [Shoot While Move Code (1706)](https://www.chiefdelphi.com/t/shoot-while-move-code-1706/410494) (Chief Delphi) | Team 1706 walking through its 2022 code. |
| [Orbit 1690 Online Software Sessions](https://www.chiefdelphi.com/t/orbit-1690-online-software-sessions/467221) (Chief Delphi) | Recorded sessions from Team 1690 on their swerve and system control software. |
| [Mechanical Advantage 6328's `LaunchCalculator.java`](https://github.com/Mechanical-Advantage/RobotCode2026Public/blob/main/src/main/java/org/littletonrobotics/frc2026/subsystems/launcher/LaunchCalculator.java) | A complete launch calculator from a 2026 robot. |
| [Hammerheads 5000's `TurretCalculator.java`](https://github.com/hammerheads5000/2026Rebuilt/blob/main/src/main/java/frc/robot/subsystems/turret/TurretCalculator.java), with a [build thread post](https://www.chiefdelphi.com/t/frc-5000-hammerheads-2026-build-thread-open-alliance/507502/129) that walks through it | A complete shot calculator for a turret from a 2026 robot. |

{% hint style="info" %}
The WPILib fire control pages are a work in progress in [wpilib-docs #3242](https://github.com/wpilibsuite/wpilib-docs/pull/3242). They have not been merged, so the links go to the pull request's preview and may move.
{% endhint %}

## The idea in one picture

A game piece keeps the robot's velocity when it leaves the shooter, so a robot that aims straight at the target while driving sideways misses. The sources above all solve the same problem: work out a **lead** that cancels the robot's motion.

<figure><img src="../.gitbook/assets/howto-sotm-virtual-target.gif" alt="Animation of two robots seen from above, both driving to the right at the same speed and shooting at a target. The first aims straight at the target, and every shot drifts to the right of it and misses. The second aims at a virtual target to the left of the real one, and every shot drifts into the real target."><figcaption><p>The shot keeps the robot's velocity, so the aim has to lead it.</p></figcaption></figure>

## A primer: vector subtraction

**Read:** Eeshwar's [Shooting on the Fly](https://blog.eeshwark.com/robotblog/shooting-on-the-fly) and [Shooting on the Fly, Part 2: The LUT Approach](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2).

Before you pick a firing strategy, it helps to see the lead as one subtraction, which is how Eeshwar's posts explain it:

1. The time of flight in your lookup table gives the horizontal speed of the standing shot: the distance divided by the time.
2. Point a velocity of that speed at the target. This is the velocity the game piece needs over the ground.
3. Subtract the robot's velocity. What is left is the velocity the shooter has to give the game piece.
4. The direction of that vector is where to aim, and its length is the speed the shot needs.

<figure><img src="../.gitbook/assets/howto-sotm-vector-subtraction.gif" alt="Animation of a robot driving to the right with a target ahead and to its right, built up in four steps. A purple arrow from the robot points at the target: the velocity the game piece needs over the ground. A green arrow points right: the robot's velocity. A dashed green arrow of the same length runs backwards from the tip of the purple arrow: the subtraction. An orange arrow from the robot to the end of the dashed arrow is the velocity the shooter has to give, and the robot turns to face along it."><figcaption><p>The orange arrow is what the shooter has to give: the purple arrow minus the green one.</p></figcaption></figure>

Two things follow from this, and every firing strategy below has to deal with both. The **aim** changes when the robot moves, and so does the **speed** of the shot: drive toward the target and the shooter needs less, drive away and it needs more. The strategies below are different ways of working out that same lead from the time of flight in your table.

<figure><img src="../.gitbook/assets/howto-sotm-vector-sweep.gif" alt="Animation of a robot with a target straight ahead, driving at 2 meters per second in every direction in turn. The velocity needed over the ground stays at 4.3 meters per second toward the target. Driving toward the target, the velocity the shooter gives shrinks to 2.3 meters per second. Driving away, it grows to 6.3 meters per second. Driving across, it stays near 4.7 meters per second and swings about 25 degrees back against the motion."><figcaption><p>Toward the target the shot needs less speed, away it needs more, and across it the aim swings back.</p></figcaption></figure>

## How the sources' ideas apply to YAMS

Each row is a concept from the sources, where to read about it, and the YAMS piece that does that job.

| Concept | Read about it in | In YAMS |
| ------- | ---------------- | ------- |
| The robot's position and field velocity | Eeshwar's [part 1](https://blog.eeshwark.com/robotblog/shooting-on-the-fly), "Prerequisites" | `drive.getPose()` and `drive.getFieldRelativeSpeed()` on the [Swerve Drive](../details/swerve-drive.md) |
| A lookup table of shots that score, measured standing still | [Static Shooting via Look-Up Table](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/static-shooting.html), and Eeshwar's [part 2](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2) | WPILib's [`InterpolatingDoubleTreeMap`](https://github.wpilib.org/allwpilib/docs/development/java/org/wpilib/math/interpolation/InterpolatingDoubleTreeMap.html), with the result sent to a [FlyWheel](../details/shooters.md) |
| A flywheel speed that changes every loop | Eeshwar's [part 1](https://blog.eeshwark.com/robotblog/shooting-on-the-fly), "Wait, The Shooter Speed Changed?" | `FlyWheel.run(Supplier)`, which reads the supplier every loop |
| Turning to face a point while the driver keeps driving | Every source | [`withAim`](../details/swerve-input-stream.md#aim-at-a-target), which takes a supplier so the point can move every loop |
| A time of flight lead, as a virtual target | [Dynamic Shooting via Time-of-Flight Recursion](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/dynamic-shooting.html), and [Shoot on the move from the code perspective](https://www.chiefdelphi.com/t/shoot-on-the-move-from-the-code-perspective/511815) | You calculate it. Give the result to `withAim` |
| A time of flight lead, as a shot vector with the robot's velocity subtracted | Eeshwar's [part 1](https://blog.eeshwark.com/robotblog/shooting-on-the-fly), "The Math Implementation", and [part 2](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2), "Getting Horizontal Velocity from ToF" | You calculate it. Give its direction to `withAim`, [`withHeading`](../details/swerve-input-stream.md#heading-snap-control) or a turret, and its speed to the flywheel |
| Latency compensation: aim from where the robot will be | Eeshwar's [part 1](https://blog.eeshwark.com/robotblog/shooting-on-the-fly), "Latency Compensation", and [part 2](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2), "The Latency Constant" | [`withAimLookahead`](../details/swerve-input-stream.md#aim-lookahead) |
| A heading that keeps up with a moving aim point | Specific to turning the whole robot. The sources mostly aim a turret | [`withAimFeedforward`](../details/swerve-input-stream.md#aim-feedforward), and the rotation controller in [`SwerveDriveConfig`](../details/mechanism-configs/swerve-drive-config.md) |
| A shooter that does not point out of the front, or a small aim correction for spin | Eeshwar's [part 1](https://blog.eeshwark.com/robotblog/shooting-on-the-fly), "The Magnus Effect and Air Resistance" | [`withAimHeadingOffset`](../details/swerve-input-stream.md#aim-heading-offset) |
| Only shooting when the shot is trustworthy | [Dynamic Shooting via Time-of-Flight Recursion](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/dynamic-shooting.html), "Shot Quality in Practice", and Eeshwar's [part 1](https://blog.eeshwark.com/robotblog/shooting-on-the-fly), "Shooter Wheel Dynamics" | [`aimLock`](../details/swerve-input-stream.md#aim-lock) for the heading, and the flywheel's `near` trigger for the speed: see [How do I shoot with a shooter?](how-to-shoot-with-a-shooter.md) |
| A hood, or a turret | Eeshwar's [part 2](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2), the hood variants, and the two team calculators | An [Arm](../details/arms.md) or a [Pivot](../details/turrets-wrists.md), each in its own subsystem |

## What `SwerveInputStream` does, and what it leaves to you

The stream's aim mode uses the simplest method there is, **point at the pose**. [`withAim`](../details/swerve-input-stream.md#aim-at-a-target) takes a target pose, works out the direction from the robot to it, and lets the drive's rotation PID controller turn the front of the robot that way while the left stick keeps translating.

<figure><img src="../.gitbook/assets/sis-aim.gif" alt="Animation of a robot driven left and right by the left stick while it stays facing a target on the field"><figcaption><p><code>withAim</code>: the left stick drives, and the robot keeps facing the target it is given.</p></figcaption></figure>

Four options refine it: [`withAimLookahead`](../details/swerve-input-stream.md#aim-lookahead), [`withAimFeedforward`](../details/swerve-input-stream.md#aim-feedforward), [`withAimHeadingOffset`](../details/swerve-input-stream.md#aim-heading-offset) and [`aimLock`](../details/swerve-input-stream.md#aim-lock). The [Swerve Input Stream](../details/swerve-input-stream.md#aim-at-a-target) page explains each one.

<figure><img src="../.gitbook/assets/sis-aim-lookahead.gif" alt="Animation of a robot driving to the right with a faded copy of itself a short way ahead along its velocity. A grey dashed line runs from the robot to the target, and an orange dashed line runs from the faded copy to the target. The robot turns to the direction of the orange line."><figcaption><p><code>withAimLookahead</code>: the robot turns to the direction it will need a moment from now.</p></figcaption></figure>

Even with all four, the stream on its own does **not** shoot on the move the way you might expect:

* **The lookahead is one fixed time.** Aiming from where the robot will be in 0.3 seconds is the same as leading the target by 0.3 seconds of travel. That is only the right lead when the time of flight is 0.3 seconds, which is true at one distance. Use the lookahead for what the sources call latency: the short, constant delay before the game piece leaves the robot.
* **It only turns the robot.** A lead changes the distance of the shot as well as its direction. The stream knows nothing about your flywheel, your hood or your tables, so it cannot change the shot to match.
* **It assumes a straight line.** The lookahead takes the robot's velocity right now and assumes it stays the same.
* **Without a feedforward, the rotation lags.** The PID controller only reacts to error, so while the robot drives past the target it trails a little behind.

So let the stream do the turning, and give it the right thing to turn to. The lead itself comes from one of the methods in the sources, which is the part you write.

## A worked example

[Shoot on the move: worked example](shoot-on-the-move-example.md) goes from an empty project to a robot that aims, sets its shooter speed from a lookup table, and feeds only when the shot is ready, in Java with Commands v2 and Commands v3.

## Tuning the YAMS side

The sources cover how to tune the shot itself. These are the YAMS settings that go with their advice:

| Their advice | The YAMS setting |
| ------------ | ---------------- |
| Get the standing shot consistent before anything else (Eeshwar's [part 1](https://blog.eeshwark.com/robotblog/shooting-on-the-fly), "Prerequisites") | The flywheel's gains: see WPILib's [Tuning a Flywheel Velocity Controller](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/tuning-flywheel.html) and [How do I shoot with a shooter?](how-to-shoot-with-a-shooter.md) |
| Tune the latency constant by driving straight across the front of the target (Eeshwar's [part 2](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2), "The Latency Constant") | The time you give [`withAimLookahead`](../details/swerve-input-stream.md#aim-lookahead). Start at 0.1 seconds |
| A moving shot is only as good as the pose it starts from | Vision measurements into the drive's pose estimate: see WPILib's [Pose Estimators](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/state-space/state-space-pose-estimators.html) |
| Watch whether the heading keeps up as you drive past | The rotation controller in [`SwerveDriveConfig`](../details/mechanism-configs/swerve-drive-config.md), then [`withAimFeedforward`](../details/swerve-input-stream.md#aim-feedforward) |
| Hold fire when the shot is not trustworthy ([Dynamic Shooting via Time-of-Flight Recursion](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/dynamic-shooting.html), "Shot Quality in Practice") | The tolerance you give [`aimLock`](../details/swerve-input-stream.md#aim-lock), and the flywheel's `near` tolerance |

## WPILib building blocks

The released WPILib documentation covers the classes the example uses. These pages are written for WPILib 2026. The ideas are the same in 2027, though some classes were renamed: `ChassisSpeeds`, for example, is `ChassisVelocities`.

| Page | Why it matters here |
| ---- | ------------------- |
| [Coordinate System](https://docs.wpilib.org/en/stable/docs/software/basic-programming/coordinate-system.html) | Which way X, Y and a positive rotation point. |
| [Translation, Rotation, and Pose](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/geometry/pose.html) | The classes that the distance and direction to the target are calculated with. |
| [Transformations](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/geometry/transformations.html) | Getting from the center of the robot to where the shooter really is. |
| [Introduction to Kinematics and The ChassisSpeeds Class](https://docs.wpilib.org/en/stable/docs/software/kinematics-and-odometry/intro-and-chassis-speeds.html) | Robot relative and field relative velocity. |
