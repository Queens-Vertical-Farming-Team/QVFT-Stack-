# Intelligence & Full-Stack Development Guide

This guide is the working playbook for QVFT members contributing to **Intelligence & Decision Support**, especially members who also want experience with **SQL, backend APIs, React, TypeScript, and full-stack development**.

The goal is not to build "AI" for the sake of saying QVFT uses AI.

The goal is to build a system that can:

```text
Farm data
   ↓
Understand current state
   ↓
Detect useful conditions
   ↓
Explain what is happening
   ↓
Recommend an action
   ↓
Show that recommendation clearly to a human
```

The first milestone is **decision support**, not autonomous control.

---

## 1. Where Intelligence fits in the Stack

The planned architecture is:

```text
Sensors / Cameras
       ↓
ESP32 / Arduino
       ↓
FastAPI Backend
       ↓
PostgreSQL / Supabase
       ↓
Intelligence & Decision Support
       ↓
FastAPI
       ↓
React + TypeScript Dashboard
```

Later, Intelligence may recommend actions to the Automation & Control system, but it should not directly control physical hardware.

That separation is intentional:

- **Intelligence** decides what the data means.
- **Automation & Control** decides whether a physical action is allowed and safe.
- **Firmware** performs the actual hardware action.
- **Database/Supabase** stores the data and history.
- **Frontend** explains the system state to the user.

---

# 2. Recommended issue order

Work through the Intelligence tasks in this order:

```text
#16  Define first rule-based farm alerts
 ↓
#29  Build Intelligence data-access queries
 ↓
#30  Implement first Intelligence decision-support engine
 ↓
#31  Expose Intelligence results through FastAPI
 ↓
#32  Build Intelligence & Decision Support dashboard view
 ↓
#33  Add first sensor anomaly-detection prototype
 ↓
#34  Evaluate predictive and ML opportunities
```

Do not skip directly to #33 or #34 unless the data pipeline and decision-support baseline are already working.

---

# 3. Phase 1 — Learn how data becomes a decision

Related Issue: **#16**

Before writing ML code, understand what the system is deciding.

Start by listing the measurements the Intelligence system may receive:

- air temperature
- relative humidity
- pH
- EC
- water temperature
- reservoir level
- light level
- device status
- sensor timestamps
- data-quality flags
- camera/vision outputs later

For each measurement, document:

| Field | Meaning |
|---|---|
| measurement | What is being measured? |
| unit | °C, %, pH, mS/cm, etc. |
| source | Which sensor/device provides it? |
| update frequency | How often should it arrive? |
| valid range | What values are physically believable? |
| plant threshold | Is there a validated plant-science threshold? |
| owner | Who validates this value? |

If a threshold is not known, write:

```text
TBD — Plant Science validation required
```

Do not silently invent agronomic thresholds.

## Rule structure

Each first-generation rule should define:

- inputs
- condition
- duration
- severity
- explanation
- recommendation
- conditions that suppress the rule
- threshold owner/source

Example:

```text
Rule:
High air temperature

Condition:
temperature > configured_high_threshold

Duration:
condition must remain true for 5 minutes

Severity:
warning

Explanation:
Air temperature has remained above the configured limit.

Recommendation:
Inspect ventilation and verify fan availability.
```

## Important concepts

### Debounce / duration

One bad sample should not always create an alert.

Example:

```text
temperature > threshold for 5 minutes
```

### Hysteresis

The trigger and clear thresholds may be different so the system does not rapidly switch states.

Example:

```text
trigger alert above 28 °C
clear alert below 26 °C
```

The actual values must be validated.

### Data freshness

A 20-minute-old sensor value should not be treated the same as a 5-second-old value.

Suggested freshness states:

```text
good
suspect
stale
invalid
```

### Duplicate suppression

Do not create the same alert every few seconds.

---

# 4. Phase 2 — SQL and Supabase data access

Related Issue: **#29**

This phase teaches the database side of full-stack development.

The Intelligence layer needs to ask questions such as:

- What is the newest temperature reading?
- What were the last 60 minutes of humidity?
- What are the latest readings for Growth Zone 1?
- Which devices are offline?
- Which alerts are still unresolved?
- Is this sensor data fresh enough to trust?

## SQL concepts to learn

At minimum:

```text
SELECT
FROM
WHERE
ORDER BY
LIMIT
JOIN
GROUP BY
COUNT
AVG
MIN
MAX
```

Example — latest reading:

```sql
SELECT value, recorded_at
FROM sensor_readings
WHERE sensor_id = 'TEMP-01'
ORDER BY recorded_at DESC
LIMIT 1;
```

