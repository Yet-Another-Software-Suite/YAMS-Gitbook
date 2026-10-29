# Telemetry docs: overall structure

Scope: the telemetry docs in YAMS-Gitbook. Today these are `understanding/telemetry.md`, `details/datalog-best-practices.md` and `how-to/how-do-i-view-my-datalog-in-advantagescope.md`. Live tuning (`details/live-tuning/`) and the SwerveInputStream page stay as they are and are linked, not rewritten.

## Audiences

**Primary:** FRC student programmers
They know Java and command-based. They want a mechanism moving in simulation and on the robot, tuned easily while they learn control theory.

**Secondary:**
- Mentors and lead programmers: they choose and adopt YAMS for a team, and care about vendor coverage, design decisions and upgrading from 2026.

## Primary use case

Graph an arm's setpoint against its position while tuning it. The reader turns on the arm's telemetry, finds it in NetworkTables, reads the Mechanism2d, and plots `closedloop/setpoint/position` against `mechanism/position` in AdvantageScope. The gap between the two lines is what tuning closes, so the page hands off to Live tuning.

## Getting Started: "Watching your arm follow its setpoint"

New page under Tutorials, built on the arm tutorial's code so the reader has nothing new to set up.

1. **Turning on the arm's telemetry**
   `withTelemetry(name, TelemetryVerbosity.HIGH)` on the motor config and the arm config. Covers: verbosity in one sentence (HIGH while tuning), and that nothing else is needed each loop.
2. **Finding your arm in NetworkTables**
   `NT:/Mechanisms/<Arm>/<Motor>/...` for what it is doing, `NT:/Tuning/...` for what you can change. Covers: the two tables, how names map to paths.
3. **Reading the Mechanism2d**
   The arm drawn live, the thinner setpoint line beside it, and the limit colors. Line colors are not fixed in the text: the arm is drawn in its `withSimColor` color. Covers: Mechanism2d, simulation and robot showing the same data.
4. **Graphing setpoint against position**
   AdvantageScope live, both keys on one Line Graph, shown in degrees. Covers: units carried on the topics, unit conversion, what the gap between the lines means.
5. **Closing the gap with live tuning**
   One change to kP, watch the graph respond, then go to Live tuning. Covers: the hand-off to `details/live-tuning/`.

## Diving Deeper topics

| Topic | Intent | Covers | Today's source |
| ----- | ------ | ------ | -------------- |
| Choosing what each motor publishes | Publish only the fields you need, by verbosity or one at a time | Verbosity table, `SmartMotorControllerTelemetryConfig`, `with*()` fields, `withCustom`, Java/C++ names | `understanding/telemetry.md` |
| Recording a match to a DataLog | Keep a record of every match and keep the SystemCore's storage under control | `withDataLogName`, `withoutNetworkTables`, NT off when the FMS is attached, `DataLogManager`, where files land, choosing what to log | `details/datalog-best-practices.md` (merged) |
| Reviewing a recorded match in AdvantageScope | Open a `.wpilog` and find what happened, without the robot | Open file vs connect live, field tree from DataLog prefixes, Line Graph, Swerve and 2D Field tabs, not Log Replay | `how-to/how-do-i-view-my-datalog-in-advantagescope.md` (kept as a how-to, trimmed) |
| Watching a swerve drive | Publish poses, chassis velocities and module states that AdvantageScope draws | `SwerveDriveTelemetryConfig`, `SwerveModuleTelemetryConfig`, structs, what differs from a motor, motors configured separately, link to SwerveInputStream telemetry | `understanding/telemetry.md`, `details/datalog-best-practices.md` |
| How YAMS publishes telemetry *(optional, mentors)* | Explain the design: field enums, one field to NT and DataLog, units on the topic, dashboard edits not overwritten by code | `DoubleTelemetry`, `BooleanTelemetry`, `StringTelemetry`, `StructTelemetry`, why not WPILib `Telemetry`/`Tunables`, the `/Mechanisms` and `/Tuning` backends | `understanding/telemetry.md` details box, plus new |

## Reference

| Item | Where |
| ---- | ----- |
| Telemetry keys and units: motor (`mechanism/*`, `rotor/*`, `measurement/*`, `externalencoder/*`, `closedloop/*`, `current/*`, `motor/*`), LQR, linear, mechanism `loopTime` | A "Telemetry keys and units" page in YAMS-Gitbook, from today's unit tables, completed against the field enums |
| `SmartMotorControllerTelemetryConfig`, `SwerveDriveTelemetryConfig`, `SwerveModuleTelemetryConfig` methods | YAMS-Gitbook-API (exists; linked, not duplicated) |

## Cut or moved

- **"Simulation vs reality":** folded into Getting Started step 3.
- **"Exporting data with units" and the long AdvantageScope conversion how-to:** cut to what step 4 needs, with a link to AdvantageScope's docs.
- **The DataLog and swerve rules repeated across three pages:** each lives in one Diving Deeper topic, and the others link to it.
