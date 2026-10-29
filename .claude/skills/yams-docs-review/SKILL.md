---
name: yams-docs-review
description: Review or write YAMS-Gitbook pages so a new FRC programmer can learn YAMS from them. Use when adding, editing or reviewing any page in YAMS-Gitbook, when restructuring SUMMARY.md, or when someone says the docs are hard to learn from. Checks the learning path, page type, length, examples and YAMS house style, and reports where a beginner gets stuck.
---

# YAMS docs review

The docs exist so that a student who has never used YAMS can get a mechanism moving, tune it, and run it on a robot. Accuracy matters, but a page that is accurate and still leaves the reader stuck has failed. Review every page as that student.

## The reader

Assume the reader:

* Knows basic Java (classes, fields, methods) or basic C++.
* Has used WPILib a little, but may not know command-based programming well.
* Does not know PID, feedforward or motion profiles beyond the names.
* Has never used YAMS, YAMG or the YAMS Live Tuner.
* Is often reading on a phone at a build meeting, with a robot that does not move yet.

If a page needs more than this, it says so at the top and links to where to learn it (WPILib docs for command-based and PID basics).

## Step 1: Place the page in the learning path

Read `SUMMARY.md` first. Decide which type the page is, and check that it sits in the matching section:

| Type | Answers | Belongs in | Must not contain |
| --- | --- | --- | --- |
| Tutorial | "Teach me, start to finish" | Tutorials | Options the reader does not need yet, tables of every method |
| How-to | "How do I do X on my robot?" | How-to | Background theory beyond a sentence or two, with a link for more |
| Concept | "Why does it work this way?" | Understanding | Step-by-step instructions, full option lists |
| Reference | "What does this option do?" | Reference (one page per config class) | Tutorials, opinions, long stories |

Flag any page that mixes two types. A multi-step walkthrough (such as the AdvantageKit steps) is a tutorial, wherever it currently sits.

## Step 2: Walk the page as the reader

Go through the page top to bottom and ask at each section:

1. **Do I know every word here yet?** Flag terms used before they are explained or linked: `SmartMotorController`, mechanism, setpoint, closed loop, `kG`, stator current, soft limit, verbosity, op mode, `Notifier`.
2. **Can I tell what I should see now?** Every tutorial step ends with something the reader can check: it builds, a value shows on the dashboard, the mechanism moves. A step that only adds code with nothing to check is a stuck point.
3. **What do I do if it did not work?** Each tutorial step names the one or two most likely failures and how to fix them (wrong CAN ID, inverted motor, mechanism not in `simulationInit`, wrong op mode, forgot the vendor library).
4. **Where do I go next?** GitBook already shows Previous and Next buttons at the end of every page, in table of contents order, so tutorial pages must not add a "Next" section that links the following step. Flag one if you find it. Only the last page of a tutorial, or a page whose next step is somewhere other than the following page in the table of contents, should point the reader on, under a heading that says what it is, such as "Read more" or the step's name.

## Step 3: Check the whole path, not just the page

When reviewing a tutorial or the table of contents, check the reader can get through this whole sequence without leaving the tutorials:

1. Pick a flavor and create the project.
2. Understand the three layers in a sentence each: the motor controller (`SmartMotorController` and its config), the mechanism (`Arm`, `Elevator`, `Pivot`, `FlyWheel`), and the subsystem or `Mechanism` that turns it into commands.
3. See something move in simulation, as early as possible.
4. Tune it: find `kG` or `kS`, then `kP`, then the profile, with the numbers to try and what the graph should look like.
5. Run it on the real robot safely: CAN IDs, direction check at low voltage, starting position, soft limits, first enable.
6. Fix it when it misbehaves: a troubleshooting page for the common failures.

Any missing step is the most important finding in the review.

## Step 4: Size and repetition

* A tutorial step should be readable in one sitting: aim for under about 1,200 words in one tab.
* A concept page explains one idea. Over about 2,500 words, it is probably several pages.
* A reference page can be long, but every option gets a heading, a one line summary first, then the detail.
* When two tutorials repeat the same steps (arm and elevator), the second should say what is the same, link back, and spend its words on what differs (gravity, linear units, carriage mass).
* Flag two pages that explain the same confusion in different ways (for example `run()` vs `runTo()` and setpoint methods). Merge them, or give each a clear different job.

## Step 5: Examples

* Every code block compiles as shown, or says clearly that it is a fragment and where it goes.
* Every tutorial has a complete file at the end, with imports.
* Java, Commands v3 and C++ tabs agree with the Commands v2 tab, apart from the documented differences.
* Placeholder gains are called out as placeholders, with what the reader should expect to see with them.

## Step 6: YAMS house style

These apply to every page:

* No em dashes. Use a comma, colon, parentheses or a new sentence.
* Example robot code uses the `first.robot` package with `first/Main.java`, never `frc.robot`.
* Say SystemCore, never RIO or roboRIO.
* Intakes, rollers, feeders and indexers use the `FlyWheel` mechanism, even open loop.
* At most one YAMS mechanism per subsystem, so live tuning works.
* Swerve examples use `drive.getField2d()`, never a new `Field2d`.
* Commands v3 code uses `while (true)` with `yield` coroutines, never `runRepeatedly` or Commands v2 decorator chains.
* Java signatures take WPILib `Measure` types, never `double` for physical quantities. `Telemetry.log` takes a `Measure` directly.
* Images have alt text that describes what the reader should notice.
* Link text names the destination ("Live tuning"), never "here".

## Report

Report findings as a table, worst first:

| Severity | Page | Where | Stuck point | Fix |
| --- | --- | --- | --- | --- |

Severity:

* **Blocker:** the reader cannot continue (a missing step in the path, code that does not compile, a step with no way to check it worked).
* **Major:** the reader continues but is confused or learns the wrong thing (an unexplained term, a page in the wrong section, a mixed-type page).
* **Minor:** style, length or house-style issues.

End with the single change that would help the most readers, if only one gets made.

When asked to fix rather than review, make the Blocker and Major fixes, keep the page's existing tabs and structure, and follow the house style above.