Be able to explain this in plain language:

> Find the readings for TEMP-01, sort newest first, and return only the newest one.

Example — historical readings:

```sql
SELECT value, recorded_at
FROM sensor_readings
WHERE sensor_id = 'TEMP-01'
  AND recorded_at >= NOW() - INTERVAL '1 hour'
ORDER BY recorded_at;
```

Example — join sensors to devices:

```sql
SELECT
  s.id,
  s.sensor_type,
  d.name AS device_name,
  d.zone_id
FROM sensors s
JOIN devices d ON s.device_id = d.id;
```

## Learning checkpoint

Before the PR is approved, the contributor should be able to explain:

- what a primary key is
- what a foreign key is
- why JOIN exists
- why timestamps matter
- what an index does conceptually
- why database queries should not be duplicated randomly throughout the codebase

---

# 5. Phase 3 — Build the decision-support engine

Related Issue: **#30**

Create a dedicated Intelligence module rather than putting decision logic directly inside the API routes.

Suggested structure:

```text
intelligence/
  README.md
  rules/
    README.md
  models/
  service.py
  tests/
    README.md
```

A useful separation is:

```text
database/query layer
        ↓
normalized farm state
        ↓
intelligence service
        ↓
rule evaluation
        ↓
decision object
```

## Suggested decision output

```json
{
  "decision_id": "decision-001",
  "zone_id": "growth-zone-01",
  "severity": "warning",
  "title": "High air temperature",
  "explanation": "Air temperature has remained above the configured threshold for 10 minutes.",
  "recommendation": "Inspect ventilation and verify fan availability.",
  "data_freshness": "good",
  "supporting_readings": [
    {
      "sensor_id": "TEMP-01",
      "value": 29.1,
      "unit": "C"
    }
  ]
}
```

## Design rule

Every recommendation should answer:

1. What happened?
2. Why does the system believe it?
3. What data supports it?
4. What should the operator consider doing?
5. How fresh/reliable is the data?

## Do not directly control hardware

The Intelligence layer should produce:

```text
Recommendation:
Inspect ventilation / consider fan operation.
```

not:

```text
fan = ON
```

That physical decision belongs to Automation & Control.

---

# 6. Phase 4 — Expose Intelligence through FastAPI

Related Issue: **#31**

This phase teaches backend/API development.

Possible endpoints:

```text
GET /intelligence/status
GET /intelligence/decisions
GET /intelligence/decisions/latest
GET /intelligence/zones/{zone_id}
```

Exact endpoint names may evolve.

## What to learn

- FastAPI routing
- GET requests
- response models
- Pydantic
- JSON
- HTTP status codes
- validation
- error handling
- separation of routing and business logic

Bad pattern:

```text
API route
  ├── SQL query
  ├── decision logic
  ├── formatting
  └── response
```

Preferred pattern:

```text
API route
   ↓
service
   ↓
intelligence logic
   ↓
data/query layer
```

The route should stay relatively thin.

## Example response

```json
{
  "zone_id": "growth-zone-01",
  "status": "warning",
  "active_decisions": 2,
  "data_freshness": "good"
}
```

## Required testing

Test:

- valid request
- unknown zone
- no available data
- stale data
- backend/database failure
- valid response schema

---

# 7. Phase 5 — React + TypeScript dashboard

Related Issue: **#32**

This phase gives the contributor frontend/full-stack experience.

The dashboard should not just show numbers.

It should explain what the system thinks those numbers mean.

Example:

```text
Growth Zone 1

Temperature
29.1 °C

WARNING
High air temperature

Why?
Temperature has remained above the configured threshold
for 10 minutes.

Recommendation
Inspect ventilation.

Supporting reading
TEMP-01: 29.1 °C

Data freshness
Updated 12 seconds ago
```

## Frontend concepts to learn

- React components
- TypeScript interfaces/types
- API calls
- fetch/async logic
- component state
- loading states
- error states
- empty states
- reusable components
- charts
- mapping JSON into UI

Suggested component ideas:

```text
IntelligencePage
DecisionCard
SeverityBadge
RecommendationPanel
SupportingReadings
DataFreshnessBadge
ZoneSelector
DecisionHistory
```

## Required UI states

Every API-connected component should consider:

```text
loading
success
empty
error
stale
```

Do not assume the API always works.

## Important UX distinction

Clearly separate:

```text
Measured:
Temperature = 29.1 °C

System assessment:
High-temperature warning

Recommendation:
Inspect ventilation
```

Never make a recommendation look like a measured fact.

---

