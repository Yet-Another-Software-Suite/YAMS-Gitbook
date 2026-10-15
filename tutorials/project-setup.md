---
description: Make the robot project that every tutorial starts from.
---

# Create a project and install YAMS

Every tutorial starts from the same place: a WPILib command based project with YAMS installed. This page makes one.

Each tutorial page has a tab for **Java with Commands v2**, **Java with Commands v3** and **C++**. Pick the one you want now, because it decides which project template and which YAMS vendordep you need.

<figure><img src="../.gitbook/assets/tutorial-flavors.png" alt="A table comparing the three ways to follow the tutorials. Java with Commands v2 uses the Command v2 Robot template and the YAMS Commands v2 vendordep; the mechanism lives in a subsystem class that extends SubsystemBase, buttons are bound in RobotContainer.java, and the scheduler calls periodic. Java with Commands v3 uses the Command v3 Robot template and the YAMS Commands v3 vendordep; the mechanism lives in a class that implements Mechanism, buttons are bound in ExampleTeleop.java, and you call periodic yourself from Robot.robotPeriodic. C++ uses the Command v2 Robot template for C++ and the YAMS Commands v2 vendordep; the mechanism lives in a subsystem header and source file, buttons are bound in RobotContainer.cpp, and the scheduler calls periodic."><figcaption><p>The YAMS calls are the same. The project around them is what changes.</p></figcaption></figure>

{% hint style="success" %}
Not sure which to pick? Use **Java, Commands v2**. It is what most teams use today, and most of the other pages in these docs show it first.
{% endhint %}

## Create the project

Bring up the Visual Studio Code command palette with <kbd>Ctrl+Shift+P</kbd> and type "WPILib". Select **WPILib: Create a new project**.

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

This brings up the New Project Creator:

<figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

{% tabs %}
{% tab title="Java, Commands v2" %}
1. Click on **Select a project type (Example or Template)**.
2. Select **Template**, then **Java**, then **Command v2 Robot**.
{% endtab %}

{% tab title="Java, Commands v3" %}
1. Click on **Select a project type (Example or Template)**.
2. Select **Template**, then **Java**, then **Command v3 Robot**.
{% endtab %}

{% tab title="C++" %}
1. Click on **Select a project type (Example or Template)**.
2. Select **Template**, then **C++**, then **Command v2 Robot**.
{% endtab %}
{% endtabs %}

Then, whichever you picked:

3. Click on **Select a new project folder** and choose where to keep the project.
4. Fill in **Project Name**.
5. Enter your **Team Number**, so that you can deploy to your robot.
6. Check **Enable Desktop Support**, so that you can run the simulation.

Filled in, it looks something like this. The screenshots on this page show the Java Commands v2 choice.

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

You now have a command based robot project.

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

## Install YAMS

YAMS ships as two vendordeps, one for each WPILib command framework. Install only the one for your tab: the two conflict with each other.

Click on the **WPILib logo** in the left bar to open the vendor library list.

<figure><img src="../.gitbook/assets/RobotContainer.java - simple_robot - Visual Studio Code 9_2_2025 3_40_21 PM.png" alt=""><figcaption></figcaption></figure>

{% tabs %}
{% tab title="Java, Commands v2" %}
Scroll down to **Yet Another Mechanism System (Commands v2)** and click **Install**.

If it is not in the list, open the command palette with <kbd>Ctrl+Shift+P</kbd>, run **WPILib: Manage Vendor Libraries**, choose **Install new libraries (online)** and paste `https://cdn.yassrobotics.com/yams_commands2.json`.
{% endtab %}

{% tab title="Java, Commands v3" %}
Scroll down to **Yet Another Mechanism System (Commands v3)** and click **Install**.

If it is not in the list, open the command palette with <kbd>Ctrl+Shift+P</kbd>, run **WPILib: Manage Vendor Libraries**, choose **Install new libraries (online)** and paste `https://cdn.yassrobotics.com/yams_commands3.json`.
{% endtab %}

