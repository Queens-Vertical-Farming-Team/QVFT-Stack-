# QVFT Stack — Full-Stack & Intelligence Contributor Onboarding

> **Who this is for:** A contributor interested in frontend development, SQL/PostgreSQL/Supabase, and Intelligence & Decision Support, with a goal of learning practical full-stack engineering.
>
> **Status:** This is an implementation and learning guide. It does **not** mean the referenced features, backend APIs, database tables, or frontend app already exist.
>
> **Recommended first project:** **Farm Insights Dashboard** — a read-only temperature/humidity monitoring view with historical charts and explainable alert insights using **synthetic data first**.

## Start here

1. Read [this onboarding guide](fullstack-intelligence-onboarding.md).
2. Follow [the step-by-step feature guide](fullstack-intelligence-feature-guide.md).
3. Use [the proposed API/data contract and test examples](fullstack-intelligence-api-contract.md).
4. Read the [existing QVFT architecture](architecture.md), [frontend README](../frontend/README.md), [backend README](../backend/README.md), and [database README](../database/README.md).
5. Review existing GitHub issues before claiming work: [#4 dashboard epic](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/4), [#14 frontend bootstrap](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/14), [#6 intelligence epic](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/6), [#16 alerts](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/16), [#3 database epic](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/3), and [#12 FastAPI bootstrap](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/12).

**Coordination is essential:** The Intelligence epic #6 and Database epic #3 have existing assigned owners as of this guide. **Do not take over their work or change schema without consulting them.** Issues #14 and #16 were open/unassigned when reviewed; recheck ownership before claiming. The proposed feature integrates with these issues, but is not itself a new GitHub issue until the lead creates/approves one.

## What does "full stack" mean at QVFT?

~~~text
Simulated or farm sensors
        |
        v
ESP32 / Arduino
        |
        v
Authenticated FastAPI ingestion / retrieval
        |
        v
PostgreSQL / Supabase  <-- SQL: sensor histories, alerts, devices, zones
        |
        v
Read-only FastAPI query endpoints  <-- Python: validation and API responses
        |
        v
React + TypeScript dashboard  <-- UI cards, history chart, alert list
        |
        v
Human interpretation / decision support
~~~

**Intelligence is not automatically machine learning.** Version 1 should be an explainable insight feature: rule-based warnings, missing/stale sensor warnings, comparisons to recent history, and relevant recommendations for a human. Never automatically start a pump, adjust nutrient dosing, or issue physical control commands from this project. Control and actuator safety belong to the separate automation workstream.

**Key concepts**

- **Frontend:** The webpage someone sees and uses (React components, layout, accessibility, TypeScript).
- **Backend:** The API that validates requests, controls authorized data access and returns structured results (FastAPI).
- **SQL/PostgreSQL:** Query and organize persisted time-series measurements by sensor, zone and timestamp.
- **Supabase:** Hosted/local PostgreSQL plus access and development tools. Do not expose privileged credentials.
- **Decision support:** Produce understandable warnings and evidence, with metadata and caveats, not uncontrolled actuation.
- **Full-stack integration:** Make data travel safely through all layers and handle empty/error/loading states.

## Role and scope

**Suggested title:** Full-Stack & Farm Intelligence Developer.

**Primary owned feature:** Farm Insights Dashboard (read-only); use synthetic data until the backend/database contracts are available.

**Collaborators:**
- Database/Supabase owner: schema, RLS, index patterns and migration review (issues #3/#13/#21/#23).
- Backend owner: FastAPI route contracts, authorization and data adapters (issues #2/#12).
- Intelligence owner: alert classification, quality flags and explainable recommendation standards (issues #6/#16).
- Plant Science lead: approves actual crop/environment thresholds, interpretation and recommended actions.
- Automation owner: separate command authority; dashboard may display actuator state but should not directly control it.

**Explicitly out of scope for the first milestone:** New database schema ownership, unrestricted direct browser-to-Supabase access, ML model training, physical controls, production deployment, active nutrient dosing, unvalidated crop-health claims.

## Four-week beginner-friendly learning path

Estimated milestones, not hard deadlines. If the contributor has limited time, combine or extend phases without sacrificing tests.

| Week | Focus | Task | Evidence / acceptance |
| --- | --- | --- | --- |
| 1 | React + TypeScript | Local Vite UI using provided mock JSON | Temperature/humidity cards, zone selector, loading/empty/error states; screenshots |
| 2 | SQL + backend | Read-only query + FastAPI GET endpoints | SQL examples, documented response models, FastAPI tests |
| 3 | Intelligence | Explainable high-temperature and stale-sensor insight logic | Rule explanation, sample inputs, severity, no false alert on stale data, unit tests |
| 4 | Integration | Connect live *local* read-only API to dashboard; chart and alerts | Demo with seed data; test failures; PR documentation |

### Learning checkpoints for the team lead

**After Week 1:** Ask them to explain props, state, data types, and where the dashboard's data comes from. Their UI should render sample records without any database connection.

**After Week 2:** Ask them to show a SQL query for latest and historic readings, then explain the difference between a database query and an API endpoint.

**After Week 3:** Ask why a device that stopped reporting should be marked *stale/offline* instead of using its last recorded value as fresh; ask what justifies an environmental threshold.

**After Week 4:** Ask them to reproduce the app from a new checkout, demo an alert from local seed data, then disconnect the API and show the proper error state.

## Setup (Windows PowerShell; adapt to other operating systems)

Prerequisites: Git, a GitHub account with QVFT repository access, VS Code, current supported Node.js/npm, Python 3, and optionally Docker Desktop for local Supabase. Confirm installed versions meet [Vite's requirements](https://vite.dev/guide/) rather than assuming any Node version works.

~~~powershell
git clone https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-.git
cd QVFT-Stack-
git switch -c feature/farm-insights-dashboard
code .
~~~

**First inspect the repo.** If the frontend already has a package.json, use its existing tooling (`npm install`, `npm run dev`). If it is still only a README and there is no app, coordinate with #14, then scaffold *inside the existing frontend/ directory*:

~~~powershell
cd frontend
npm create vite@latest . -- --template react-ts
npm install
npm run dev
~~~

Do not overwrite or scaffold over someone else's implementation; this step is conditional on frontend/ not already having an application.

For a **minimal isolated learning exercise** only, after coordinating with the backend owner, create a Python virtual environment in the repo root:

~~~powershell
cd ..  # only if you are currently in frontend/
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install fastapi uvicorn pytest httpx
~~~

Do not commit .venv, .env files or real keys. Reuse existing backend dependency management if present. Use a dedicated branch and request review before merging.

### Project file suggestions (planned, not verified as existing)

~~~text
frontend/
  src/
    components/
      SensorCard.tsx
      SensorHistoryChart.tsx
      AlertList.tsx
    pages/
      FarmInsightsPage.tsx
    api/
      farmInsights.ts
    types/
      farm.ts
backend/
  app/
    routes/
      insights.py
    services/
      insight_rules.py
  tests/
    test_insights.py
docs/
  fullstack-intelligence-*.md
~~~

## Learning resources: official docs

Read just the relevant sections when working on a task; no need to finish every tutorial first.

| Topic | Authoritative starting point | Goal |
| --- | --- | --- |
| React | https://react.dev/learn | Components, state, rendering lists |
| React + TypeScript | https://react.dev/learn/typescript | Types for props and API data |
| Vite | https://vite.dev/guide/ | Build/run a local React TS app |
| TypeScript | https://www.typescriptlang.org/docs/handbook/intro.html | Interfaces, unions, null handling |
| FastAPI | https://fastapi.tiangolo.com/tutorial/first-steps/ | Read-only GET endpoints and validation |
| FastAPI testing | https://fastapi.tiangolo.com/tutorial/testing/ | TestClient + pytest |
| PostgreSQL SQL | https://www.postgresql.org/docs/current/tutorial-sql.html | SELECT, WHERE, JOIN, ORDER BY, LIMIT |
| Supabase local development | https://supabase.com/docs/guides/local-development | Reproducible local development |
| Supabase security/RLS | https://supabase.com/docs/guides/database/postgres/row-level-security | Least-privilege access |
| Recharts | https://recharts.github.io/ | Time-series visualizations |
| GitHub contribution workflow | https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-pull-requests | PR and code review fundamentals |

## PR checklist

- [ ] Relevant issue claimed/linked and assigned owner(s) consulted.
- [ ] Local setup documented; project uses existing tooling if present.
- [ ] No production secrets, sensitive farm data, unauthorized access or unsafe automation.
- [ ] Mock data works independently of unavailable backend and Supabase infrastructure.
- [ ] UI is responsive, keyboard usable and shows units, timezone and sensor data age.
- [ ] Charts show actual query times and do not connect missing data as though measured.
- [ ] SQL queries are parameterized when invoked from code; no direct privileged DB access in browser.
- [ ] Insight output is interpretable and has proper uncertainty/quality flags.
- [ ] Tests cover normal, empty, invalid, stale, network-error and unauthorized cases.
- [ ] The Pull Request tells another member exactly how to reproduce and evaluate the demo.
