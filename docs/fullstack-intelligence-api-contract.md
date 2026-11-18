# QVFT Farm Insights — Proposed API, Sample Data and Tests

**Purpose:** Shared example contract for the contributor implementing [the Farm Insights Dashboard](fullstack-intelligence-feature-guide.md). This is a **proposal**, not a claim that these API endpoints, SQL views, or files already exist.

**Agreement required:** Backend, Database, Intelligence, and Frontend owners must approve endpoint paths and exact field names before implementation. Current repository architecture calls for React + TypeScript -> authenticated FastAPI -> PostgreSQL/Supabase; this proposal respects that boundary.

## 1. Minimal vertical slice

**User story:** As an authorized QVFT team member, I can select a growing zone, view its recent environmental readings and last 24 hours of temperature, and see understandable warnings (including missing or stale sensor data).

**No actuator control.** Intelligence output is read-only, explanatory and for human review. Values/suggestions do not constitute crop-science prescriptions.

## 2. Proposed read-only endpoints

| Method | Route | Purpose | Example response |
| --- | --- | --- | --- |
| GET | /health | Backend health | { "status": "ok" } |
| GET | /zones | Accessible zones | list of zone objects |
| GET | /zones/{zone_id}/readings/latest | Latest readings for zone | zone, readings array, query time |
| GET | /sensors/{sensor_id}/history?from=...&to=... | Time window of readings | sensor ID, readings array |
| GET | /zones/{zone_id}/insights | Explainable insights | zone, insights array, generated at |

These endpoint names are **proposed**; use existing route conventions if #12 already established them.

### Proposed objects

**Zone**

~~~json
{
  "id": "zone-01",
  "name": "Growth Zone 1",
  "description": "Synthetic demonstration zone"
}
~~~

**Latest readings response**

~~~json
{
  "zone_id": "zone-01",
  "generated_at": "2026-11-18T14:05:00Z",
  "readings": [
    {
      "sensor_id": "temp-01",
      "sensor_type": "air_temperature",
      "value": 29.1,
      "unit": "C",
      "recorded_at": "2026-11-18T14:04:00Z",
      "quality_flag": "valid"
    },
    {
      "sensor_id": "humidity-01",
      "sensor_type": "relative_humidity",
      "value": 65.2,
      "unit": "%",
      "recorded_at": "2026-11-18T14:04:00Z",
      "quality_flag": "valid"
    }
  ]
}
~~~

**History response**

~~~json
{
  "sensor_id": "temp-01",
  "from": "2026-11-17T14:00:00Z",
  "to": "2026-11-18T14:00:00Z",
  "readings": [
    { "recorded_at": "2026-11-17T14:00:00Z", "value": 23.6, "quality_flag": "valid" },
    { "recorded_at": "2026-11-18T14:00:00Z", "value": 28.8, "quality_flag": "valid" }
  ]
}
~~~

**Insight response**

~~~json
{
  "zone_id": "zone-01",
  "generated_at": "2026-11-18T14:05:00Z",
  "insights": [
    {
      "id": "insight-001",
      "zone_id": "zone-01",
      "code": "HIGH_TEMPERATURE_DEMO",
      "severity": "warning",
      "summary": "Simulated temperature above example threshold",
      "explanation": "The latest fresh, valid temperature is 29.1°C, above the demonstration threshold of 28°C.",
      "recommended_action": "Check the sensor and review the zone conditions with Plant Science.",
      "generated_at": "2026-11-18T14:05:00Z",
      "source_sensor_ids": ["temp-01"],
      "status": "active"
    }
  ]
}
~~~

**Note on fixtures:** All times and IDs above are frozen *synthetic examples*. To test freshness, inject a **simulated current time** or dynamically generate timestamps. Do not use November 18, 2026 as the real clock for tests executed on another day. Identifiers like zone-01 differ from the prospective UUID-based SQL schema; map API identifiers to the final database IDs or adapt the fixtures after the DB owner approves the schema.

**Error response proposal**

~~~json
{
  "error": {
    "code": "INVALID_TIME_RANGE",
    "message": "The requested time window is invalid."
  }
}
~~~

- 400: invalid input/time range.
- 401: authentication required.
- 403: user not authorized for this zone.
- 404: unknown zone/sensor.
- 503: downstream backend/database dependency unavailable.
- 200 with an empty readings array: valid request with no measurements.

Do not leak database errors or credentials to clients. Authorize **every requested zone/sensor**, not just successful login.

## 3. API/data interpretation rules

1. **Use UTC ISO 8601 timestamps** on the wire, formatted in the user's timezone in the UI.
2. **Unit discipline:** temperature C, relative humidity %, EC units must be explicitly stated and consistent. If converting units, document formula.
3. **Freshness is time-relative:** a value can be the "latest stored" reading and still be stale. Determine freshness using a configurable threshold and server/query reference time.
4. **Quality is independent from freshness:** a recent but invalid reading is not suitable for decision support. The absence of any valid fresh reading should be explicit.
5. **Missing values are not zero:** show "No data" rather than assigning a misleading 0.
6. **History may have gaps:** preserve timestamps and mark large missing periods; charts must not imply continuous measurement through missing data.
7. **No future-dated data:** test and reject or clearly flag readings that exceed an approved clock-skew allowance.
8. **Read-only APIs in the first project:** no privileged writes or actuator commands from the browser.
9. **Bound queries:** enforce a maximum time window and result count; paginate or aggregate longer histories.
10. **Keep insight rules transparent:** rule ID, condition, evidence, threshold provenance and severity should be understandable to a human.