# 8. Phase 6 — Anomaly detection

Related Issue: **#33**

After the rule-based system works, start exploring unusual behavior that cannot be described by one fixed threshold.

Example:

```text
Normal temperature change:
~0.5 °C/hour

Observed:
+6 °C in 3 minutes
```

This may represent:

- real environmental change
- sensor failure
- disconnected probe
- bad calibration
- communication/data problem

## Start simple

Do not begin with a neural network.

Evaluate:

- rolling mean
- rolling standard deviation
- rate-of-change
- median / MAD
- comparison between similar sensors
- persistence rules

## Evaluation

For every anomaly system, ask:

- What is a true positive?
- What is a false positive?
- What is a false negative?
- How much history is needed?
- How will broken sensors be separated from real farm anomalies?

Create synthetic test data before claiming the detector works.

---

# 9. Phase 7 — Predictive / ML work

Related Issue: **#34**

ML comes after the data pipeline and baseline decision system work.

Potential future questions:

- Can we predict environmental conditions 30 minutes ahead?
- Can we predict sensor/device failure?
- Can we estimate irrigation need?
- Can we estimate plant growth?
- Can vision + sensor data identify plant stress?
- Can we forecast trends that actually change an operator decision?

For every candidate model, document:

```text
Problem:
What are we predicting?

Target:
What output should the model produce?

Inputs:
What data is required?

Baseline:
What simple method are we trying to beat?

Metric:
How will success be measured?

Operational value:
What decision changes if the prediction is correct?
```

Do not train a model just so the project can say it uses AI.

---

# 10. Suggested development workflow

For every task:

```bash
git switch main
git pull
git switch -c feature/intelligence-short-description
```

Do the work.

Then:

```bash
git status
git add .
git commit -m "feat: describe the intelligence change"
git push -u origin feature/intelligence-short-description
```

Open a Pull Request into `main`.

Do not push feature work directly to `main`.

---

# 11. What should go in the Pull Request?

Every Intelligence PR should explain:

## What changed?

What feature or logic was added?

## Why?

What problem does it solve?

## Data flow

Where does the input come from and where does the output go?

## Testing

What test data/scenarios were used?

## Learning

Answer briefly:

- What did you learn?
- What did AI help with?
- What did you personally verify?
- What could break this feature?
- What assumptions are still unvalidated?

## Screenshots/examples

For frontend work, include screenshots where useful.

For API work, include example requests/responses.

For SQL work, include representative queries/results using synthetic data.

---

# 12. AI use policy for learning

AI tools are allowed and encouraged.

The standard is not:

> Did you write every line manually?

The standard is:

> Do you understand the system you are responsible for?

Before a PR is approved, the contributor should be able to:

- explain their code
- explain their SQL query
- explain the API request/response
- explain why a rule fired
- identify failure cases
- change the implementation when requirements change
- test the implementation independently

A useful review question is:

> What would make this system wrong?

If the contributor cannot answer that, the work is not ready to merge.

---

# 13. First practical milestone

The first major Intelligence milestone should demonstrate:

```text
Synthetic sensor data
        ↓
Supabase/PostgreSQL
        ↓
SQL query layer
        ↓
Decision-support engine
        ↓
FastAPI endpoint
        ↓
React dashboard
```

Example:

```text
TEMP-01 = 29.1 °C
        ↓
database stores reading
        ↓
query retrieves latest/history
        ↓
rule evaluates persistent high temperature
        ↓
decision created
        ↓
API returns decision
        ↓
dashboard displays:
warning + why + recommendation + supporting data
```

If this entire chain works, the contributor has built a real full-stack QVFT feature.

---

# 14. Definition of done for the first Intelligence milestone

- [ ] database queries are working
- [ ] contributor can explain the SQL
- [ ] at least 5 deterministic rules exist
- [ ] stale/invalid data is handled
- [ ] decisions contain explanations
- [ ] decisions include supporting readings
- [ ] FastAPI exposes the decisions
- [ ] React dashboard displays them
- [ ] loading/error/empty states exist
- [ ] tests cover normal and failure cases
- [ ] no Intelligence code directly controls hardware
- [ ] assumptions requiring Plant Science validation are documented
- [ ] contributor can explain the full data path end-to-end

---

# 15. Where to ask questions

Use the related GitHub Issue for task-specific discussion.

If a design decision affects multiple workstreams, document it in the Issue/PR and coordinate with the relevant owners:

- Database & Supabase
- Backend & Communications
- Automation & Control
- Dashboard
- Plant Science
- Sensors & Firmware

The goal is to build one connected Stack, not isolated features.
