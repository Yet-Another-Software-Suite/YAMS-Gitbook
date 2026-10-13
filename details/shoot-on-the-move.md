---
icon: bullseye
description: Where to read about shooting while the robot is driving, in the WPILib docs and from FRC teams.
---

# Shoot on the move

Shooting on the move means scoring while the robot is still driving, instead of stopping to line up first. YAMS does not do the math for you, but the `SwerveDrive` gives you everything the math needs. This page explains the idea in a few lines and then points you at the WPILib documentation and at the write-ups, threads and code that FRC teams have already published.

## The idea in short

A game piece keeps the robot's velocity when it leaves the shooter. If the robot is moving sideways, the shot drifts sideways and misses. Almost every team solves this the same way:

1. Work out the shot as if the robot was standing still: the distance to the target gives a flywheel speed, a hood angle, and a **time of flight**.
2. Multiply the robot's field relative velocity by that time of flight. That is how far the shot will drift.
3. Move the target the opposite way by that much. This is the **virtual target**.
4. Aim at the virtual target and work the shot out again for its distance. Repeating this two or three times is usually enough, because the new distance changes the time of flight a little.

## What YAMS gives you

| You need | Where it comes from |
| -------- | ------------------- |
| Where the robot is on the field | `drive.getPose()` on the [Swerve Drive](swerve-drive.md) |
| How fast the robot is moving on the field | `drive.getFieldRelativeSpeed()` on the [Swerve Drive](swerve-drive.md) |
| Turning the robot to face a target while the driver keeps driving | [`withAim`](swerve-input-stream.md#aim-at-a-target) on the [Swerve Input Stream](swerve-input-stream.md). It takes a supplier, so it can follow a virtual target that changes every loop. |
| A flywheel that holds its speed | [Shooters](shooters.md) |
| A hood or a turret that goes to an angle | [Arms](arms.md) and [Turrets/Wrists](turrets-wrists.md) |

## WPILib documentation

WPILib does not have a page on shooting on the move. It does document every building block the technique is made of.

{% hint style="info" %}
These pages are written for WPILib 2026. The ideas are the same in 2027, but some classes were renamed. `ChassisSpeeds`, for example, is `ChassisVelocities` in WPILib 2027.
{% endhint %}

| Page | Why it matters here |
| ---- | ------------------- |
| [Coordinate System](https://docs.wpilib.org/en/stable/docs/software/basic-programming/coordinate-system.html) | Which way X, Y and a positive rotation point, on the robot and on the field. Get this wrong and the virtual target moves the wrong way. |
| [Translation, Rotation, and Pose](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/geometry/pose.html) | The `Translation2d`, `Rotation2d` and `Pose2d` classes that the distance and angle to the target are calculated with. |
| [Transformations](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/geometry/transformations.html) | Moving a pose by an offset, which is how you get from the center of the robot to where the shooter really is. |
| [Introduction to Kinematics and The ChassisSpeeds Class](https://docs.wpilib.org/en/stable/docs/software/kinematics-and-odometry/intro-and-chassis-speeds.html) | Robot relative and field relative velocity, and how to convert between them. |
| [Swerve Drive Kinematics](https://docs.wpilib.org/en/stable/docs/software/kinematics-and-odometry/swerve-drive-kinematics.html) | How the module states become the velocity of the whole robot. |
| [Pose Estimators](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/state-space/state-space-pose-estimators.html) | Combining odometry with vision. A moving shot is only as good as the pose it starts from. |
| [Tuning a Flywheel Velocity Controller](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/tuning-flywheel.html) | Getting the flywheel to reach its speed quickly and hold it. |
| [InterpolatingDoubleTreeMap](https://github.wpilib.org/allwpilib/docs/release/java/edu/wpi/first/math/interpolation/InterpolatingDoubleTreeMap.html) (API) | A lookup table that fills in the gaps between the distances you measured, for flywheel speed, hood angle and time of flight. |

## FRC write-ups and discussions

| Resource | What you get from it |
| -------- | -------------------- |
| [Shoot on the move from the code perspective](https://www.chiefdelphi.com/t/shoot-on-the-move-from-the-code-perspective/511815) (Chief Delphi) | The best place to start. It explains the virtual target, why you repeat the calculation, and links to most of the other resources on this page. |
| [Shooting on the fly, part 1](https://blog.eeshwark.com/robotblog/shooting-on-the-fly) and [part 2](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2) (Eeshwar Krishnan) | A blog series on the technique. Part 2 covers tuning it and using it on a real robot. |
| [Time of Flight Determination](https://www.chiefdelphi.com/t/time-of-flight-determination/512542) (Chief Delphi) | The different ways teams find the time of flight: measuring it, calculating it, or tuning it until shots land. |
| [Shoot While Move Code (1706)](https://www.chiefdelphi.com/t/shoot-while-move-code-1706/410494) (Chief Delphi) | Team 1706 walks through its 2022 code: a table of time of flight against distance, a virtual goal, and a few repeats of the calculation. |
| [Orbit 1690 Online Software Sessions](https://www.chiefdelphi.com/t/orbit-1690-online-software-sessions/467221) (Chief Delphi) | Recorded sessions from Team 1690 on their swerve and system control software, which other teams point to for the shoot on the move math. |

## Team code to read

| Code | What it shows |
| ---- | ------------- |
| [Mechanical Advantage 6328, `LaunchCalculator.java`](https://github.com/Mechanical-Advantage/RobotCode2026Public/blob/main/src/main/java/org/littletonrobotics/frc2026/subsystems/launcher/LaunchCalculator.java) | The launch calculation from their 2026 robot code. |
| [Hammerheads 5000, `TurretCalculator.java`](https://github.com/hammerheads5000/2026Rebuilt/blob/main/src/main/java/frc/robot/subsystems/turret/TurretCalculator.java) | The shot calculation for a turret from their 2026 robot code. Their [build thread post](https://www.chiefdelphi.com/t/frc-5000-hammerheads-2026-build-thread-open-alliance/507502/129) walks through it step by step. |

{% hint style="info" %}
None of this code uses YAMS, so the class and method names will be different from yours. Read it for the math, and use the table in [What YAMS gives you](#what-yams-gives-you) to find the YAMS call that supplies each input.
{% endhint %}