## 4. Suggested frontend data fetching

Create a typed module such as `frontend/src/api/farmInsights.ts`. For development, use a non-secret API base URL in environment configuration. Vite exposes variables prefixed `VITE_` to frontend code, so **never put passwords, Supabase secret/service-role keys, private tokens or API credentials in these variables**.

~~~typescript
import type { Reading } from "../types/farm";

type LatestResponse = {
  zone_id: string;
  generated_at: string;
  readings: Reading[];
};

const BASE_URL = import.meta.env.VITE_API_BASE_URL || "http://localhost:8000";

export async function getLatestReadings(
  zoneId: string,
  signal?: AbortSignal,
): Promise<LatestResponse> {
  const response = await fetch(
    `${BASE_URL}/zones/${encodeURIComponent(zoneId)}/readings/latest`,
    { signal, credentials: "include" },
  );
  if (!response.ok) {
    throw new Error(`Could not fetch readings (HTTP ${response.status})`);
  }
  return (await response.json()) as LatestResponse;
}
~~~

`credentials: "include"` is appropriate **only if** the project has approved a cookie-based authenticated API and configured narrowly scoped CORS/CSRF protections. If the team adopts bearer tokens or a same-origin proxy, change this implementation accordingly. This example is not a completed authentication design.

Add `AbortController` handling in the component on unmount/zone change. Treat failed requests as errors rather than silently replacing data with zeroes. Validate/normalize server responses at the appropriate boundary in production rather than trusting a TypeScript type assertion alone.

## 5. Test matrix (reviewer should exercise each case)

| Test | Input / situation | Expected UI and API behavior |
| --- | --- | --- |
| Fresh valid sensor | New, valid temperature record | Correct value/unit/timestamp and plotted point |
| No readings | Valid zone, empty result | Explicit empty state, no fabricated zero |
| Stale last reading | Measurement older than freshness limit | Mark stale; do not claim "temperature right now" |
| Invalid reading | Quality invalid | Do not use for live high-temperature insight |
| Duplicate reading | Same message transmitted twice | No duplicated false alert or duplicate chart point by policy |
| Out-of-order data | Older reading arrives later | Sort by recorded_at; "latest" stays correct |
| Gap in history | Missing multiple sample intervals | Mark/represent missing data; no false continuity |
| API offline | Fetch rejects/503 | Friendly error + retry; not silently "all healthy" |
| Unauthorized access | 401/403 | No protected data shown; no secrets leaked |
| Unknown zone | 404 | Not-found state |
| Invalid range | End before start / enormous interval | Clear validation error |
| High-temp demo | Fresh valid simulated 29.1°C | Explainable warning with explicitly unvalidated 28°C test threshold |
| Rule disabled | Same values, disabled rule | No triggered high-temp insight |
| Device offline | Last heartbeat overdue | Device-health warning, not a fabricated live reading |
| Units mismatch | C vs F or EC unit mismatch | Normalize explicitly or reject, never compare silently |

### Example test cases for the intelligence evaluator

~~~python
from datetime import datetime, timezone

DEMO_NOW = datetime(2026, 11, 18, 14, 5, tzinfo=timezone.utc)

def test_high_temperature_demo():
    reading = {
        "value": 29.1,
        "unit": "C",
        "quality_flag": "valid",
        "recorded_at": "2026-11-18T14:04:00Z",
    }
    # Replace with actual import once evaluator is implemented:
    # insight = high_temp_insight(reading, now=DEMO_NOW)
    # assert insight["severity"] == "warning"
    # assert insight["code"] == "HIGH_TEMPERATURE_DEMO"

def test_stale_data_does_not_trigger_live_high_temp():
    reading = {
        "value": 35.0,
        "unit": "C",
        "quality_flag": "valid",
        "recorded_at": "2026-11-18T12:00:00Z",
    }
    # insight = high_temp_insight(reading, now=DEMO_NOW)
    # assert insight is None
    # A separate stale-sensor insight should be tested too.
~~~

These functions are **test outlines** and intentionally do not claim to be passing, runnable assertions. Once the evaluator exists, replace the comments with real imports/assertions, run pytest and include output in the PR.

## 6. Reviewer demonstration checklist

- [ ] From a clean clone, start the frontend and backend with documented commands.
- [ ] Select two synthetic zones; switch between their readings.
- [ ] Show the same sample sensor record in database query result, backend JSON and React chart.
- [ ] Show a high-temperature *demo insight*, with threshold provenance and no actuator command.
- [ ] Show an empty zone, stale device, invalid reading and failed API.
- [ ] Explain selected SQL JOIN, WHERE, ORDER BY and index usage.
- [ ] Show unauthorized API access fails without revealing data.
- [ ] Show unit tests for evaluator and API, and frontend build/typecheck.
- [ ] Confirm all dataset contents are synthetic and no production secrets are committed.

## 7. What to coordinate with other workstreams

- **Issue #11:** telemetry payload identifiers/units/timestamps.
- **Issue #13:** canonical devices/sensors/readings table design.
- **Issue #23:** time-series indexes/quality/retention.
- **Issue #16:** shared alert taxonomy, severity, thresholds and recommendations.
- **Issue #24:** automation control engine remains separate from this human-facing insight engine.
- **Issue #18:** include this vertical slice in integration testing.

**No premature AI claims:** A rule such as "fresh validated temperature >= demo threshold" is deterministic decision support, not a trained predictive model. More advanced anomaly detection should come only after measured data quality, a baseline and evaluation criteria are established.
