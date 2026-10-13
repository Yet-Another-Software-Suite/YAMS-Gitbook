---
description: Lead the target by the time of flight, and repeat the calculation until the time stops changing.
---

# Strategy 2: Time of flight recursion

This is the strategy most teams mean by "shoot on the move". It leads the shot by aiming at a virtual target, and it finds that target by guessing, checking and adjusting. It is WPILib's [Dynamic Shooting via Time-of-Flight Recursion](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/dynamic-shooting.html), and [Shoot on the move from the code perspective](https://www.chiefdelphi.com/t/shoot-on-the-move-from-the-code-perspective/511815) on Chief Delphi describes the same thing.

{% hint style="info" %}
The WPILib pages on fire control are a work in progress in [wpilib-docs #3242](https://github.com/wpilibsuite/wpilib-docs/pull/3242). They have not been merged, so the links go to the pull request's preview and may move.
{% endhint %}

## How it works

1. Start with the standing shot: the distance to the real target gives a time of flight.
2. Multiply the robot's field relative velocity by that time. That is how far the shot will drift.
3. Move the target the opposite way by that much. This is the virtual target.
4. The virtual target is at a different distance, so it has a different time of flight. Go back to step 2 with the new time.
5. Stop when the time stops changing. Two or three passes is usually enough.

<figure><img src="../../.gitbook/assets/howto-sotm-iterate.gif" alt="Animation of a robot driving right at 2 meters per second with a target 3 meters straight ahead. Standing still the distance is 3.00 meters and the time of flight is 0.70 seconds, which gives a drift of 1.40 meters. The virtual target moves 1.40 meters back. The next pass gives 3.31 meters, 0.75 seconds and 1.50 meters. The pass after gives 3.36 meters, 0.76 seconds and 1.52 meters, and the last pass barely changes it."><figcaption><p>Each pass uses the distance from the pass before. After three it has settled.</p></figcaption></figure>

Only the time of flight is repeated. The flywheel speed is read from the table once, at the end, for the distance to the final virtual target. That is why you measured the time of flight in [Strategy 1](lookup-table.md): with it in the table, the code never needs a physics model of the shot.

{% hint style="success" %}
**Read this too:** Eeshwar's blog series, [Shooting on the fly, part 1](https://blog.eeshwark.com/robotblog/shooting-on-the-fly) and [part 2](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2), is the best companion to these pages. Part 1 covers the vector math. Part 2 is the practical side: building the lookup table, getting the shot's speed from the time of flight, what to do with a hood, and how to tune the latency.
{% endhint %}

## The calculator

Add this method to the `ShotCalculator` from [Strategy 1](lookup-table.md). WPILib's [Translation, Rotation, and Pose](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/geometry/pose.html) covers the geometry classes it uses, and [Introduction to Kinematics and The ChassisSpeeds Class](https://docs.wpilib.org/en/stable/docs/software/kinematics-and-odometry/intro-and-chassis-speeds.html) covers the velocity.

{% code title="ShotCalculator.java" %}
```java
/** Strategy 2: lead the target by the time of flight, and repeat until the time settles. */
public static Shot recursive(Pose2d robotPose, ChassisVelocities fieldVelocity) {
  Translation2d virtualTarget = TARGET;
  // Each pass uses the distance from the pass before, so the answer settles after a few.
  for (int i = 0; i < 3; i++) {
    double distance = robotPose.getTranslation().getDistance(virtualTarget);
    double timeOfFlight = TIME_OF_FLIGHT.get(distance);
    // The shot drifts by the robot's velocity times the time of flight, so aim that far the other way.
    virtualTarget = TARGET.minus(
        new Translation2d(fieldVelocity.vx * timeOfFlight, fieldVelocity.vy * timeOfFlight));
  }
  return shotAt(robotPose, virtualTarget);
}
```
{% endcode %}

**For your robot:** three passes suits most robots. If you drive fast straight at the target or straight away from it, try five, or use [Strategy 3](newtons-method.md).

## Use it

Follow steps 4 and 5 of [How do I shoot on the move?](../how-do-i-shoot-on-the-move.md), with this as `shot()`. It is the one that page already shows.

{% tabs %}
{% tab title="Java, Commands v2" %}
{% code title="RobotContainer.java" %}
```java
private ShotCalculator.Shot shot() {
  return ShotCalculator.recursive(drive.getPose(), drive.getFieldVelocity());
}
```
{% endcode %}
{% endtab %}

{% tab title="Java, Commands v3" %}
{% code title="Robot.java" %}
```java
ShotCalculator.Shot shot() {
  return ShotCalculator.recursive(drive.getPose(), drive.getFieldVelocity());
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

`withAim` now faces the virtual target, and the shooter follows the speed for its distance. Keep `withAimLookahead` short: it covers the delay before the game piece leaves, and the calculator covers the time of flight.

## Where it struggles

* **Driving straight at the target, or straight away from it.** The time of flight is most sensitive to the robot's speed in these directions, and the recursion takes the most passes to settle. Driving past at a diagonal is the most stable approach. The simulation on WPILib's [Dynamic Shooting via Time-of-Flight Recursion](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/dynamic-shooting.html) page lets you see this for any velocity.
* **Driving fast.** A faster robot turns a small error in the time of flight into a larger miss, and its measured velocity is less certain too. Slowing down to shoot is a legitimate strategy.
* **Driving faster than the shot.** If the robot closes on the target faster than the game piece travels, there is no sensible answer.
* **A shot the shooter cannot make.** If the virtual target is beyond the far end of your table, the table just returns its last entry and the shot falls short. Check the distance against the range you measured, and slow the robot down or hold fire when it is outside. [part 2](https://blog.eeshwark.com/robotblog/shooting-on-the-fly-pt2) of Eeshwar's series discusses the options.
* **Air drag at high speed.** The table already includes the drag on a standing shot. It does not include the extra drag from the robot's own motion. WPILib's [Linear Drag (First-Order Air Friction)](https://frc-docs--3242.org.readthedocs.build/en/3242/docs/software/advanced-controls/fire-control/linear-drag.html) page gives a one line correction: use `(1 - Math.exp(-k * timeOfFlight)) / k` in place of `timeOfFlight` when you move the target, where `k` is a drag constant you tune.

If the recursion is slow to settle for the way your robot drives, [Strategy 3: Newton's method](newtons-method.md) finds the same answer in fewer passes.
