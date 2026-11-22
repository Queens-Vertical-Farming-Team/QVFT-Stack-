# QVFT Stack — Automation & Control Step-by-Step Guide

**Companion:** [Implementation roadmap](implementation-roadmap.md) | [Supabase guide](database-implementation-guide.md)

**Scope:** Assignments [#17](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/17), [#25](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/25), [#24](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/24), [#26](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/26). These are implementation **instructions**, not claims of completed or deployed features.

> **Physical safety boundary:** Begin entirely with synthetic readings and mock devices. Never activate a real pump, fan, grow light, relay, dosing system or mains-powered load as part of these issues without separate electrical/mechanical, plant-science and farm safety reviews. The local ESP32/controller must enforce critical safety even if the API, Wi-Fi or database fails.

## What is automation versus control?

- **Automation** chooses which action should be requested when a condition or schedule is met (e.g. ask a fan to turn ON when a temperature condition persists).
- **Control** uses measurements of a system to regulate behavior and maintain a desired operating state. A thermostat with hysteresis is a simple feedback controller; a variable-speed fan might later support more continuous control.
- **State machine** defines the allowed operating modes and transitions, including safety and manual override behavior.
- **Interlock** is a condition that prevents an unsafe actuator action.
- **Command lifecycle** distinguishes request, delivery, acknowledgement, execution, completion and failures.
- **Supabase** stores configuration, telemetry, history and audit records; it is not a hard-real-time actuation/safety layer.

Target first milestone:

~~~text
valid synthetic temperature
  -> configurable rule evaluation
  -> proposed FAN_ON command
  -> mode/interlock/expiry checks
  -> mock controller
  -> acknowledgement + simulated reported state
  -> command/event records in Supabase
~~~

### Recommended code organization

Coordinate with the onboarding PR and existing backend owner; these are **proposed files** and may need adaptation.

~~~text
backend/
  automation/
    commands.py         # models, validation and command lifecycle
    state_machine.py    # OFF/MANUAL/AUTO/SAFE transitions
    safety.py           # interlocks
    rules.py            # rule evaluator
    simulator.py        # fake farm + mock device
  tests/
    test_commands.py
    test_state_machine.py
    test_rules.py
    test_simulator.py
docs/
  automation-command-contract.md
  automation-safety.md
  automation-demo.md
~~~

Python prerequisites: 3.x virtual environment, pytest, and whichever validation library the backend team chooses (for example Pydantic). Don't add unnecessary dependencies without coordinating with the backend owner.

~~~powershell
# From repo root on Windows
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install pytest
python -m pytest
~~~

Ensure .venv/ and .env are ignored by Git; do not modify backend package management without coordinating on existing requirements.

---

## #17 — Safe actuator command format

### What it means

A request such as "turn on the fan" is ambiguous unless it names the actuator, who asked, when the command was created, when it expires and how success is confirmed. This issue defines a stable contract used by FastAPI, the mock controller, later ESP32 firmware and Supabase.

### Steps to implement

1. **Inventory supported actuator actions:** FAN (ON/OFF/SET_SPEED), PUMP (START/STOP), LIGHT (ON/OFF/SET_LEVEL), VALVE (OPEN/CLOSE), RELAY (ON/OFF). Mark unsupported actions for each device, including value ranges and units.
2. **Define command JSON** and validation rules for required fields, types and allowed status values.
3. **Give every command a unique ID**; define how devices handle duplicate IDs without re-executing a previously applied command.
4. **Define time semantics:** timezone-aware UTC timestamps; reject expired commands at the executing device; account for synchronization/clock reliability and an acceptable clock-skew policy.
5. **Define the lifecycle**: pending -> sent -> acknowledged -> executing -> completed, with rejected/failed/expired/cancelled terminal outcomes. Document which component writes each status and ensure status transitions are validated.
6. **Define operating modes** OFF, MANUAL, AUTO and SAFE, including permissions for each command source.
7. **Define a device acknowledgement**: command_id, device_id, actuator_id, status/result, reported_at, optional reason/error code and observed state when available.
8. **Document timeout/retry behavior:** no replay of expired commands after reconnect; retries are idempotent; lack of ACK is not equivalent to successful actuation.
9. **Create a minimal Python mock controller** that validates one command, returns an ACK, and maintains a simulated actuator state.
10. **Write positive and failure tests**, then document command examples and open a PR.

Illustrative request (not a live control instruction):

~~~json
{
  "command_id": "cmd-0001",
  "actuator_id": "fan-zone-01",
  "action": "ON",
  "value": null,
  "source": "automation",
  "requested_at": "2026-11-18T16:00:00Z",
  "expires_at": "2026-11-18T16:00:10Z",
  "priority": 2,
  "status": "pending"
}
~~~

Illustrative acknowledgement:

~~~json
{
  "command_id": "cmd-0001",
  "device_id": "qvft-zone-01",
  "actuator_id": "fan-zone-01",
  "status": "completed",
  "reported_state": "ON",
  "timestamp": "2026-11-18T16:00:02Z"
}
~~~

**Design warning:** Requested state and reported state are separate. Do not mark a command completed merely because a database row was inserted or the API returned 200. For real equipment, an electrical feedback signal may still be needed to establish actual operation.

**Test cases:** valid MANUAL command; valid AUTO command; invalid action; disabled actuator; expired command; wrong mode; offline device; duplicate ID; ACK updates status; missing ACK yields an unresolved/failed/timeout outcome according to documented policy.

**Deliverables:** command contract and ACK format under docs/, mock controller under backend/automation/, test output and a PR linked to #17.

---

## #25 — Actuator state machine, manual overrides and safety interlocks

### What it means

This is the logic that decides whether an otherwise well-formed command **may** execute. Mode transitions and safety checks must take precedence over ordinary automation rules.

### Define mode semantics first

| Mode | Intended behavior |
| --- | --- |
| OFF | Intentionally disabled; no ordinary activation commands |
| MANUAL | Authorized operator controls subsystem; normal AUTO rules cannot fight override |
| AUTO | Eligible automation commands may be considered, subject to interlocks |
| SAFE | Fault handling; predetermined validated safe behavior; ordinary commands blocked |

**Caution:** A safety-critical stop/interlock may need to apply even in MANUAL or OFF. A user override is not permission to bypass mechanical safety constraints. SAFE behavior for each physical actuator must be approved before hardware use.

### Steps to implement

1. **Draw a state transition table/diagram:** What enters SAFE? Who may exit? Is reset explicit? What happens to queued commands when leaving MANUAL? Document exact transitions.
2. **Create an operating mode Enum** and an actuator state object (mode, enabled, device_online, last_heartbeat, reported_value, fault flags).
3. **Write a command gate function** that checks authorization, command validity/expiry, actuator enable state, operating mode, device health and interlocks before creating a dispatchable command.
4. **Implement override precedence:** entering MANUAL suppresses/cancels conflicting pending AUTO commands; no automatic resume until a defined transition.
5. **Implement at least five test interlocks**, e.g. low reservoir prevents pump start; exceeded pump runtime blocks/requires shutdown; offline device blocks requests; stale sensor inhibits actions that depend on its value; disabled actuator blocks activation; conflicting operations cannot run together.
6. **Implement command expiry and replay defense** even when a device reconnects after the command was queued.
7. **Record every rejection** with a machine-readable reason code and human-readable explanation; never quietly discard faults.
8. **Demonstrate SAFE mode** with a mock actuator only, and show which state it enters after an injected fault.
9. **Run unit tests for every transition** and for each interlock.
10. **Write docs/automation-safety.md** listing each condition, blocked action, intended result, source of truth and test.

Pseudocode sketch (not complete production code):

~~~python
def can_dispatch(command, state, now):
    if now >= command.expires_at:
        return False, "EXPIRED"

    if not state.enabled:
        return False, "ACTUATOR_DISABLED"

    if not state.device_online:
        return False, "DEVICE_OFFLINE"

    if state.mode == "SAFE":
        return False, "SAFE_MODE"

    if state.mode == "OFF" and command.is_activation:
        return False, "OFF_MODE"

    if state.mode == "MANUAL" and command.source == "automation":
        return False, "MANUAL_OVERRIDE"

    if command.action == "PUMP_START" and state.reservoir_low:
        return False, "LOW_RESERVOIR"

    return True, "ALLOWED"
~~~

Extend this with real type validation, authentication, mode transitions, source precedence, fault recovery, relevant sensor freshness, five or more tested interlocks, and device-local enforcement.

### Minimum required test table

| Situation | Expected simulation outcome |
| --- | --- |
| Valid AUTO command, normal state | Accepted |
| MANUAL mode, conflicting AUTO command | Rejected |
| Expired command | Rejected, including after reconnect |
| Actuator disabled | Activation rejected |
| Reservoir low + pump START | Rejected |
| Sensor stale + sensor-dependent request | Rejected |
| Device offline | Not dispatched; appropriately failed/expired |
| Fault forces SAFE | Ordinary commands blocked and safe behavior logged |
| Command duplicate | No unintended repeated actuation |
| Recovery from SAFE | Requires explicit permitted transition |

**Deliverables:** state machine, transition diagram, interlock table, tests, rejection log examples and PR linked to #25.

---

## #24 — First deterministic rule-based automation engine

### What it means

Rule logic converts fresh, validated sensor state into **proposed** commands. It must be separate from safety enforcement and from physical I/O. Initially use simple transparent rules rather than ML or PID.

### First three rules (simulation-only values)

1. **High-temperature FAN_ON:** reading >= 28°C continuously for >= 60 s, with sensor data valid and fresh, mode AUTO and no interlock.
2. **Low-temperature FAN_OFF:** reading <= 26°C continuously for >= 60 s, similarly validated.
3. **Grow-light schedule:** propose LIGHT_ON/OFF according to a configurable daily schedule while in AUTO.

The 28/26°C and 60-second figures are **illustrative test parameters**, not approved growing setpoints. Ask Plant Science to validate the real growing ranges. A production controller must also account for sensor placement, fan characteristics and environmental response.

### Steps to implement

1. **Agree on a farm-state input model:** sensor ID/type, value/unit, timestamp, quality, device health, zone, mode, last actuator state.
2. **Choose rule configuration format:** database configuration from #22 or a clearly documented local configuration in early unit tests.
3. **Implement a pure rule evaluation function:** given validated state/configuration and current time, return proposed command(s) and explanatory rule event(s).
4. **Add hysteresis:** use separate ON/OFF thresholds so the fan does not rapidly chatter near one value.
5. **Add debounce/duration:** the condition must persist for the configured time; one isolated spike should not trigger actuation.
6. **Check freshness/quality:** never treat missing, corrupt or stale measurements as valid control input.
7. **Check modes/interlocks:** integrate with #25; do not allow rules to bypass manual override or safety policy.
8. **Avoid command storms:** only generate commands on meaningful state changes and use deduplication/cooldown rules. Define how repeat triggers are logged.
9. **Log each evaluation/trigger** with rule ID, inputs, timestamps, action request, and rejection/decision reason.
10. **Write tests** for threshold crossings, chatter, duration, invalid input, manual override, and rule disable.
11. **Document all thresholds as test values** until Plant Science confirms real parameters and testing.

Illustrative evaluation pseudocode:

~~~python
def evaluate_fan_rule(temp_c, elapsed_above_s, elapsed_below_s,
                      current_fan_state, valid_and_fresh):
    if not valid_and_fresh:
        return None

    if current_fan_state == "OFF" and temp_c >= 28.0 and elapsed_above_s >= 60:
        return {"action": "ON", "source": "automation"}

    if current_fan_state == "ON" and temp_c <= 26.0 and elapsed_below_s >= 60:
        return {"action": "OFF", "source": "automation"}

    return None
~~~

This simple function only proposes actions. The state machine and local controller still enforce safety. Persist condition duration robustly; do not count repeated copies of the same reading as continuous independent observations.

### Test sequence to demo

~~~text
t=0   temp=27.0°C -> no request
t=10  temp=28.5°C -> begin high-temp timer
t=40  temp=29.0°C -> no request yet
t=70  temp=29.1°C -> propose FAN_ON
t=90  temp=27.0°C -> keep fan ON (hysteresis)
t=110 temp=25.5°C -> begin low-temp timer
t=170 temp=25.6°C -> propose FAN_OFF
~~~

Use a simulated clock and deterministic timestamps, not real sleeps, for unit tests. Model observation frequency and missing readings.

**Deliverables:** configurable rule engine, at least three rules, logs and unit tests, integration with #25 gate, PR linked to #24.

---

## #26 — End-to-end automation simulator

### What it means

This milestone proves all previous pieces fit together **without physically operating anything**.

### Steps to implement

1. **Create simulated telemetry** for at least one temperature sensor and one fan actuator; use timestamps and a repeatable random seed or fixed scenarios.
2. **Feed telemetry into the same validation path** planned for the real FastAPI endpoint (coordinate with #12).
3. **Write readings to local Supabase** using the #13 schema; handle backend/database failures explicitly.
4. **Evaluate configured rules** using the engine from #24; create a recorded automation event.
5. **Send proposed commands through the #25 gate** for mode, freshness, expiry, interlocks and precedence.
6. **Record command lifecycle** in #22 tables before mock dispatch, then update on ACK/failure.
7. **Simulate an ESP32** that verifies allowed command, ID, expiry and local simulated interlocks, updates state, and returns ACK.
8. **Persist device-reported state** separately from requested state.
9. **Run all failure scenarios** listed below, and show the resulting database event/command history.
10. **Document one command-line demo** and optionally a dashboard readout. Add repeatable automated tests and screenshots/log snippets.
11. **Open a PR** explaining the commands to reproduce the demo from a clean checkout.

Suggested entry point (exact CLI is to be implemented/documented by the contributor):

~~~text
python -m backend.automation.simulator
~~~

This is a **target interface**, not an existing verified executable.

### Mandatory scenario matrix

| Scenario | What to simulate | Expected result |
| --- | --- | --- |
| Normal | Fresh temp exceeds high threshold long enough | One FAN_ON command, ACK, simulated state ON, event stored |
| Expiry | Command expires before mock receives it | Reject/expire; fan state unchanged |
| Offline | Device is unavailable | No false completion; failure/timeout recorded |
| Manual override | Auto rule triggers in MANUAL | Suppress/reject AUTO request; manual state preserved |
| Interlock | Low reservoir + pump start | Pump start rejected; reason logged |
| Lost/delayed ACK | Mock executes late or drops ACK | Pending timeout/unknown outcome is explicit; safe, idempotent reconciliation |
| Reconnect | Offline mock returns after request expired | Old expired command never replayed |
| Invalid/stale data | Sensor data invalid or too old | No automatic actuation |
| Database unavailable | Telemetry/command write fails | No unsafe assumption that cloud command completed; local fallback policy explicit |

For the lost-ACK case, the actuator might have changed state even though the backend never received confirmation; do not blindly retry an actuation. Query/mock-reconcile actual reported state before proceeding, using unique IDs and idempotent handling.

### Demonstration script for a team meeting

1. Start local Supabase and the backend simulation.
2. Display a valid 27°C reading and current fan OFF.
3. Advance the virtual clock through >=28°C for 60 seconds.
4. Show rule event, command created, mock ACK and new fan state.
5. Switch to MANUAL and trigger the rule again; show rejection.
6. Inject an expired command and an offline device; show failure records.
7. Recover device and prove expired command did not execute.
8. Query Supabase to trace the original rule ID, command ID and reported state.

**Deliverables:** working local simulator, setup steps, reproducible scenarios, test results, Supabase audit history query and PR linked to #26.

---

## Next steps after these assignments (not part of the first milestone)

- Integrate with hardware only after validated fail-safe requirements and approved electrical/mechanical testing.
- Build full feedback control using local sensor data; estimate dynamic response before choosing a controller.
- Use PID only when actuators are variable-output and a measured, appropriately sampled process benefits from continuous regulation.
- Keep automatic nutrient/pH chemical dosing and other hazardous actions outside the first simulated milestone.
- Add metrics: command latency, rejected/expired counts, device heartbeat freshness, acknowledgement rate and sensor data quality.

## Reviewer checklist for every automation PR

- [ ] Traces its changes to the linked GitHub issue.
- [ ] Runs entirely with mock devices/synthetic data.
- [ ] Documents input/output, allowable actuator commands, units and errors.
- [ ] Handles OFF, MANUAL, AUTO and SAFE states.
- [ ] Applies expiry, timeout, override and interlock checks.
- [ ] Separates requested command, ACK and reported physical/mock state.
- [ ] Is idempotent under retries/reconnects.
- [ ] Logs why commands fail or are rejected.
- [ ] Has deterministic normal and negative tests.
- [ ] Does not embed Supabase privileged secrets or enable hardware without review.
- [ ] Gives another team member reproducible setup and test steps.
