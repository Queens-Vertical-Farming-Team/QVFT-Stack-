# QVFT Stack — Intelligence & Full-Stack Step-by-Step Guide

**Companion:** [Implementation roadmap](implementation-roadmap.md) | [Database & Supabase guide](database-implementation-guide.md) | [Automation & Control guide](automation-control-implementation-guide.md)

**Scope:** Implementation assignments for [#16](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/16), [#29](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/29), [#30](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/30), [#31](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/31), [#32](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/32), [#33](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/33), and [#34](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/34). These are implementation **instructions and examples**, not claims that the features are already complete.

> **Boundary:** Intelligence & Decision Support interprets data and produces alerts, explanations, recommendations and predictions. It does **not** directly bypass Automation & Control to operate physical equipment. Any future control action must still pass the control layer's modes, interlocks, authorization and fail-safe checks.

## Before starting

Understand the intended data path:

~~~text
Farm sensors / cameras
        ↓
ESP32 / Arduino
        ↓
FastAPI validation
        ↓
Supabase / PostgreSQL
        ↓
Intelligence & Decision Support
        ↓
FastAPI response
        ↓
React + TypeScript dashboard
~~~

The first milestone is not "AI controls the farm." The first milestone is:

~~~text
validated farm data
  -> explainable rule / assessment
  -> structured decision
  -> recommendation
  -> API response
  -> dashboard display with supporting evidence
~~~

Review:

- [architecture.md](architecture.md)
- [Database & Supabase guide](database-implementation-guide.md)
- [Issue #13 schema](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/13)
- [Issue #12 FastAPI bootstrap](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/12)
- [Issue #14 React dashboard](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/14)
- [Issue #16 rule-based farm alerts](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/16)

### Recommended setup

From a clean checkout:

~~~powershell
git clone https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-.git
cd QVFT-Stack-
git switch main
git pull
git switch -c feature/issue-16-intelligence-rules
code .
~~~

For Python work, use a virtual environment:

~~~powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
~~~

Install project dependencies only after checking whether the backend already has a package/requirements file. Avoid creating competing dependency files without coordinating with the backend owner.

For frontend work, use the existing React/TypeScript setup created by #14. Do not create a second unrelated frontend application if one already exists.

---

## #16 — Define first rule-based farm alerts

### Why this exists

Raw sensor values are not yet useful decisions. A value such as 29.1°C only becomes useful when the system knows:

- what was measured
- where it was measured
- whether the reading is fresh
- whether the sensor is valid
- whether the condition persisted
- whether the threshold was validated
- what the operator should do next

The goal is to turn raw measurements into **transparent, explainable farm-state assessments**.

### Work in order

1. **Inventory candidate inputs.** Review the planned schema and list the measurements available or expected:
   - air temperature
   - relative humidity
   - pH
   - EC
   - water temperature
   - reservoir level
   - light
   - device status
   - timestamps
   - quality/freshness metadata
2. **Document units and ownership.** For every variable, define unit, source sensor/device and who validates meaningful plant thresholds.
3. **Do not invent crop thresholds.** If Plant Science has not approved a number, label it clearly:
   `TBD — Plant Science validation required`.
4. **Define at least five candidate decision rules.**
5. **Define severity levels**, e.g. info / warning / critical.
6. **Define data freshness states**, e.g. good / suspect / stale / invalid.
7. **Add debounce/duration**, so one transient spike does not create a warning when persistence is required.
8. **Add hysteresis where appropriate**, so a warning does not rapidly flip on/off near one threshold.
9. **Define duplicate suppression/cooldown**, so the same condition does not create alert spam.
10. **Create synthetic test cases** and document expected outcomes.

Example rule specification:

~~~text
Rule ID:
air_temperature_high

Input:
air_temperature

Trigger:
temperature > configured high threshold

Persistence:
must remain true for 5 minutes

Clear:
temperature < configured clear threshold

Severity:
warning

Suppress when:
sensor data is stale / invalid

Explanation:
Air temperature has remained above the configured threshold.

Recommendation:
Inspect ventilation and verify fan availability.

Threshold owner:
Plant Science
~~~

### Suggested decision object

~~~json
{
  "decision_id": "decision-0001",
  "zone_id": "growth-zone-01",
  "rule_id": "air_temperature_high",
  "severity": "warning",
  "title": "High air temperature",
  "explanation": "Air temperature has remained above the configured threshold for 5 minutes.",
  "recommendation": "Inspect ventilation and verify fan availability.",
  "data_freshness": "good",
  "supporting_readings": [
    {
      "sensor_id": "TEMP-01",
      "value": 29.1,
      "unit": "C",
      "recorded_at": "2026-11-18T17:00:00Z"
    }
  ]
}
~~~

### Required synthetic scenarios

1. Normal data -> no alert.
2. One brief threshold excursion -> no alert if persistence is required.
3. Persistent threshold excursion -> alert.
4. Stale data -> decision blocked or explicitly downgraded.
5. Invalid reading -> decision blocked.
6. Condition clears -> alert resolves.
7. Repeated same condition -> duplicate alert spam suppressed.

### Design questions

- Does the rule represent a plant condition or a sensor fault?
- Who owns the threshold?
- What happens when two sensors disagree?
- Does a stale reading create a warning about the farm or about data quality?
- Should a recommendation be shown if evidence is incomplete?

**Acceptance demonstration:** A reviewer can read the rule definitions without source code and understand when each rule triggers, why it triggers, how bad data is handled and what recommendation is produced.

---

## #29 — Build Intelligence data-access queries

### Why this exists

The Intelligence layer cannot reason from data it cannot retrieve reliably. This issue teaches the SQL/data-access side of full-stack development and creates reusable queries for farm state.

### Step by step

1. **Read the current schema** from #13 before writing queries.
2. **Identify the Intelligence query needs**:
   - latest reading for one sensor
   - latest readings for all sensors in one zone
   - readings over a time window
   - average/min/max over a window
   - unresolved alerts
   - device last-seen state
   - sensor data freshness
3. **Write SQL queries first** so the contributor understands the relational model.
4. **Use JOINs** rather than copying zone/device/sensor names into every table.
5. **Explain each query in plain language.**
6. **Test on synthetic data.**
7. **Check performance** for queries expected to run frequently.
8. **Coordinate indexes** with #23 rather than adding random indexes without evidence.
9. **Create a small query/data-access layer** instead of scattering raw queries through API routes.
10. **Document expected result shapes** for backend use.

### SQL examples

Latest reading:

~~~sql
SELECT r.sensor_id, r.value, r.recorded_at
FROM public.sensor_readings r
WHERE r.sensor_id = 'REPLACE_WITH_SEEDED_SENSOR_UUID'
ORDER BY r.recorded_at DESC
LIMIT 1;
~~~

One-hour history:

~~~sql
SELECT r.value, r.recorded_at
FROM public.sensor_readings r
WHERE r.sensor_id = 'REPLACE_WITH_SEEDED_SENSOR_UUID'
  AND r.recorded_at >= now() - interval '1 hour'
ORDER BY r.recorded_at;
~~~

Join readings to physical context:

~~~sql
SELECT
    z.name AS zone,
    d.name AS device,
    s.sensor_type,
    s.unit,
    r.value,
    r.recorded_at
FROM public.sensor_readings r
JOIN public.sensors s ON s.id = r.sensor_id
JOIN public.devices d ON d.id = s.device_id
JOIN public.zones z ON z.id = d.zone_id
ORDER BY r.recorded_at DESC
LIMIT 50;
~~~

Aggregate example:

~~~sql
SELECT
    sensor_id,
    avg(value) AS avg_value,
    min(value) AS min_value,
    max(value) AS max_value
FROM public.sensor_readings
WHERE recorded_at >= now() - interval '24 hours'
GROUP BY sensor_id;
~~~

### Learning checkpoint

The contributor should be able to explain:

- SELECT
- WHERE
- ORDER BY
- LIMIT
- JOIN
- GROUP BY
- AVG / MIN / MAX
- primary key
- foreign key
- timestamp/time-window query
- why an index can improve frequent lookups

**Acceptance demonstration:** A reviewer provides a seeded zone/sensor and the contributor retrieves latest, historical and aggregate information and explains the SQL without relying on generated text.

---

## #30 — Implement the first Intelligence decision-support engine

### Why this exists

This is the first real Intelligence implementation. It converts validated data into structured, explainable decisions.

Keep the logic separate from database code, API routes and UI components.

### Recommended code organization

Adapt to the backend structure chosen by #12:

~~~text
backend/
  intelligence/
    __init__.py
    models.py
    rules.py
    service.py
  tests/
    test_intelligence_rules.py
    test_intelligence_service.py
~~~

or, if the team chooses a top-level intelligence package:

~~~text
intelligence/
  rules/
  models/
  service.py
  tests/
~~~

Use one convention consistently rather than duplicating both.

### Step by step

1. **Define typed input models** for normalized farm-state values.
2. **Define a typed decision output model.**
3. **Implement the rules from #16** as deterministic logic.
4. **Check freshness before evaluation.**
5. **Reject or downgrade invalid/suspect data** according to documented policy.
6. **Attach supporting readings** to each decision.
7. **Keep recommendations human-readable.**
8. **Record the rule ID/version** that produced the decision.
9. **Write unit tests** for every normal and edge case.
10. **Keep hardware actuation outside this module.**

Illustrative Python shape:

~~~python
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Reading:
    sensor_id: str
    value: float
    unit: str
    recorded_at: datetime
    quality: str

@dataclass
class Decision:
    rule_id: str
    severity: str
    title: str
    explanation: str
    recommendation: str
    supporting_readings: list[Reading]
~~~

Illustrative rule function:

~~~python
def evaluate_high_temperature(readings, threshold):
    fresh_valid = [
        r for r in readings
        if r.quality == "valid"
    ]

    if not fresh_valid:
        return None

    latest = fresh_valid[-1]

    if latest.value <= threshold:
        return None

    return Decision(
        rule_id="air_temperature_high",
        severity="warning",
        title="High air temperature",
        explanation="Air temperature is above the configured threshold.",
        recommendation="Inspect ventilation and verify fan availability.",
        supporting_readings=[latest],
    )
~~~

This is intentionally incomplete. Add time persistence, freshness, threshold ownership/configuration and tests before considering the rule finished.

### Required tests

- normal condition
- threshold condition
- stale input
- invalid input
- missing sensor
- duplicate/repeated evaluation
- boundary value exactly at threshold
- clear/recovery condition

**Acceptance demonstration:** Run the test suite and show at least five rules producing deterministic decisions from synthetic inputs, including stale/invalid-data cases.

---

## #31 — Expose Intelligence results through FastAPI

### Why this exists

The frontend should not import Python decision logic directly. The backend exposes a stable API contract that hides implementation details.

### Step by step

1. **Read the existing FastAPI structure** from #12.
2. **Create response models** for decisions and zone status.
3. **Add thin route handlers** that call Intelligence services.
4. **Do not place SQL and decision logic directly inside route functions.**
5. **Validate route parameters.**
6. **Define error responses** for missing zone/data/backend failures.
7. **Add API tests.**
8. **Document example responses.**
9. **Do not expose privileged database credentials.**
10. **Keep recommendation output distinct from actuator commands.**

Possible routes:

~~~text
GET /intelligence/status
GET /intelligence/decisions
GET /intelligence/decisions/latest
GET /intelligence/zones/{zone_id}
~~~

Illustrative response:

~~~json
{
  "zone_id": "growth-zone-01",
  "status": "warning",
  "data_freshness": "good",
  "active_decisions": [
    {
      "severity": "warning",
      "title": "High air temperature",
      "explanation": "Air temperature is above the configured threshold.",
      "recommendation": "Inspect ventilation.",
      "created_at": "2026-11-18T17:05:00Z"
    }
  ]
}
~~~

### Required API cases

- valid zone
- unknown zone
- no readings
- stale readings
- backend/database error
- no active decisions

**Acceptance demonstration:** Start FastAPI locally, make example requests, show expected JSON and tests, and prove the route calls the Intelligence service instead of duplicating rule logic.

---

## #32 — Build Intelligence & Decision Support dashboard view

### Why this exists

A useful decision-support system must explain its conclusions to a human operator. The dashboard is where raw data, system assessment and recommendation come together.

### Step by step

1. **Use the existing React + TypeScript app from #14.**
2. **Define TypeScript interfaces** matching the FastAPI response.
3. **Fetch Intelligence data** from the backend API.
4. **Build reusable components** rather than one giant page.
5. **Handle loading, success, empty, error and stale states.**
6. **Display supporting readings.**
7. **Display why the rule fired.**
8. **Display the recommendation separately from measured facts.**
9. **Add zone/severity filtering only after the basic view works.**
10. **Add screenshots and manual test notes to the PR.**

Suggested components:

~~~text
IntelligencePage
DecisionCard
SeverityBadge
RecommendationPanel
SupportingReadings
DataFreshnessBadge
ZoneSelector
DecisionHistory
~~~

Illustrative TypeScript interface:

~~~typescript
export interface IntelligenceDecision {
  rule_id: string;
  zone_id: string;
  severity: "info" | "warning" | "critical";
  title: string;
  explanation: string;
  recommendation: string;
  data_freshness: "good" | "suspect" | "stale" | "invalid";
  created_at: string;
}
~~~

### UX requirement

Show the difference between fact and inference:

~~~text
Measured:
Temperature = 29.1 °C

System assessment:
High-temperature warning

Why:
Temperature has remained above the configured threshold.

Recommendation:
Inspect ventilation.
~~~

Never style a recommendation as though it were a directly measured sensor value.

### Required UI states

- loading
- data available
- no active decision
- stale data
- backend error
- unknown zone

**Acceptance demonstration:** Run the dashboard locally, fetch data from the FastAPI endpoint, show at least one decision card and demonstrate loading/error/no-alert behavior.

---

## #33 — Add first sensor anomaly-detection prototype

### Why this exists

Fixed thresholds cannot detect every unusual pattern. Anomaly detection asks whether the current behavior differs from a normal baseline.

Start with interpretable statistical methods before complex ML.

### Candidate baseline methods

- rolling mean
- rolling standard deviation
- rolling median / MAD
- rate-of-change thresholds
- comparison between redundant sensors
- persistence rules

Example rate-of-change question:

~~~text
Normal:
temperature changes about 0.5 °C/hour

Observed:
temperature rises 6 °C in 3 minutes

Possible explanations:
- real environmental event
- sensor disconnection
- calibration fault
- bad payload
~~~

### Step by step

1. **Choose one sensor signal** and define what "normal" means.
2. **Create a synthetic dataset** with normal and injected abnormal behavior.
3. **Implement one simple baseline detector.**
4. **Label expected anomalies in the synthetic data.**
5. **Measure true positives, false positives and false negatives.**
6. **Compare the anomaly detector against the threshold-rule baseline.**
7. **Document how missing/stale data is handled.**
8. **Document how sensor failure differs from farm-state anomaly.**
9. **Keep the prototype offline/synthetic before using live decisions.**
10. **Do not claim accuracy without measured evaluation.**

Illustrative simple detector:

~~~python
rolling_mean = series.rolling(window=20).mean()
rolling_std = series.rolling(window=20).std()

upper = rolling_mean + 3 * rolling_std
lower = rolling_mean - 3 * rolling_std

anomaly = (series > upper) | (series < lower)
~~~

This is an experiment, not a universal production solution.

### Evaluation table

| Case | Expected |
| --- | --- |
| Stable normal signal | Mostly no anomalies |
| Injected spike | Detected |
| Slow legitimate drift | Evaluate carefully |
| Missing data | Not treated as ordinary valid reading |
| Sensor flatlines | Flag as possible sensor fault |
| Frequent false positives | Tune/rethink method |

**Acceptance demonstration:** Provide synthetic data, detector output, measured false/true positives, at least one failure case and an explanation of whether the detector adds value beyond existing rules.

---

## #34 — Evaluate predictive and ML opportunities

### Why this exists

ML should solve an operational problem and outperform a meaningful baseline. It should not exist only so the team can say it "uses AI."

### Candidate problems

Evaluate at least three:

- environmental trend prediction
- device/sensor failure prediction
- irrigation need prediction
- plant growth estimation
- plant stress detection
- camera + sensor data fusion
- anomaly classification

### For each candidate document

1. **Problem:** What decision are we trying to improve?
2. **Target:** What exactly would the model predict?
3. **Inputs:** Which sensors/images/history are required?
4. **Data availability:** Do we actually have enough quality data?
5. **Baseline:** What simple non-ML method must the model beat?
6. **Metric:** MAE, RMSE, precision, recall, F1, etc., as appropriate.
7. **Operational value:** What would the farm team do differently if the prediction is good?
8. **Failure cost:** What happens when the model is wrong?
9. **Explainability:** Does the operator need to understand why?
10. **Recommendation:** Build now, collect more data first, or do not pursue.

Example evaluation:

~~~text
Candidate:
30-minute temperature forecasting

Target:
zone temperature 30 minutes in the future

Inputs:
recent temperature, humidity, light status, fan status, outside conditions if available

Baseline:
current temperature / simple moving trend

Metric:
MAE in °C

Operational value:
early warning before temperature crosses a validated limit

Main risk:
insufficient historical data across operating conditions
~~~

### If a prototype is approved

Use a proper train/validation/test split. Avoid leaking future data into training. Compare against the documented baseline. Record dataset version, features, metrics and known limitations.

**Acceptance demonstration:** Present at least three candidate ML problems, reject weak ones honestly, and recommend one future prototype only if the data and operational value justify it.

---

## Suggested first integrated Intelligence demonstration

Use synthetic values until the pipeline is stable.

~~~text
1. Seed TEMP-01 readings in Supabase.
2. Query the latest/history through the data-access layer.
3. Evaluate a demo high-temperature rule.
4. Produce a structured decision with supporting readings.
5. Expose it through GET /intelligence/decisions/latest.
6. Fetch it from React.
7. Display:
   - measured value
   - system assessment
   - explanation
   - recommendation
   - freshness
8. Demonstrate stale-data behavior.
9. Demonstrate no-alert behavior.
~~~

Any numeric thresholds used only for the demonstration must be clearly labelled illustrative unless validated by Plant Science.

---

## Intelligence PR checklist

- [ ] Issue linked and branch is focused on one task.
- [ ] Contributor can explain the code/query/API/UI they submit.
- [ ] SQL/data queries use the agreed schema and synthetic test data.
- [ ] No privileged Supabase credentials appear in frontend code.
- [ ] Decision rules document threshold ownership/validation.
- [ ] Stale, invalid, missing and boundary data are tested.
- [ ] Decisions include explanation and supporting evidence.
- [ ] Recommendations are separated from measured facts.
- [ ] Intelligence code does not directly bypass Automation & Control.
- [ ] API responses are typed/validated and failure states tested.
- [ ] Frontend includes loading/error/empty/stale states.
- [ ] Anomaly/ML claims include measured evaluation rather than unsupported accuracy claims.
- [ ] AI-assisted code is understood and personally verified.
- [ ] Another Stack member reviews before merge.

## Learning review questions

Before approving a PR, the reviewer may ask:

- What did you learn?
- What did AI help you with?
- What did you personally verify?
- Where does this data come from?
- Why is this SQL query correct?
- Why did this rule fire?
- What happens with stale data?
- What happens if the database/API is unavailable?
- What would cause a false positive?
- What would make this implementation wrong?
- Can you change one requirement without asking AI where the relevant code lives?

The objective is to use AI to accelerate engineering, not replace understanding.
