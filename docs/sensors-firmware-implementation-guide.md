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