{% tab title="C++" %}
Scroll down to **Yet Another Mechanism System (Commands v2)** and click **Install**. The Commands v2 vendordep holds both the Java and the C++ library.

If it is not in the list, open the command palette with <kbd>Ctrl+Shift+P</kbd>, run **WPILib: Manage Vendor Libraries**, choose **Install new libraries (online)** and paste `https://cdn.yassrobotics.com/yams_commands2.json`.
{% endtab %}
{% endtabs %}

See [Installation](../README.md#installation) for the offline install.

## Install your motor controller's library

YAMS drives the vendor's own motor controller objects, so the vendor's library has to be installed too. Install it from the same list:

| Tutorial | Hardware | Vendor library |
| --- | --- | --- |
| [Arm](arm.md), [Elevator](elevator.md), [Shooter/FlyWheels](shooter-flywheels.md) | `SparkMax` | REVLib |
| [Double FlyWheel](double-flywheel.md) | `TalonFX` | Phoenix 6 |
| [Swerve Drive](swerve-drive.md) | `SparkMax`, `CANcoder`, `Pigeon2` | REVLib and Phoenix 6 |

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

## Where the tutorials put things

{% tabs %}
{% tab title="Java, Commands v2" %}
The template gives you `RobotContainer.java` and a `subsystems` folder. Each tutorial adds one subsystem class to `subsystems`, and then creates it and binds its buttons in `RobotContainer.java`.

A subsystem `extends SubsystemBase`, so the command scheduler calls its `periodic()` and `simulationPeriodic()` for you.
{% endtab %}

{% tab title="Java, Commands v3" %}
The template gives you `Robot.java`, `ExampleTeleop.java` and a `mechanisms` folder. Each tutorial adds one mechanism class to `mechanisms`, creates it in `Robot.java`, and binds its buttons in `ExampleTeleop.java`.

A Commands v3 mechanism `implements Mechanism`. Nothing calls a periodic method on it for you, so each tutorial gives the class a `periodic()` and a `simulationPeriodic()`, and `Robot.java` calls them:

* `Robot.robotPeriodic()` runs every loop, on the robot and in simulation. It calls each mechanism's `periodic()`, runs the scheduler, and then publishes the scheduler so that a dashboard shows which commands are running.
* `Robot.simulationPeriodic()` runs every loop in simulation only. It calls each mechanism's `simulationPeriodic()`.

With an arm and an elevator on the robot, `Robot.java` ends up with this:

```java
private final Scheduler scheduler = Scheduler.getDefault();

@Override
public void robotPeriodic() {
  arm.periodic();
  elevator.periodic();
  scheduler.run();
  Telemetry.log("Scheduler", scheduler, Scheduler.proto);
}

@Override
public void simulationPeriodic() {
  arm.simulationPeriodic();
  elevator.simulationPeriodic();
}
```

Forget the first and the mechanism publishes no telemetry and ignores live tuning. Forget the second and it never moves in simulation.

YAMS for Commands v3 lives in `yams.commands3` where the Commands v2 pages say `yams.commands2`. Everything in `yams.core` is shared.
{% endtab %}

{% tab title="C++" %}
The template gives you `RobotContainer.hpp`, `RobotContainer.cpp`, and a `subsystems` folder under both `src/main/include` and `src/main/cpp`. Each tutorial adds one subsystem as a header and a source file, and then creates it and binds its buttons in `RobotContainer`.

The C++ library uses Commands v2. Its methods have the same names as the Java ones with a capital first letter, such as `WithGearing` or `Run`. Where a name is different, the tutorial says so.
{% endtab %}
{% endtabs %}

## Pick a tutorial

* [Arm](arm.md)
* [Elevator](elevator.md)
* [Shooter/FlyWheels](shooter-flywheels.md)
* [Double FlyWheel](double-flywheel.md)
* [Swerve Drive](swerve-drive.md)
