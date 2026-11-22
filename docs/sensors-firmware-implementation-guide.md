# QVFT Stack — Sensors & Firmware Step-by-Step Guide

**Companion:** [Implementation roadmap](implementation-roadmap.md) | [Database & Supabase guide](database-implementation-guide.md) | [Automation & Control guide](automation-control-implementation-guide.md) | [Intelligence guide](intelligence-fullstack-implementation-guide.md)

**Scope:** Implementation assignments for [#10](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/10), [#11](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/11), [#35](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/35), [#36](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/36), [#37](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/37), [#38](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/38), [#39](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/39), and [#40](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/40). These are implementation **instructions and examples**, not claims that the hardware/firmware is already complete.

> **Hardware safety boundary:** Do not connect unknown-voltage sensors, relays, pumps, mains-powered loads, dosing equipment, or other actuators without checking datasheets, voltage/current requirements, grounding, isolation, and the relevant Mechanical/Electrical safety requirements. Start with low-voltage sensors and mock data where possible.

## Before starting

Understand the role of firmware in the QVFT Stack:

~~~text
Physical sensor
    ↓
ESP32 / Arduino
    ↓
sensor driver
    ↓
validation / quality state
    ↓
telemetry JSON
    ↓
HTTP / MQTT
    ↓
FastAPI backend
    ↓
Supabase / PostgreSQL
    ↓
Dashboard / Intelligence / Automation
~~~

Firmware is responsible for reliable interaction with physical hardware.

It should:

- initialize sensors
- read measurements
- detect obvious sensor failures
- identify the device and sensor
- timestamp or sequence readings appropriately
- format telemetry consistently
- communicate with the backend
- report device health
- recover from temporary communication failures
- remain understandable and testable by future QVFT members

Firmware should **not**:

- contain privileged Supabase credentials
- connect directly to the database using service credentials
- silently convert sensor failures into believable values
- directly decide plant-science thresholds unless explicitly assigned
- rely on cloud connectivity for critical physical safety

---

## Recommended task order

~~~text
#10  Inventory sensors and controllers
 ↓
#11  Define standard telemetry format
 ↓
#35  Set up ESP32 firmware development structure
 ↓
#36  Implement reusable sensor driver + first sensor
 ↓
#37  Implement telemetry transport + reconnect behavior
 ↓
#38  Define calibration / validation / data quality
 ↓
#39  Add device health / heartbeat / diagnostics
 ↓
#40  Add buffering + reliability testing
 ↓
#18  Contribute to end-to-end integration testing
~~~

Some of these can run in parallel after the interfaces are agreed.


## Engineering checks and balances

For this workstream, **"the code compiles" is not enough**.

Every firmware task should pass five gates before it is considered complete:

~~~text
1. BUILD
   Does it compile / run?

2. FUNCTION
   Does the intended feature work?

3. FAILURE
   What happens when the sensor, network, backend, or configuration is wrong?

4. RECOVERY
   Does the device recover without requiring a full rewrite or unsafe manual intervention?

5. REPRODUCTION
   Can another QVFT member reproduce the result from the repository and documentation?
~~~

### Evidence required in each PR

The contributor should include evidence appropriate to the task:

- exact build command used
- build output or test output
- board/controller model
- firmware version
- wiring/pin map where hardware is involved
- serial-monitor logs
- example telemetry/heartbeat payloads
- screenshots or terminal output from the receiving backend where relevant
- failure-injection test results
- recovery test results
- known limitations
- exact reproduction steps for another member

Do not accept statements such as:

> "It worked on my computer."

Prefer evidence such as:

~~~text
Test: disconnect temperature sensor during runtime
Expected: TEMP-01 changes to read_error, device remains online
Observed: PASS
Evidence: serial log + heartbeat payload attached
~~~

### Standard test record

For each meaningful test, record:

| Field | What to record |
| --- | --- |
| Test ID | Short stable name |
| Purpose | What behavior is being verified |
| Setup | Hardware/software/configuration used |
| Steps | Exact actions taken |
| Expected | What should happen |
| Observed | What actually happened |
| Result | PASS / FAIL / BLOCKED |
| Evidence | Log, screenshot, payload, test output |
| Notes | Limitation, follow-up, issue number |

A failed test is useful. Do not hide it. Document it and either fix it in the same PR or open a follow-up Issue if it is outside scope.

### Second-person verification

At least one important milestone per contributor should be reproduced by someone other than the original author.

The reviewer should:

1. pull the branch
2. follow only the documented setup
3. build/run it
4. repeat at least one normal test
5. repeat at least one failure test

If the reviewer needs undocumented instructions from the author, documentation is incomplete.

---


# #10 — Inventory QVFT sensors and controllers

## Why this exists

Before writing code, the firmware owner needs to know what hardware actually exists.

A sensor integration is not just a name like "pH sensor." You need to know:

- exact model
- electrical interface
- supply voltage
- logic voltage
- signal range
- connector
- calibration requirements
- library/protocol availability
- physical location
- controller compatibility

## Step by step

1. Gather every sensor/controller currently owned or planned.
2. Record the exact manufacturer/model where possible.
3. Find the official datasheet/manual.
4. Record:
   - supply voltage
   - signal/interface
   - expected current
   - measurement range
   - accuracy/resolution
   - calibration requirements
   - operating environment
   - connector/pinout
5. Identify the intended controller:
   - ESP32
   - Arduino
   - other
6. Identify whether level shifting/isolation/ADC conditioning may be required.
7. Mark unknown information explicitly.
8. Photograph/label hardware if useful for team identification.
9. Put the inventory under `docs/`.
10. Link each future firmware task to the hardware inventory.

Suggested table:

| Device | Model | Type | Interface | Supply | Logic/signal | Controller | Calibration | Status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Air temp/RH | TBD | sensor | I2C | TBD | TBD | ESP32 | manufacturer procedure | planned |
| pH probe/interface | TBD | sensor | analog/UART/TBD | TBD | TBD | ESP32 | required | planned |
| EC probe/interface | TBD | sensor | analog/UART/TBD | TBD | TBD | ESP32 | required | planned |

Do not guess missing electrical values.

**Acceptance demonstration:** Another member can pick up a listed sensor and identify what documentation, voltage, interface, and controller are required before wiring it.

---

# #11 — Define standard telemetry format

## Why this exists

Every sensor should not invent a different payload.

The backend needs predictable fields so it can validate and store readings.

## Minimum concepts

A telemetry reading should identify:

- device
- sensor
- physical zone
- measurement type
- numeric value
- unit
- measurement timestamp
- data-quality/status
- firmware version where useful
- message/sequence identity where useful

Illustrative payload:

~~~json
{
  "device_id": "qvft-zone-01",
  "sensor_id": "temp-01",
  "zone_id": "growth-zone-01",
  "sensor_type": "air_temperature",
  "value": 23.4,
  "unit": "C",
  "quality": "valid",
  "recorded_at": "2026-11-22T19:00:00Z",
  "firmware_version": "0.1.0",
  "sequence": 1842
}
~~~

This is illustrative; coordinate exact names with backend/database owners before locking it.

## Step by step

1. Agree on canonical field names.
2. Agree on canonical sensor type names.
3. Agree on units.
4. Use timezone-aware UTC timestamps when the device has reliable time.
5. Define behavior when the device clock is unavailable.
6. Define quality states.
7. Decide whether one request contains:
   - one reading
   - many readings
   - both heartbeat + telemetry
8. Define message/sequence IDs for duplicate handling.
9. Document examples for every currently planned sensor type.
10. Validate the payload against backend expectations.

## Design questions

- Can the backend distinguish measurement time from receive time?
- Can duplicate retries be identified?
- Are units explicit?
- Can sensor failure be represented without fake values?
- Is firmware version available for debugging?

**Acceptance demonstration:** A mock ESP32 payload can be passed to the backend owner and they can validate it without guessing what any field means.

---

# #35 — Set up ESP32 firmware development structure

## Why this exists

A team firmware project needs to be reproducible. "It builds on my laptop" is not enough.

The project should separate:

~~~text
sensor hardware code
network code
telemetry formatting
device health
configuration
tests
~~~

## Toolchain choice

Choose one primary toolchain with the Stack lead.

For many student projects, an ESP32 workflow may use either:

- PlatformIO + Arduino framework
- Arduino IDE / Arduino CLI
- ESP-IDF

Do not maintain multiple competing firmware architectures unless there is a reason.

The chosen toolchain should support:

- repeatable builds
- version-controlled configuration
- serial logging
- dependency management
- future automated testing where practical

## Suggested structure

Adapt to the chosen framework:

~~~text
firmware/
  README.md
  src/
    main.cpp
  include/
  sensors/
  networking/
  telemetry/
  health/
  config/
  tests/
~~~

Example conceptual `main.cpp` flow:

~~~cpp
void setup() {
    initialize_logging();
    initialize_device_identity();
    initialize_sensors();
    initialize_network();
}

void loop() {
    read_sensors();
    update_device_health();
    send_due_telemetry();
    service_network();
}
~~~

Avoid putting every function in one enormous file.

## Step by step

1. Select target ESP32 board.
2. Select firmware framework/toolchain.
3. Create the base project.
4. Build an empty/minimal firmware image.
5. Flash it.
6. Open serial monitor.
7. Print:
   - device ID
   - firmware version
   - boot/reset reason where available
8. Separate secrets/configuration.
9. Document exact build/flash commands.
10. Ask a second member to reproduce the build.

## Versioning

At minimum, firmware should identify its version:

~~~text
QVFT firmware v0.1.0
device: qvft-zone-01
~~~

This becomes valuable when several ESP32s are deployed.

**Acceptance demonstration:** Another member clones the repo on a clean machine, follows the README, builds, flashes, and sees the expected boot log.

---

# #36 — Reusable sensor driver + first sensor integration

## Why this exists

Sensor code should not be mixed directly with Wi-Fi requests, JSON construction, and automation logic.

The goal is:

~~~text
physical sensor
    ↓
driver
    ↓
normalized reading
~~~

## Learn the interface first

The member should understand the basics of:

- I2C
- SPI
- UART
- ADC / analog signals
- GPIO

They do not need to become an electrical engineer before coding, but they should know what interface their sensor uses and why.

## Suggested normalized reading

Conceptually:

~~~cpp
struct SensorReading {
    const char* sensor_id;
    const char* sensor_type;
    float value;
    const char* unit;
    bool valid;
    unsigned long measured_ms;
};
~~~

A real implementation may use different types/classes.

## Step by step

1. Choose one low-risk first sensor.
2. Read the official datasheet.
3. Confirm wiring and voltage.
4. Verify the sensor using the simplest manufacturer/library example first.
5. Move working sensor access into QVFT's driver structure.
6. Implement:
   - initialization
   - read
   - status/error reporting
7. Normalize units.
8. Return an explicit failure/quality state.
9. Create a mock/fake sensor implementation.
10. Test repeated reads and disconnected-sensor behavior.

## Failure behavior

Bad:

~~~cpp
if (sensor_failed) {
    temperature = 0.0;
}
~~~

Why? Because 0°C could be a real measurement.

Better conceptually:

~~~text
value: unavailable
quality: invalid
error: SENSOR_READ_FAILED
~~~

## Questions the contributor should answer

- Why is this sensor using I2C/SPI/UART/ADC?
- What voltage does it require?
- What happens when it disconnects?
- What library is being used?
- Are units converted?
- How often is it safe/useful to sample?

**Acceptance demonstration:** Show real or simulated sensor reads, then disconnect/fail the sensor and show that firmware reports a failure instead of inventing a believable value.

---

# #37 — Telemetry transport and reconnect behavior

## Why this exists

Reading a sensor locally is only half the job.

The Stack needs:

~~~text
sensor reading
   ↓
telemetry payload
   ↓
network request
   ↓
FastAPI
~~~

Use the backend rather than connecting ESP32s directly to Supabase with privileged credentials.

## HTTP vs MQTT

The architecture supports HTTP and/or MQTT.

For a first implementation, HTTP may be simpler for occasional telemetry.

MQTT becomes useful when the project needs:

- persistent messaging
- many devices
- publish/subscribe
- command channels
- frequent real-time updates

Choose deliberately rather than implementing both immediately.

## Step by step

1. Connect ESP32 to development Wi-Fi.
2. Confirm network status.
3. Build a payload matching #11.
4. Send to a local/test FastAPI endpoint.
5. Handle response status.
6. Set reasonable request timeouts.
7. Detect network loss.
8. Reconnect without blocking the full firmware forever.
9. Add retry/backoff.
10. Expose delivery failure through logs/device health.

Illustrative logic:

~~~cpp
if (network_available()) {
    if (telemetry_due()) {
        auto result = send_telemetry(reading);

        if (!result.success) {
            queue_or_mark_failed(reading);
        }
    }
}
~~~

Avoid an infinite tight retry loop.

## Secrets

Never commit:

~~~text
real Wi-Fi passwords
backend private keys
Supabase service/secret keys
production tokens
~~~

Use local development configuration and placeholders.

**Acceptance demonstration:** Send valid synthetic/real telemetry to a test backend, disconnect Wi-Fi, show graceful failure/reconnect behavior, then restore Wi-Fi and resume operation.

---

# #38 — Calibration, validation and quality procedures

## Why this exists

A sensor returning a number does not mean the number is trustworthy.

This is especially important for measurements such as:

- pH
- EC
- analog probes
- water-level sensors
- optical sensors

## Separate four concepts

### Calibration

Adjust/characterize the measurement against a known reference.

### Validation

Check whether the result is plausible and system behavior is correct.

### Data quality

Describe confidence/status of the reading.

### Plant operating range

A biological/agronomic range validated by Plant Science.

These are not the same thing.

## Suggested quality states

~~~text
valid
suspect
invalid
~~~

Freshness can be derived separately:

~~~text
fresh
stale
~~~

## Step by step

For each integrated sensor:

1. Read manufacturer calibration instructions.
2. Identify reference materials/equipment.
3. Record calibration procedure.
4. Record calibration date.
5. Record coefficients/offsets if required.
6. Define obvious hardware/measurement errors.
7. Define how quality is reported.
8. Test before/after calibration where possible.
9. Document recalibration/check interval.
10. Coordinate plant-specific ranges with Plant Science.

## pH / EC caution

Do not treat these as generic plug-and-play values.

Consider:

- probe condition
- calibration solutions
- storage requirements
- temperature compensation
- electrical noise
- grounding
- settling time
- contamination

Use the exact hardware documentation for the chosen interface.

**Acceptance demonstration:** Perform or document one repeatable sensor check/calibration and show how the firmware/database can distinguish valid from suspect/invalid results.

---

# #39 — Device health, heartbeat and diagnostics

## Why this exists

If a zone suddenly reports no temperature change, QVFT must know whether:

- the environment is stable
- the sensor is dead
- the ESP32 is offline
- Wi-Fi failed
- the backend rejected messages

## Suggested health information

~~~text
device_id
firmware_version
uptime
RSSI / Wi-Fi state
last sensor read
last successful telemetry send
sensor availability
reconnect count
reset reason
fault/error codes
~~~

Illustrative heartbeat:

~~~json
{
  "device_id": "qvft-zone-01",
  "firmware_version": "0.1.0",
  "uptime_s": 18342,
  "wifi_rssi_dbm": -61,
  "status": "degraded",
  "sensor_status": {
    "TEMP-01": "ok",
    "PH-01": "read_error"
  }
}
~~~

## Step by step

1. Define heartbeat interval.
2. Define health fields.
3. Track sensor initialization/read state.
4. Track network state.
5. Track last successful telemetry.
6. Expose reset reason where available.
7. Define error codes.
8. Coordinate backend offline timeout.
9. Simulate one sensor failure.
10. Simulate full device/network failure.

## Important distinction

~~~text
device online
≠
all sensors healthy
~~~

An ESP32 can be online while one connected sensor is broken.

**Acceptance demonstration:** Backend/mock receiver can distinguish healthy, degraded, and offline-like conditions from heartbeat/status information.

---

# #40 — Offline buffering and firmware reliability

## Why this exists

Wi-Fi and backend services will occasionally fail.

A device should not:

- freeze
- crash
- retry thousands of times per second
- grow an unlimited queue
- lose measurement timestamps
- replay ancient commands/readings blindly

## Buffering strategy

Define:

- maximum readings
- maximum age
- memory/storage location
- behavior when full
- replay order
- duplicate identity

Example concept:

~~~text
Buffer capacity: N readings
Policy when full: drop oldest / documented alternative
Preserve: original recorded_at + sequence
Reconnect: send oldest valid queued reading first
~~~

Exact values should be chosen from device memory and operational needs.

## Retry strategy

Use bounded retry/backoff.

Conceptually:

~~~text
attempt 1 -> wait short interval
attempt 2 -> wait longer
attempt 3 -> wait longer
...
cap retry delay
~~~

Do not block sensor reading while waiting for the network if the firmware architecture allows avoiding it.

## Reliability tests

At minimum:

1. sensor disconnected
2. Wi-Fi disconnected
3. backend unavailable
4. malformed backend response
5. repeated reconnects
6. device reboot during outage
7. buffer reaches capacity
8. queued readings replay after reconnect
9. duplicate/retry handling
10. long-running loop remains responsive

## Watchdog

If using a watchdog, understand:

- what code is expected to service it
- what failure causes reset
- what happens after reset
- how reset reason is reported

Do not use a watchdog as a substitute for fixing blocking code.

**Acceptance demonstration:** Simulate a network outage, continue collecting bounded telemetry locally, restore connectivity, and show controlled recovery without a tight retry loop or unbounded buffer.

---

# Contribution to #18 — End-to-end integration testing

The firmware owner should help verify:

~~~text
sensor/mock reading
  -> firmware normalization
  -> telemetry JSON
  -> FastAPI
  -> database
  -> dashboard
~~~

## Firmware-related test cases

- valid reading
- invalid reading
- missing sensor
- sensor disconnect
- device reconnect
- malformed telemetry
- duplicate telemetry
- old/stale reading
- backend unavailable
- Wi-Fi unavailable
- firmware reboot
- heartbeat timeout

The result should clearly identify which component failed.

---


# Detailed verification gates by Issue

The following are the **minimum checks you, as Stack lead, can use to decide whether the work actually functions**.

---

## Verification gate for #10 — Hardware inventory

### Contributor must prove

- every physical sensor/controller they claim to support has an exact model or is explicitly marked unknown
- voltage and interface information comes from a datasheet/manual or is marked unknown
- no unknown-voltage device is wired just to "see if it works"
- the intended ESP32/Arduino compatibility is documented
- calibration requirements are noted

### Lead test

Pick one item from the inventory at random and ask the contributor:

1. What exact model is this?
2. What voltage powers it?
3. What signal/interface does it use?
4. Which ESP32 pins/interfaces could connect to it?
5. Does it need calibration?
6. What document did you use to determine this?

**Pass:** they can answer from the inventory/datasheet without guessing.

**Fail:** important electrical values are assumptions, copied from a different model, or missing without being labelled unknown.

---

## Verification gate for #11 — Telemetry contract

### Contributor must prove

The same payload can be understood by:

- firmware
- backend
- database owner
- dashboard/intelligence owner

### Required tests

#### T11-01 — Valid payload

Create one known-good payload.

Expected:
- backend validation accepts it
- identifiers are unambiguous
- unit is explicit
- timestamp can be parsed
- quality state is recognized

#### T11-02 — Missing required field

Delete `sensor_id` or another required field.

Expected:
- validation rejects it
- error explains what is missing
- no malformed reading is silently inserted

#### T11-03 — Wrong type

Example:

~~~json
{
  "value": "twenty-three"
}
~~~

Expected:
- rejected or explicitly normalized according to a documented rule
- never silently treated as a valid numeric reading

#### T11-04 — Duplicate message

Send the same message/sequence identifier twice.

Expected:
- duplicate behavior matches the documented policy
- repeated delivery must not create uncontrolled duplicate downstream actions

#### T11-05 — Timestamp problem

Send missing/invalid/old time data.

Expected:
- firmware/backend behavior is documented
- measurement time and receive time remain distinguishable

### Lead test

Ask the backend owner to consume the payload without asking the firmware author what each field means.

**Pass:** the contract is sufficient by itself.

---

## Verification gate for #35 — Firmware project setup

### Contributor must prove

A clean machine can build the firmware.

### Required tests

#### T35-01 — Clean build

From a fresh checkout:

~~~powershell
git clone https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-.git
cd QVFT-Stack-
git switch <feature-branch>
# follow only firmware/README.md
~~~

Expected:
- dependencies install
- firmware compiles
- no local-only path is required

#### T35-02 — Flash

Expected:
- selected ESP32 accepts the firmware
- flash procedure is documented

#### T35-03 — Boot identity

Expected serial output should contain at least:

~~~text
device_id=<expected ID>
firmware_version=<expected version>
boot=<successful>
~~~

#### T35-04 — Secret scan

Search the branch for:

- Wi-Fi passwords
- tokens
- private keys
- Supabase privileged keys
- hard-coded production endpoints if they are meant to be configurable

Expected:
- no secrets committed

#### T35-05 — Reproduction by another member

A second member performs the build/flash using only the guide.

**Pass:** another person can reproduce it.

---

## Verification gate for #36 — Sensor driver

### Contributor must prove

The driver handles both **correct measurements and sensor failure**.

### Required tests

#### T36-01 — Initialization

Power/connect the sensor.

Expected:
- initialization succeeds
- sensor identity/status is logged

#### T36-02 — Repeated reads

Collect a meaningful sequence of readings rather than one sample.

Expected:
- firmware remains stable
- units are consistent
- no obvious unbounded memory/logging behavior

#### T36-03 — Known-condition sanity test

Expose the sensor to a safe, obvious change appropriate to the device.

Examples:
- temperature sensor: warm/cool environment slightly
- light sensor: cover/uncover
- reservoir/level simulator: change known input

Expected:
- reading changes in the physically expected direction

This is a **sanity test**, not a substitute for calibration.

#### T36-04 — Disconnect sensor

Disconnect the sensor or simulate a read failure.

Expected:
- driver returns an explicit error/invalid state
- firmware does not report a believable fake value
- the rest of the device remains responsive

#### T36-05 — Reconnect sensor

Restore the sensor.

Expected:
- driver recovers according to documented behavior
- quality/status returns to normal
- reboot requirement, if any, is explicitly documented

#### T36-06 — Mock driver

Run without the physical sensor using a fake/synthetic driver.

Expected:
- downstream telemetry can still be tested

### Lead questions

Ask:

- Where in the code is hardware-specific access?
- Where are units normalized?
- How is a read failure represented?
- What happens if initialization succeeds but a later read fails?

If the contributor cannot point to these locations, they do not yet understand the implementation.

---

## Verification gate for #37 — Networking and telemetry

### Contributor must prove

Telemetry works when the network is healthy **and fails safely when it is not**.

### Required tests

#### T37-01 — Successful send

Expected chain:

~~~text
sensor/mock reading
→ JSON payload
→ network request
→ FastAPI accepts
→ success recorded in firmware log
~~~

Evidence:
- serial log
- backend request/log
- example payload

#### T37-02 — Wrong backend address

Expected:
- timeout/failure is bounded
- firmware remains responsive
- failure is visible

#### T37-03 — Wi-Fi removed

Disable the development Wi-Fi or use an unreachable SSID.

Expected:
- sensor reading loop continues where appropriate
- reconnect attempts follow documented timing
- no tight retry loop
- device health reflects network problem

#### T37-04 — Wi-Fi restored

Expected:
- connection returns without reflashing
- telemetry resumes

#### T37-05 — Backend returns error

Test representative 4xx and 5xx behavior using a mock backend where practical.

Expected:
- firmware distinguishes delivery failure from successful delivery
- it does not mark an unsuccessful request as sent successfully

#### T37-06 — Payload validation failure

Send a deliberately invalid payload from the test path.

Expected:
- backend rejects it
- firmware/logging captures the failure

### Lead pass condition

Disconnect Wi-Fi **while the demo is running**. If the whole firmware freezes or requires reflashing to recover, the task is not complete.

---

## Verification gate for #38 — Calibration and measurement quality

### Contributor must prove

They understand the difference between:

- a sensor producing a number
- a calibrated/checked measurement
- a valid value
- a stale value
- a plant-acceptable value

### Required checks

#### T38-01 — Reference comparison

Compare against a suitable known/reference condition where practical.

Record:

~~~text
reference value
sensor value
difference
date/time
sensor model
calibration state
~~~

#### T38-02 — Repeatability

Take repeated measurements under a stable condition.

Expected:
- variation is recorded
- unexplained large jumps are investigated

#### T38-03 — Invalid condition

Create or simulate an impossible/error reading.

Expected:
- quality becomes suspect/invalid according to policy
- the value is not silently treated as trustworthy

#### T38-04 — Stale data

Stop new readings while keeping an old value stored.

Expected:
- old historical data remains historical
- downstream code can identify that the current state is stale

#### T38-05 — Calibration record

Expected:
- calibration/check date and relevant metadata are recorded
- procedure can be repeated by another member

### pH/EC extra sign-off

For pH/EC work, require:
- exact probe/interface model
- manufacturer calibration procedure
- calibration solutions/reference values used
- temperature-compensation behavior if supported/required
- storage/conditioning notes
- evidence that electrical noise/grounding was considered

Do not approve "the pH sensor works" based only on one plausible-looking number.

---

## Verification gate for #39 — Health and diagnostics

### Contributor must prove

The system can tell the difference between:

~~~text
healthy device
degraded device
offline device
~~~

and can identify individual sensor faults.

### Required tests

#### T39-01 — Healthy heartbeat

Expected:
- correct device ID
- firmware version
- uptime
- network status
- sensor statuses

#### T39-02 — One sensor fails

Expected:
- device stays online
- device becomes degraded or equivalent
- failed sensor is identified
- healthy sensors remain distinguishable

#### T39-03 — Network loss

Expected:
- health state/log records communication problem
- firmware itself remains running

#### T39-04 — Reboot

Expected:
- uptime resets
- reset reason is reported where supported
- device identity remains correct

#### T39-05 — Backend offline timeout

Coordinate with backend owner.

Expected:
- after the agreed heartbeat timeout, backend can mark device offline
- when heartbeat resumes, status returns according to policy

### Lead demo

During the demo, unplug only one sensor.

Ask:

> "Is the ESP32 offline, or is one sensor broken?"

The system should make that answer obvious.

---

## Verification gate for #40 — Reliability and buffering

### Contributor must prove

Temporary failures do not cause data explosion, firmware lockup, or uncontrolled replay.

### Required tests

#### T40-01 — Short outage

Disable network briefly.

Expected:
- readings continue according to local design
- failed deliveries are buffered/marked according to policy
- queue does not exceed its configured bound

#### T40-02 — Buffer-full condition

Force enough failed sends to fill the buffer.

Expected:
- documented overflow policy occurs
- firmware remains stable
- no heap/memory runaway

#### T40-03 — Recovery

Restore the network.

Expected:
- queued data is processed according to documented order
- original measurement timestamps remain intact

#### T40-04 — Duplicate/retry

Force retry of a previously sent message.

Expected:
- sequence/message identity allows duplicate behavior to be identified downstream

#### T40-05 — Reboot during outage

Expected:
- behavior matches documented persistence policy
- contributor can explain what buffered data is lost or retained

#### T40-06 — Long-run test

Run for an agreed extended development period with regular telemetry.

Record:
- reset count
- reconnect count
- send failures
- memory/resource observations where available
- sensor read failures

The exact duration can scale with development stage; the important part is that long-running behavior is tested rather than inferred from a two-minute demo.

### Lead pass condition

The contributor must be able to state:

- maximum buffer size
- retry policy
- overflow behavior
- duplicate behavior
- what survives reboot
- what does not

If these are unknown, the reliability design is incomplete.

---

# End-to-end system verification

After the individual tasks pass, run one integrated test with the firmware, backend and database owners.

## E2E-01 — Healthy path

~~~text
physical/mock sensor
→ firmware read
→ normalized reading
→ JSON telemetry
→ FastAPI validation
→ database insert
→ latest reading query
~~~

**Pass:** the value, unit, sensor ID, device/zone relationship and measurement time remain correct end-to-end.

## E2E-02 — Sensor failure

Break/disconnect the sensor.

**Pass:**
- firmware detects error
- no fake environmental measurement is stored as valid
- device-health information reflects the problem

## E2E-03 — Network failure

Disconnect network.

**Pass:**
- firmware remains responsive
- failure is reported
- recovery occurs after network restoration

## E2E-04 — Backend/database failure

Make the receiving service unavailable in a controlled test environment.

**Pass:**
- firmware does not interpret failure as success
- retry/buffering behavior matches design
- no uncontrolled rapid requests

## E2E-05 — Duplicate delivery

Replay one telemetry message.

**Pass:** downstream behavior matches the team's duplicate/idempotency policy.

## E2E-06 — Stale reading

Stop telemetry long enough to cross the defined freshness timeout.

**Pass:** dashboard/intelligence/backend can identify that the most recent stored value is no longer current.

---

# Stack-lead sign-off sheet

Before you mark the firmware milestone complete, you should be able to check all of these:

### Build/reproduction

- [ ] Another member can clone and build.
- [ ] Another member can flash/run.
- [ ] Toolchain/version requirements are documented.
- [ ] No machine-specific paths are required.

### Hardware

- [ ] Exact board/sensor model documented.
- [ ] Wiring/pins documented.
- [ ] Voltage/interface checked.
- [ ] Unknown electrical details are not guessed.

### Sensor behavior

- [ ] Normal reading demonstrated.
- [ ] Repeated readings demonstrated.
- [ ] Sensor disconnect demonstrated.
- [ ] Failure is represented explicitly.
- [ ] Sensor can recover or recovery requirement is documented.

### Telemetry

- [ ] Standard payload demonstrated.
- [ ] Backend accepts valid data.
- [ ] Backend rejects bad data.
- [ ] Timestamps/units/IDs remain correct.

### Network/recovery

- [ ] Wi-Fi loss demonstrated live.
- [ ] Backend outage demonstrated.
- [ ] Retry is bounded.
- [ ] Reconnect succeeds.
- [ ] Firmware does not freeze.

### Device health

- [ ] Healthy heartbeat demonstrated.
- [ ] Individual sensor fault demonstrated.
- [ ] Firmware version visible.
- [ ] Offline/degraded semantics are documented.

### Data quality

- [ ] Calibration/check process documented.
- [ ] Invalid vs stale distinguished.
- [ ] No fake fallback measurements.
- [ ] Plant thresholds are owned/validated outside firmware.

### Reliability

- [ ] Buffer has a maximum size.
- [ ] Buffer-full behavior tested.
- [ ] Duplicate/replay behavior documented.
- [ ] Reboot/outage behavior documented.
- [ ] Longer-running test completed.

### Learning

- [ ] Contributor can explain their sensor interface.
- [ ] Contributor can explain their code structure.
- [ ] Contributor can identify failure paths.
- [ ] Contributor can modify sampling/network settings without blindly regenerating the project.
- [ ] Contributor states what AI assisted with and what they verified themselves.

If several of these cannot be checked, the work may be a useful prototype, but it should not yet be described as a reliable firmware subsystem.

---

# Suggested first firmware demonstration

Use one low-risk sensor or a simulator.

~~~text
1. ESP32 boots.
2. Serial shows device ID + firmware version.
3. Sensor initializes.
4. Sensor produces reading.
5. Firmware converts it to standard telemetry.
6. Telemetry is sent to local FastAPI.
7. Backend accepts it.
8. Wi-Fi is disconnected.
9. Firmware remains responsive and reports the failure.
10. Wi-Fi is restored.
11. Firmware reconnects and resumes.
12. Disconnect sensor.
13. Device reports degraded/sensor error rather than fake measurement.
~~~

If the first physical sensor is unavailable, use a fake sensor driver that generates known synthetic values. The architecture can still be developed and tested.

---

# Sensors & Firmware PR checklist

- [ ] Exact GitHub Issue linked.
- [ ] Hardware model/datasheet identified where physical hardware is used.
- [ ] Supply and logic voltage checked before wiring.
- [ ] Pin mapping documented.
- [ ] Firmware builds from a clean checkout.
- [ ] Secrets are not committed.
- [ ] Sensor driver is separate from networking logic.
- [ ] Units and sensor IDs match the telemetry contract.
- [ ] Failure is explicit; no magic/fake fallback values.
- [ ] Sensor disconnect behavior tested.
- [ ] Network/backend failure behavior tested.
- [ ] Retry behavior is bounded.
- [ ] Device health is observable.
- [ ] Calibration/quality assumptions are documented.
- [ ] Plant thresholds are not invented by firmware code.
- [ ] Contributor can explain the interface/protocol being used.
- [ ] Another Stack member reviews before merge.

# Learning review questions

Before approving a firmware PR, ask the contributor:

- What interface does this sensor use and why?
- What voltage does it need?
- How does the ESP32 know the read failed?
- What happens if Wi-Fi disappears?
- What happens if the backend is down?
- Why is the sensor driver separate from networking?
- How are duplicates/retries handled?
- How do we distinguish stale data from invalid data?
- What would make this measurement wrong?
- How is the sensor calibrated?
- What did AI help you with?
- What did you personally verify on hardware?
- Can you identify where you would change the sampling interval, sensor driver, or telemetry format without asking AI where the code lives?

The objective is to use AI to speed up firmware development while still learning embedded systems, sensor interfaces, debugging, reliability, and system integration.
