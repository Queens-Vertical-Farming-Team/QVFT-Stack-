# QVFT Stack — Database & Automation Implementation Roadmap

> **Status:** Engineering assignment and implementation guide, not a claim that the software has already been built or deployed.
>
> **Audience:** Stack members learning PostgreSQL/Supabase, Python, automation and control; Stack lead and PR reviewers.
>
> **Scope:** Local/synthetic development first. Physical actuator deployment requires independent hardware, Mechanical and Plant Science safety review.

## Read these guides

- [Database & Supabase — step-by-step guide](database-implementation-guide.md) — GitHub issues **#13, #21, #23, #22**.
- [Automation & Control — step-by-step guide](automation-control-implementation-guide.md) — GitHub issues **#17, #25, #24, #26**.
- [Existing system architecture](architecture.md) — higher-level planned Stack architecture.
- [Database epic #3](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/3).
- [Automation epic #7](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/7).

These instructions implement and expand the existing GitHub issue descriptions. **Open each issue before starting**: its acceptance criteria and dependencies are authoritative for that PR.

## What are we building?

The intended demonstrable milestone is:

~~~text
synthetic sensor reading
  -> validated telemetry in FastAPI
  -> database reading/history
  -> configurable automation rule
  -> safety/state-machine validation
  -> traceable actuator command
  -> simulated ESP32/mock actuator
  -> acknowledgement and reported state
  -> Supabase command/event history
  -> dashboard can retrieve results
~~~

For actual hardware, local controller firmware enforces actuator-specific safety independently of the internet/database. Supabase is a historical/configuration system, **not** a real-time safety controller. A database write proves a command was requested, not that a physical actuator changed state.

## Recommended task order

| Order | Issue | Assignment | Depends on | Proof of completion |
| --- | --- | --- | --- | --- |
| 1 | [#13](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/13) | Initial Supabase schema | None | Recreate database from migrations, insert/query telemetry and alerts |
| 1 (parallel) | [#17](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/17) | Safe actuator command contract | None | Mock device accepts valid commands and rejects invalid/expired ones |
| 2 | [#21](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/21) | Supabase access security/RLS | #13 | Unauthorized operations fail; privileges documented |
| 2 | [#23](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/23) | Telemetry performance/data quality | #13 | Duplicate, invalid and stale data handled; indexes tested |
| 2 (parallel) | [#25](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/25) | State machine, overrides and interlocks | #17 | Tests for all modes and >=5 interlocks |
| 3 | [#22](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/22) | Actuator/automation SQL tables | #13 and #17 | Trace a rule event through command to device state |
| 3 | [#24](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/24) | Deterministic rule engine | #17 and initial data model; integrate #25 | Three tested rules, hysteresis, debounce, manual/safety checks |
| 4 | [#26](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/26) | End-to-end simulation | #22, #24 and #25 | Successful and failure workflows, persistent audit trail |

**Dependencies with other team members:** Coordinate with [#11 telemetry format](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/11), [#12 FastAPI bootstrap](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/12), [#16 environmental alerts](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/16), and [#18 integration testing](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/18).

## Contributor setup (Windows PowerShell example)

Prerequisites: Git, VS Code, Python, Node.js/npm, and Docker Desktop with a functioning container engine. Make sure that required tools are available before installing anything globally.

~~~powershell
git clone https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-.git
cd QVFT-Stack-
code .
npm install --save-dev supabase
npx supabase init
npx supabase start
~~~

If the repo already has initialized Supabase configuration, **do not reinitialize/overwrite it**. Install dependencies and run the existing setup instead. Use a feature branch per issue, e.g.:

~~~powershell
git switch main
git pull
git switch -c feature/issue-13-database-schema
~~~

When testing database migrations:

~~~powershell
npx supabase migration new initial_telemetry
npx supabase db reset
~~~

**Danger:** The default reset acts on local development data and destroys it. Do not use flags such as --linked, --remote or --db-url against a shared/production database for these exercises. Confirm the environment before running commands.

Supabase official references:
- https://supabase.com/docs/guides/local-development
- https://supabase.com/docs/guides/local-development/database-migrations
- https://supabase.com/docs/guides/database/postgres/row-level-security

## Responsibility boundaries

| Component | Owns |
| --- | --- |
| ESP32 / Arduino firmware | Sensor reads, watchdog, physical actuator I/O, local interlocks/fail-safe |
| FastAPI backend | Device authentication, telemetry validation, command authorization, communication |
| Supabase / PostgreSQL | Data model, readings, configuration, command/event history, access policies |
| Python automation service | Validated farm-state evaluation, rules, creation of proposed commands |
| Dashboard | Monitoring and authenticated operator requests; never privileged credentials |

The team should document ownership of each API and schema contract before implementing across workstreams. Keep initial control deterministic and simulated rather than attempting PID/ML/chemical dosing immediately.

## Cross-team alignment for issues #16 and #18

**#16 — First rule-based farm alerts**
1. Ask Plant Science to identify candidate environmental variables and provisional test ranges, marked unvalidated.
2. Define each alert's condition, severity, debounce duration, associated sensor-quality conditions, and operator recommendation.
3. Test normal, threshold, stale-sensor and device-offline cases.
4. Store/query alert history without automatically treating every alert as permission to actuate.

**#18 — Integration test plan**
1. Specify a synthetic sensor payload with device, sensor, zone, time, value and units.
2. Test a normal sensor -> backend -> database -> dashboard read path.
3. Test invalid payload, disconnected device, database unavailable and repeated payload.
4. For automation, also test manual override, invalid/outdated commands, lost acknowledgement and safe mode.
5. Record expected HTTP responses, database outcomes, logs and recovery actions.

## PR workflow / lead checklist

1. Claim the appropriate GitHub issue; communicate with any dependent workstream owner.
2. Open a dedicated branch (not main); make a focused change.
3. Add implementation, tests, synthetic fixtures and documentation. Never commit .env, secret keys, production readings or passwords.
4. Run tests; include the exact reproduction steps in the PR description.
5. Link the issue in the PR (e.g. "Relates to #13") and request another Stack member's review.
6. Lead checks: reproducibility from clone; expected and adverse tests; error messages; missing-data handling; authorization; secret scan; rollback plan.
7. Merge only after review and tests pass.

## Definition of done for the integrated prototype

- [ ] A second member can set up the project from a clean checkout.
- [ ] Synthetic telemetry can be stored and queried.
- [ ] Automation rules generate proposed commands only from fresh, valid data.
- [ ] Modes, expiries, manual overrides and interlocks reject unsafe/conflicting commands.
- [ ] Mock controller acknowledges or rejects commands explicitly.
- [ ] Command request, acknowledgement and reported actuator state are separate and traceable.
- [ ] Offline, timeout, stale data and reconnect scenarios are tested.
- [ ] Supabase access is restricted appropriately; no privileged credentials appear in the browser/firmware/repo.
- [ ] No physical pump, dosing equipment, fan or mains-powered load is activated by this milestone.

**Suggested first demonstration:** simulate a fan with an illustrative 28°C ON / 26°C OFF hysteresis band, 60-second debounce, a manual override, and a deliberately expired command. These numbers are *demo values*, not QVFT crop or hardware operating limits.
