---
description: >-
  Publish each mechanism's Pose3d, so AdvantageScope moves the matching part of
  your robot's CAD model as the real mechanism moves.
---

# How do I show my mechanisms on a 3D robot model?

AdvantageScope can show your robot's CAD model on the 3D field, with each mechanism moving as the real one does: the arm swinging, the elevator rising and the turret turning. Each moving part is a separate model, called a component, and your robot code publishes a `Pose3d` for each one. Every YAMS mechanism gives you that pose with `getPose3d()`.

{% hint style="info" %}
`getPose3d()` is in YAMS releases after v2026.10.15. It replaces `getRelativeMechanismPosition()`.
{% endhint %}

{% hint style="success" %}
Only want to see your mechanisms in 3D in simulation, without a CAD model? In simulation, every YAMS mechanism publishes a `Mechanism2d` under `Mechanisms/<name>/mechanism`, and AdvantageScope can [show a `Mechanism2d` on the 3D field](https://docs.advantagescope.org/tab-reference/3d-field#2d-mechanisms) with no setup. `getPose3d()` also works on the real robot.
{% endhint %}

AdvantageScope's video walks through setting up a custom robot model and its components:

{% embed url="https://www.youtube.com/watch?v=unX1PsPi0VA" %}
Configuring Custom Robot Models for AdvantageScope
{% endembed %}

[AdvantageScope's custom assets page](https://docs.advantagescope.org/more-features/custom-assets) has the full reference.

## 1. Tell YAMS where each mechanism is

Give each mechanism a `MechanismPositionConfig`: where it is mounted on the robot, and the plane it moves in. Positions are in meters from the robot's origin, which AdvantageScope puts at the center of the robot, **on the floor**: +X forward, +Y left and +Z up.

| Mechanism                  | Where it is mounted                                    | Its pose                                                        |
| -------------------------- | ------------------------------------------------------ | --------------------------------------------------------------- |
| `Arm`, `Pivot`             | The axis it turns about                                 | The axis, turned to its angle                                   |
| `Elevator`                 | The carriage at zero height                             | The carriage, moved by its height along the elevator's angle. `getCascadePose3d()` or `getContinuousPose3d()` for each stage: see [Elevator stages](#elevator-stages) |
| `FlyWheel`                 | Its axle                                                | The axle, turned by the wheel's position                        |
| `DoubleJointedArm`         | The lower arm's axis                                    | `getPose3d()` for the lower arm, `getUpperPose3d()` for the upper arm at the elbow |

A part turns about the axis normal to its movement plane:

* **XZ** (the default), such as an arm that swings up over the front of the robot. At 0° the part points forward (+X), and a positive angle turns it up.
* **YZ**, such as an arm that swings up over the side of the robot. At 0° the part points left (+Y), and a positive angle turns it up.
* **XY**, such as a turret. At 0° the part points forward (+X), and a positive angle turns it left. A `Pivot` is in the XY plane unless you give it a `MechanismPositionConfig`.

{% tabs %}
{% tab title="Java" %}
```java
import static org.wpilib.units.Units.Degrees;

import org.wpilib.math.geometry.Rotation3d;
import org.wpilib.math.geometry.Transform3d;
import org.wpilib.math.geometry.Translation3d;
import yams.core.mechanisms.config.MechanismPositionConfig;
import yams.core.mechanisms.config.MechanismPositionConfig.Plane;

// An elevator 0.1 m behind the robot's center, its carriage 0.15 m off the floor at zero height.
private final Elevator elevator = new Elevator(new ElevatorConfig()
    // ... the rest of the config
    .withMechanismPositionConfig(new MechanismPositionConfig()
        .withRelativePosition(new Translation3d(-0.1, 0, 0.15))), elevatorMotor);

// An arm on the elevator's carriage, its axis 0.2 m above the carriage's reference point. It moves
// with the carriage.
private final Arm arm = new Arm(new ArmConfig()
    // ... the rest of the config
    .withMechanismPositionConfig(new MechanismPositionConfig()
        .withParent(elevator::getPose3d)
        .withRelativePosition(new Translation3d(0, 0, 0.2))), armMotor);

// A turret 0.3 m off the floor, turned 180° so that its 0° faces the back of the robot.
private final Pivot turret = new Pivot(new PivotConfig()
    // ... the rest of the config
    .withMechanismPositionConfig(new MechanismPositionConfig()
        .withMovementPlane(Plane.XY)
        .withRelativePosition(new Transform3d(new Translation3d(0, 0, 0.3), new Rotation3d(0, 0, Math.PI)))), turretMotor);
```
{% endtab %}

{% tab title="C++" %}
```cpp
using yams::mechanisms::config::MechanismPositionConfig;

// An elevator 0.1 m behind the robot's center, its carriage 0.15 m off the floor at zero height.
m_elevatorConfig.WithMechanismPositionConfig(
    MechanismPositionConfig{}.WithRelativePosition(wpi::math::Translation3d{-0.1_m, 0_m, 0.15_m}));

// An arm on the elevator's carriage, its axis 0.2 m above the carriage's reference point.
m_armConfig.WithMechanismPositionConfig(
    MechanismPositionConfig{}
        .WithParent([this] { return m_elevator->GetPose3d(); })
        .WithRelativePosition(wpi::math::Translation3d{0_m, 0_m, 0.2_m}));

// A turret 0.3 m off the floor, turned 180 degrees so that its 0 degrees faces the back of the robot.
m_turretConfig.WithMechanismPositionConfig(
    MechanismPositionConfig{}
        .WithMovementPlane(MechanismPositionConfig::Plane::XY)
        .WithRelativePosition(wpi::math::Transform3d{wpi::math::Translation3d{0_m, 0_m, 0.3_m},
                                                     wpi::math::Rotation3d{0_rad, 0_rad, 180_deg}}));
```
{% endtab %}
{% endtabs %}

`withRelativePosition(Transform3d)` turns the mechanism where it is mounted, such as an arm on the side of the robot facing left. With `withParent(...)`, the relative position is from the parent's pose, such as the elevator's carriage, rather than from the robot.

## 2. Publish the poses

Publish every component's pose in one array, in the same order as the components in the model (`model_0.glb`, `model_1.glb`, ...). Publish it every loop.

{% tabs %}
{% tab title="Java" %}
```java
import org.wpilib.math.geometry.Pose3d;
import org.wpilib.telemetry.Telemetry;

// In periodic(), or in Robot.robotPeriodic() with Commands v3.
Telemetry.log("Robot/Components",
              new Pose3d[] {elevator.getPose3d(), arm.getPose3d(), turret.getPose3d()},
              Pose3d.struct);
```
{% endtab %}

{% tab title="C++" %}
```cpp
#include <array>
#include <span>
#include <wpi/telemetry/Telemetry.hpp>

// In Periodic().
std::array<wpi::math::Pose3d, 3> components{m_elevator->GetPose3d(), m_arm->GetPose3d(),
                                            m_turret->GetPose3d()};
wpi::telemetry::Log("Robot/Components", std::span<const wpi::math::Pose3d>{components});
```
{% endtab %}
{% endtabs %}

With AdvantageKit, record the same array with `Logger.recordOutput("Robot/Components", poses)`.

### Elevator stages

An elevator with moving stages has a component for each stage. Give its `ElevatorConfig` the stages with [`withStages`](../details/mechanism-configs/elevator-config.md#withstages): how many move, counting the carriage as the last, and how far each travels relative to the stage below it. The heights must add up to the elevator's maximum height, and without hard limits they set it. Then publish the pose of each stage, in order from the stage nearest the fixed stage to the carriage:

* `getCascadePose3d()` for a cascading elevator: every stage moves at once, each by its share of the carriage's height.
* `getContinuousPose3d()` for a continuous elevator: the carriage rises the whole of its travel first, then the stage below it, and so on.

{% tabs %}
{% tab title="Java" %}
```java
// Two stages that each travel 0.6 m, then a carriage that travels 0.5 m.
new ElevatorConfig()
    // ... the rest of the config
    .withStages(3, List.of(Meters.of(0.6), Meters.of(0.6), Meters.of(0.5)));

// The first stage, the second stage, then the carriage, which is elevator.getPose3d().
List<Pose3d> components = new ArrayList<>(elevator.getCascadePose3d());
components.add(arm.getPose3d());
Telemetry.log("Robot/Components", components.toArray(Pose3d[]::new), Pose3d.struct);
```
{% endtab %}

{% tab title="C++" %}
```cpp
// Two stages that each travel 0.6 m, then a carriage that travels 0.5 m.
m_elevatorConfig.WithStages(3, {0.6_m, 0.6_m, 0.5_m});

// The first stage, the second stage, then the carriage, which is m_elevator->GetPose3d().
std::vector<wpi::math::Pose3d> components = m_elevator->GetCascadePose3d();
components.push_back(m_arm->GetPose3d());
wpi::telemetry::Log("Robot/Components", std::span<const wpi::math::Pose3d>{components});
```
{% endtab %}
{% endtabs %}

Both need 2 or more stages: an elevator with only a carriage uses `getPose3d()`. Each stage is a component, zeroed like the carriage.

## 3. Export the model

Export the robot from CAD without the moving parts as `model.glb`, and each moving part as its own file: `model_0.glb`, `model_1.glb`, and so on, in the order you publish their poses. Export every part where it is when the robot is at its starting position. [Converting Onshape & STEP Files to glTF](https://docs.advantagescope.org/more-features/custom-assets/gltf-convert) explains how.

Put them in a folder named `Robot_<name>`, such as `Robot_Dozer`, with a `config.json`, in AdvantageScope's assets folder (`App`/`AdvantageScope` > `Show Assets Folder`). `App`/`AdvantageScope` > `Use Custom Assets Folder` can load them from your robot project instead, so they are kept with your code.

```json
{
  "name": "Dozer",
  "rotations": [],
  "position": [0, 0, 0],
  "cameras": [],
  "components": [
    { "zeroedRotations": [], "zeroedPosition": [0, 0, 0] },
    { "zeroedRotations": [], "zeroedPosition": [0, 0, 0] },
    { "zeroedRotations": [], "zeroedPosition": [0, 0, 0] }
  ]
}
```

## 4. Line each part up with its pose

AdvantageScope moves each component from where its `zeroedRotations` and `zeroedPosition` put it, by the pose you publish. Set them so that each part sits at the origin as YAMS's pose for it at zero expects:

| Component            | Zeroed at the origin                                                                          |
| -------------------- | --------------------------------------------------------------------------------------------- |
| An arm or pivot      | Its axis at the origin, pointing along +X at 0° (+Y in the YZ plane)                          |
| An elevator carriage | The carriage's reference point, the point you gave as its relative position, at the origin   |
| An elevator stage    | The stage as it sits with the elevator down, moved by the same offset as the carriage         |
| A flywheel           | Its axle at the origin                                                                        |
| A part on another    | As for its own kind: its relative position is from its parent, not the robot                  |

To set them:

1. Publish a zeroed robot pose, such as `Pose2d.ZERO`, select it as the robot's pose in AdvantageScope, and pick the "Axes" field, which shows the origin.
2. Adjust the model's `rotations`, then its `position`, until the robot is upright, facing +X, with its center on the origin and its wheels on the floor.
3. Publish an array of zeroed poses, one for each component (`new Pose3d[] {Pose3d.ZERO, Pose3d.ZERO, Pose3d.ZERO}`), and select it as the component poses.
4. For each component, adjust its `zeroedRotations`, then its `zeroedPosition`, until it sits at the origin as the table above says.
5. Publish the real poses, as in [Publish the poses](#id-2.-publish-the-poses). Each part now moves with its mechanism.

## Common pitfalls

### The mechanism's 0° isn't where YAMS expects it

An arm's pose is turned by the arm's angle, from 0° pointing forward. If your arm reads 0° straight down, it is drawn 90° off. Set the arm's angle so 0° is level, as YAMS's [arm feedforward expects](../details/arms.md), rather than turning the model to make up for it.

### The model is measured from the bellypan

AdvantageScope's robot origin is on the floor. Measure each mechanism's relative position from the floor, and line the robot model up with the floor in step 2.

### The poses are in a different order from the components

AdvantageScope gives the first pose to `model_0.glb`, the second to `model_1.glb`, and so on. A part that moves with another mechanism's pose is in the wrong place in the array.

### A turret swings up instead of turning

A `Pivot` turns in the XY plane, like a turret, unless you give it your own `MechanismPositionConfig`. A new `MechanismPositionConfig` is in the XZ plane, so set `withMovementPlane(Plane.XY)` on the one you give a turret.

### A part on another mechanism doesn't move with it

A part mounted on another, such as an arm on an elevator's carriage, needs `withParent(...)`. Without it, the arm's relative position is from the robot, and it stays put as the carriage rises.

### The parts are exported away from their starting position

Export every part where it is at the robot's starting position. A part exported partway through its travel is drawn that far off.
