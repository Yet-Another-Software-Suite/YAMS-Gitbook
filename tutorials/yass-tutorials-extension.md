---
description: Follow the YAMS and YAGSL tutorials beside your code in WPILib VS Code.
icon: book-open
---

# Install the YASS Tutorials extension

The YASS Tutorials extension brings these tutorials into WPILib VS Code. It adds a **YASS** tab that lists every YAMS and YAGSL tutorial step by step, and opens each step beside your code. The tutorials are the same as on this site, so the extension is optional.

{% stepper %}
{% step %}
### Download the extension

Download [yass-tutorials.vsix](https://github.com/Yet-Another-Software-Suite/yass-tutorials-vscode/releases/download/latest/yass-tutorials.vsix).
{% endstep %}

{% step %}
### Install it in WPILib VS Code

1. Click **Extensions** in the activity bar on the left.
2. Click **...** at the top of the Extensions view.
3. Click **Install from VSIX...**, pick the `yass-tutorials.vsix` you downloaded and click **Open**.

**YASS Tutorials** appears in the list of installed extensions, and a book icon appears in the activity bar.

<figure><img src="../.gitbook/assets/yass-tutorials-install.gif" alt="In WPILib VS Code, the user clicks Extensions in the activity bar, then the ... button at the top of the Extensions view, then Install from VSIX. In the file picker they pick yass-tutorials.vsix in Downloads and click Open. YASS Tutorials appears in the installed extensions, a book icon appears in the activity bar, and a message says Completed installing extension."><figcaption><p>Installing yass-tutorials.vsix from the Extensions view.</p></figcaption></figure>

{% hint style="info" %}
Install it in WPILib VS Code, the one the WPILib installer adds. The extension uses WPILib's extension to create robot projects and install vendordeps.
{% endhint %}
{% endstep %}

{% step %}
### Open a tutorial

Click the book icon. **Your Robot Project** shows whether your project has YAMS and YAGSL; click either one to install it. Under **Tutorials**, click the arrow beside a tutorial to see its steps, then click a step to open it beside your code.

<figure><img src="../.gitbook/assets/yass-tutorials-open-step.gif" alt="With RobotContainer.java open, the user clicks the book icon. The YASS tab shows Your Robot Project, with MyRobot, Java, Commands v2, YAMS and YAGSL all checked, and the YAMS and YAGSL tutorials below. The user clicks the arrow beside Arm, which lists its six steps, and clicks Step 1: The motor controller. The step opens to the right of RobotContainer.java."><figcaption><p>Opening the Arm tutorial's first step beside the code.</p></figcaption></figure>
{% endstep %}
{% endstepper %}

## What it adds

* **Copy**, **Insert** and **Create file** on every code sample. **Insert** puts the code at the cursor in the file you were last editing.
* One code language for every sample: **Java, Commands v2**, **Java, Commands v3** or **C++**. By default it matches your project.
* A check mark for each step you finish, and a search across every tutorial.
* A button that opens WPILib's vendor library list when a tutorial needs a library your project doesn't have, such as REVLib for a `SparkMax`.

## Updates

The tutorials update themselves: the extension checks for new ones every day and keeps the ones it has for when you are offline. Click **Refresh** at the top of **Tutorials** to check right away.

The extension itself doesn't update on its own. To get new features, download `yass-tutorials.vsix` again and install it the same way.
