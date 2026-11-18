# QVFT — Farm Insights Dashboard: Step-by-Step Implementation Guide

**Start with [onboarding](fullstack-intelligence-onboarding.md)**; then use the [API contract, synthetic fixtures and test matrix](fullstack-intelligence-api-contract.md).

**Goal:** A read-only full-stack feature that combines a React/TypeScript interface, a FastAPI retrieval API, PostgreSQL/Supabase time-series queries, and an explainable farm-insight view. **Use synthetic data first.** All code paths below are suggested additions—not verified existing implementation.

## Task A — Frontend foundations (primary entry task; aligns with #14 and dashboard #4)

**Why:** The farm needs an operator-friendly interface that can show current measurements and historical trends. A dashboard that looks good but cannot handle invalid/stale data or failure is not complete.

### Steps

1. Read [Issue #14](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/14) and coordinate with anyone already setting up the frontend. Check if frontend/package.json exists. If yes, reuse it; if not, after agreement scaffold a React+TypeScript Vite app. See [onboarding](fullstack-intelligence-onboarding.md) for conditional setup.
2. Create a first static **Farm Insights** page with a header, zone selector, status indicators and latest-reading cards.
3. Define TypeScript types in \`frontend/src/types/farm.ts\` that match the sample API response in [the contract](fullstack-intelligence-api-contract.md).
4. Create local synthetic fixture data. Build the UI without depending on a backend being available.
5. Extract reusable components: \`SensorCard\`, \`HistoryChart\`, \`AlertList\`, optional \`ZoneSelector\`.
6. Display **units, measurement time and staleness**; differentiate "No readings" from "Sensor offline" and "API not responding".
7. Add a 24-hour history chart (Recharts or charting library approved by frontend owner). Use actual timestamps, tooltip details and appropriate units. Do not interpolate across large missing-data gaps without a visible indication.
8. Add responsive layout, descriptive labels, appropriate focus/keyboard behavior, color-independent alert messages and a reduced-motion-friendly loading state.
9. Demonstrate empty, loading, error, stale, and successful states with deterministic fixture variants.
10. Open a focused PR linked to #14 or an approved subtask under #4.

**Suggested TypeScript model (adapt to agreed backend contract):**

~~~typescript
export type QualityFlag = "valid" | "suspect" | "invalid" | "stale";

export type Reading = {
  sensor_id: string;
  sensor_type: string;
  value: number;
  unit: string;
  recorded_at: string; // UTC ISO 8601
  quality_flag: QualityFlag;
};

export type Insight = {
  id: string;
  zone_id: string;
  code: string;
  severity: "info" | "warning" | "critical";
  summary: string;
  explanation: string;
  recommended_action: string; // Human-facing advice, NOT actuator command
  generated_at: string;
  source_sensor_ids: string[];
  status: "active" | "resolved";
};
~~~

**Example React component:**

~~~tsx
import type { Reading } from "../types/farm";

export function SensorCard({ reading }: { reading: Reading | null }) {
  if (!reading) return <section aria-label="Sensor"><p>No reading available</p></section>;

  return (
    <section aria-label={reading.sensor_type}>
      <h2>{reading.sensor_type.replaceAll("_", " ")}</h2>
      <p>{reading.value} {reading.unit}</p>
      <small>Measured at: {new Date(reading.recorded_at).toLocaleString()}</small>
      <small>Quality: {reading.quality_flag}</small>
    </section>
  );
}
~~~

The minimal component above **does not** implement complete stale/error treatment; the finished page must. Values should be formatted appropriately for each sensor type; do not hard-code all variables as °C.

**Pass/acceptance:** \`npm run dev\` shows a functional UI from synthetic fixtures, type checking passes, pages work on narrow screens, and loading/error/stale states are demonstrable.

## Task B — SQL data retrieval (collaborate with Database owner #3/#13/#23)

**Why:** SQL is used to answer practical questions such as "What was the latest valid temperature for zone A?" or "How has humidity changed during the last day?" The member learns SQL without owning or changing another person's schema.

### Steps

1. Review the database team's schema for \`zones\`, \`devices\`, \`sensors\`, and \`sensor_readings\`. **Do not invent new production table/column names** without approval.
2. Ask the DB owner for a local development database and approved synthetic seed data. They should be able to reproduce it from migrations.
3. Learn \`SELECT\`, \`WHERE\`, \`JOIN\`, \`ORDER BY\`, \`LIMIT\`, \`GROUP BY\`, and \`COUNT\`/\`AVG\`.
4. Write **read-only** queries for latest valid readings for a sensor, sensor history over a window, and a zone's recent readings.
5. Think about freshness: latest **recorded** reading may still be too old for live monitoring. Filter quality flags and calculate age; don't silently treat unknowns as valid.
6. Check expected results against seeded data. Use a query plan (\`EXPLAIN ANALYZE\`) with the DB owner when necessary.
7. If SQL improvements/indexes are needed, propose them to #23's owner rather than pushing a unilateral migration.
8. Save the approved queries and their meaning in \`docs/farm-insights-sql.md\` or a folder agreed with the DB owner.

**Example (adjust to actual schema and seeded UUIDs):**

~~~sql
-- Latest valid reading for a single sensor
SELECT r.sensor_id, r.value, s.unit, r.recorded_at, r.quality_flag
FROM public.sensor_readings AS r
JOIN public.sensors AS s ON s.id = r.sensor_id
WHERE r.sensor_id = :sensor_id
  AND r.quality_flag = 'valid'
ORDER BY r.recorded_at DESC
LIMIT 1;

-- History in a specified time window (all timestamps UTC)
SELECT recorded_at, value
FROM public.sensor_readings
WHERE sensor_id = :sensor_id
  AND recorded_at >= :start_at
  AND recorded_at < :end_at
  AND quality_flag = 'valid'
ORDER BY recorded_at ASC;
~~~

\`:sensor_id\` and other colon parameters are *placeholders used by parameterized SQL clients*, not literals to paste into the Supabase SQL editor. For manually testing in an editor, replace them with **quoted seeded test IDs/time values**; in application code always use properly bound parameters.

**Pass/acceptance:** Show the results for synthetic data; explain join keys and how stale/invalid data are handled. No new privileged frontend DB connection is created.

## Task C — Read-only backend/API (collaborate with Backend owner #2/#12)

**Why:** The frontend should not contain SQL credentials or know how tables are internally structured. A backend translates an authorized request into a validated query and stable JSON format.

### Steps

1. Read Issue #12 and check for an existing FastAPI project; extend it rather than creating a second competing backend.
2. Agree an API contract with frontend and database owners; use [the proposal](fullstack-intelligence-api-contract.md) as a discussion draft.
3. Start with \`GET /health\`, \`GET /zones\`, \`GET /zones/{zone_id}/readings/latest\`, \`GET /sensors/{sensor_id}/history?from=...&to=...\`, and \`GET /zones/{zone_id}/insights\`. Match existing route conventions if any.
4. Define response models; validate UUIDs/date-range/timezone and limit history windows/count to avoid huge accidental queries.
5. For phase one, use an in-memory mock repository with synthetic data so the API works without a live database.
6. Swap the data adapter to authorized/parameterized SQL against local Supabase after the DB schema is ready. Keep privileged credentials strictly server-side.
7. Return appropriate status codes (e.g., 200 success, 400 invalid input, 401/403 auth failure, 404 unknown resource, 503 unavailable dependency) and consistent error messages.
8. Configure CORS only for known local dev frontend origins; do not allow unrestricted production origins with credentials.
9. Add unit/API tests using FastAPI TestClient and pytest. Test normal response, no data, invalid ID/window, unauthorized requests and downstream database timeout.
10. Document endpoints and example request/response payloads; PR to backend owner for review.

**Minimal learning example only:**

~~~python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/zones/demo-zone/readings/latest")
def demo_latest():
    return {
        "zone_id": "demo-zone",
        "readings": [
            {
                "sensor_id": "temp-demo",
                "sensor_type": "air_temperature",
                "value": 24.0,
                "unit": "C",
                "recorded_at": "2026-11-18T14:00:00Z",
                "quality_flag": "valid",
            }
        ],
    }
~~~

This is a **mock example** to learn FastAPI, not an authenticated production service or the final database-backed implementation.

**Pass/acceptance:** A teammate can run the API locally, view \`/docs\`, retrieve deterministic JSON, and show passing normal/error tests. The React app can fetch the mock route.

## Task D — Intelligence & Decision Support (align with assigned owner #6 and #16)

**Why:** Data visualization becomes more useful when the dashboard can explain what needs attention. Start with deterministic, auditable signals before ML.

**Scope split:** This member can implement and display *human-facing insights* and collaborate on definitions. The designated Intelligence owner should approve shared detection logic; the automation owner retains authority over physical control commands.

### Steps

1. Coordinate with Intelligence and Plant Science owners. Choose first signal definitions: e.g., high temperature, sensor stale/offline, and an unusual-but-valid variation from recent baseline.
2. Write a table for each rule: input sensors, validated units, duration/window, test threshold, severity, human recommendation, confidence/limitations and how to acknowledge/resolve.
3. Use **synthetic and explicitly unvalidated** values to test code. No crops should be declared unhealthy solely because a generic demo threshold was crossed.
4. Create a **pure Python function** that receives validated recent readings and returns an \`Insight\` object or \`None\`. No database writes or actuator calls inside the function.
5. Implement time freshness: if the newest temperature reading is too old, output a *stale sensor* insight rather than a *high temperature now* claim.
6. Implement debounce/duration and optional comparison to prior history; avoid triggering on a single isolated spike without a defensible policy.
7. Keep alert/insight behavior separate from an *actuation command*: the output is explanatory text for a human.
8. Have the FastAPI layer return insights to the frontend. Render severity, what happened, evidence/time and a recommendation.
9. Add edge-case tests: valid/suspect/invalid input, missing history, out-of-order timestamps, duplicates, stale data, threshold crossing, units mismatch, and resolution/recovery.
10. Write \`docs/farm-insights-rules.md\` with test data, limitations and Plant Science approvals needed.

**Example: explainable classification logic (illustrative test threshold, not crop guidance):**

~~~python
from datetime import datetime, timedelta, timezone

def high_temp_insight(reading: dict, now: datetime) -> dict | None:
    observed = datetime.fromisoformat(reading["recorded_at"].replace("Z", "+00:00"))
    fresh = timedelta(0) <= now - observed <= timedelta(minutes=5)
    if reading["quality_flag"] != "valid" or not fresh or reading["unit"] != "C":
        return None  # Evaluate stale/invalid sensor separately.
    if reading["value"] < 28.0:
        return None
    return {
        "code": "HIGH_TEMPERATURE_DEMO",
        "severity": "warning",
        "summary": "Simulated zone temperature exceeds demo threshold",
        "explanation": "The latest valid, fresh reading is at least 28°C.",
        "recommended_action": "Check sensor and growing conditions; confirm with Plant Science.",
    }
~~~

For actual policy, incorporate duration and hysteresis as agreed with the Intelligence owner; the small function shows only the classification concept. **28°C is a demo value**, not an approved crop-growing threshold.

**Pass/acceptance:** At least three deterministic insight cases with explainable outputs, comprehensive normal/invalid/stale tests, and no physical actuation code. #16 may own the canonical alert definitions—avoid redefining them independently.

## Task E — Full-stack integration and demo

**Why:** Full-stack learning is demonstrated by connecting all layers, not just making independent components.

### Steps

1. Run the React app and mock FastAPI on separate local ports; configure the approved API base URL using a non-secret frontend environment setting.
2. Implement a typed fetch function with abort/timeout/error handling in \`frontend/src/api/farmInsights.ts\`; do not put SQL or service-role credentials in the browser.
3. Connect the zone selector to latest-reading cards, history chart and insight list.
4. Load from local synthetic fixtures first; switch to the mock API; finally switch to authorized local Supabase behind FastAPI when available.
5. Explain "measured at" and local display timezone; prefer UTC in API responses.
6. Demonstrate four scenarios: normal readings, high-temp insight, stale/offline sensor, and backend failure.
7. Demonstrate SQL query and backend logs for a selected test reading, and show where the same reading is displayed in React.
8. Run frontend typecheck/build, FastAPI tests and manual accessibility/error checks; fix failures.
9. Provide clear \`README\` setup and a short screen recording or meeting demo.
10. Open a PR describing dependencies and exactly how someone can reproduce the demo.

**Demo acceptance:** A reviewer can trace a seeded measurement through SQL -> FastAPI -> React chart/card, see a human-readable insight, and inspect error/stale states without proprietary credentials or physical hardware.

---

## Team lead review rubric (score 0/1 for each)

| Criterion | What to check |
| --- | --- |
| Functional UI | Works from known synthetic data |
| Correct data interpretation | Units, timezone, quality, freshness displayed |
| SQL understanding | Can explain JOIN and latest/history query |
| API knowledge | Can describe endpoints, response types, errors |
| Intelligence correctness | Rules transparent; invalid/stale data do not cause misleading alerts |
| Security | No privileged credentials or raw SQL in frontend |
| Testing | At least normal, empty, stale, unauthorized, and API failure cases |
| Maintainability | Setup/docs, reusable components, PR review workflow |

**Suggested definition of done:** All eight criteria satisfied, with tests or reproducible demonstrations. Begin additional ML or algorithmic prediction only after the team has reliable data, validated baselines, and an appropriate evaluation design.

## Future stretch tasks (not in the first assignment)

- Rolling averages/diurnal baselines and clear uncertainty flags.
- Multi-sensor comparisons (e.g., temperature and humidity), only with Plant Science validation.
- Device health analytics (time since last reading, communication dropouts).
- Alert acknowledgement and resolution UI, with backend-authenticated role permissions.
- Export an authorized time-window CSV for experiments.
- After enough labeled examples exist: anomaly detection benchmark, precision/recall, false-alarm study and model drift monitoring.
