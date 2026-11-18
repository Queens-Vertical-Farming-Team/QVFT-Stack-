# QVFT Stack — Database & Supabase Step-by-Step Guide

**Companion:** [Implementation roadmap](implementation-roadmap.md) | [Automation guide](automation-control-implementation-guide.md)

**Scope:** Implementation assignments for [#13](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/13), [#21](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/21), [#23](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/23), and [#22](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/22). These are instructions and examples, **not** completed database implementations.

## Before starting

Understand the intended architecture:

~~~text
Farm sensor -> ESP32/Arduino -> authenticated FastAPI -> Supabase/PostgreSQL
                                                       -> dashboard / analytics
                                                       -> history for automation
~~~

No farm device should contain privileged Supabase credentials. The backend must authenticate devices, validate their data, and enforce authorization before database operations. Physical actuator safety must reside locally as well.

Review [architecture.md](architecture.md), [Issue #11 telemetry format](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/11), [Issue #12 FastAPI](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/12), and the existing database README.

### Setup

Use the *local* Supabase CLI environment for migration development. Prerequisites: Docker engine, Node/npm, Git.

~~~powershell
git clone https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-.git
cd QVFT-Stack-
git switch -c feature/issue-13-database-schema
npm install --save-dev supabase
npx supabase init
npx supabase start
~~~

If Supabase is already initialized or included in package.json, reuse the existing configuration. Commit package metadata and supabase/config.toml, not local secrets. See [official local development docs](https://supabase.com/docs/guides/local-development).

---

## #13 — Create initial Supabase schema

### Why this exists

A reading of 23.4°C is meaningless without sensor identity, device identity, physical zone, units and timestamp. Create relational tables that represent those relationships, preserve history and support rapid queries.

### Work in order

1. **Define identifier and naming conventions.** Decide UUIDs for entities, UTC timestamps, and canonical sensor types (e.g. air_temperature, relative_humidity, ph, ec). Coordinate with #11.
2. **Create a versioned migration:** run the command below and edit the generated .sql file.
3. **Create five tables**: zones, devices, sensors, sensor_readings, alerts.
4. **Enforce relationships:** each device belongs to a zone; each sensor belongs to a device; each reading references a sensor; alerts may reference zone, device or sensor.
5. **Add constraints and indexes:** required names/values, valid ranges where universally true, uniqueness for identifiers, efficient latest-reading lookup.
6. **Create synthetic seed data** with at least 2 zones, 2 devices, multiple sensors/readings and 1 alert. Never use actual sensitive farm data in the public repo.
7. **Reset the local database** to prove the migrations and seed data reproduce the state.
8. **Document schema and setup** in database/schema.md and database/README.md.
9. **Open a PR** with migration, seed, documentation and query results.

~~~powershell
npx supabase migration new initial_telemetry
# Edit the newly generated SQL migration
npx supabase db reset
~~~

**Safety:** db reset destroys local dev data. Do not use --linked, --remote or --db-url on shared or production infrastructure.

### Illustrative starter migration

Use this as a *starting point to adapt*, not a committed implementation or finalized design. In particular, agree on identifiers and column names across frontend, backend and firmware before merging.

~~~sql
CREATE TABLE public.zones (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    name text NOT NULL UNIQUE,
    description text,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE public.devices (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    zone_id uuid NOT NULL REFERENCES public.zones(id),
    name text NOT NULL UNIQUE,
    device_type text NOT NULL,
    firmware_version text,
    status text NOT NULL DEFAULT 'unknown',
    last_seen_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE public.sensors (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id uuid NOT NULL REFERENCES public.devices(id),
    sensor_type text NOT NULL,
    unit text NOT NULL,
    model text,
    calibration_date date,
    enabled boolean NOT NULL DEFAULT true,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE public.sensor_readings (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sensor_id uuid NOT NULL REFERENCES public.sensors(id),
    value double precision NOT NULL,
    recorded_at timestamptz NOT NULL,
    received_at timestamptz NOT NULL DEFAULT now(),
    quality_flag text NOT NULL DEFAULT 'valid'
       CHECK (quality_flag IN ('valid', 'suspect', 'invalid', 'stale'))
);

CREATE INDEX idx_sensor_readings_sensor_time
    ON public.sensor_readings (sensor_id, recorded_at DESC);

CREATE TABLE public.alerts (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    zone_id uuid REFERENCES public.zones(id),
    device_id uuid REFERENCES public.devices(id),
    sensor_id uuid REFERENCES public.sensors(id),
    alert_type text NOT NULL,
    severity text NOT NULL,
    message text NOT NULL,
    status text NOT NULL DEFAULT 'open',
    created_at timestamptz NOT NULL DEFAULT now(),
    acknowledged_at timestamptz,
    resolved_at timestamptz
);
~~~

Design review questions:
- What happens if a device is removed? Do we preserve historical readings and alerts?
- Can two devices use the same external hardware ID? If not, enforce uniqueness.
- How does a sensor with invalid readings preserve its raw input and explain the rejection?
- Are UTC timestamps unambiguous? Use timestamptz and require timezone-aware telemetry.

RLS is implemented in #21. Until access is correctly configured, do **not** expose these new tables to frontend clients or deploy them publicly.

### Verification queries

~~~sql
-- Latest reading for a known sensor:
SELECT sensor_id, value, recorded_at
FROM public.sensor_readings
WHERE sensor_id = 'REPLACE_WITH_REAL_SEEDED_UUID'
ORDER BY recorded_at DESC
LIMIT 1;

-- Historical readings for a sensor:
SELECT value, recorded_at
FROM public.sensor_readings
WHERE sensor_id = 'REPLACE_WITH_REAL_SEEDED_UUID'
  AND recorded_at >= now() - interval '24 hours'
ORDER BY recorded_at;

-- Prove the relationship:
SELECT z.name AS zone, d.name AS device, s.sensor_type, r.value, r.recorded_at
FROM public.sensor_readings r
JOIN public.sensors s ON s.id = r.sensor_id
JOIN public.devices d ON d.id = s.device_id
JOIN public.zones z ON z.id = d.zone_id
ORDER BY r.recorded_at DESC
LIMIT 20;
~~~

Replace placeholder identifiers with IDs from your seed data. Use an appropriate SQL client/SQL editor connected to your **local** database.

**Acceptance demonstration:** A teammate checks out the branch, starts local Supabase, resets the local database, successfully inserts a valid reading, fetches the most recent/historical readings, creates an alert, and sees an invalid foreign key rejected. All without access to secrets.

---

## #21 — Design security and RLS model

### Why this exists

Not everyone should be permitted to write readings, modify control settings or request actuator actions. RLS protects data accessed with client-level roles. Privileged service-role/secret credentials bypass RLS and therefore require a trustworthy backend with authorization checks.

### Step by step

1. **Create an access matrix** for every table and operation (SELECT / INSERT / UPDATE / DELETE).
2. **Define identities:** anonymous visitor, authenticated viewer, authorized operator, backend service and authenticated device.
3. **Choose the access route:** preferably frontend requests go to authenticated FastAPI endpoints; no public database writes. Devices call authenticated backend APIs.
4. **Enable RLS** on exposed public-schema tables through a *new* SQL migration, define only the minimum permissions actually needed.
5. **Implement operator authorization in backend code**, not only a client-side hidden button.
6. **Write deny and allow tests** as the relevant roles (including anonymous and unauthorized authenticated users).
7. **Document environment handling**: local .env is gitignored; .env.example contains variable names and placeholders only; production credentials live in deployment secrets.
8. **Open a PR** including the access matrix, migration, negative tests and deployment notes.

Illustrative matrix (adapt to QVFT's actual dashboard architecture):

| Actor | Read measurements | Insert measurements | Change rule/actuator config | Request actuation |
| --- | --- | --- | --- | --- |
| Public visitor | No by default | No | No | No |
| Authenticated viewer | Authorized data only | No | No | No |
| Authorized operator | Authorized data | No | Via authenticated backend | Via authenticated backend |
| ESP32 device | Limited API responses | Via authenticated FastAPI | No | Receive validated commands |
| Backend service | As required | As required | Authenticated/authorized only | Authenticated/authorized only |

~~~sql
-- Example: enable RLS; policies/grants must be separately reviewed.
ALTER TABLE public.sensor_readings ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.actuator_commands ENABLE ROW LEVEL SECURITY; -- only after #22 creates table
~~~

Do not execute the second line until #22 exists. Enabling RLS alone does not create a complete permission model. Be especially cautious of policies written as USING (true), the difference between authenticated and authorized, and service role's ability to bypass RLS.

Official guidance: [Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security).

**Acceptance demonstration:** anonymous reads/writes fail as designed; unauthorized authenticated users cannot issue actuator changes; authorized backend operations work; a secret scan shows no committed credentials.

---

## #23 — Telemetry indexing, retention and quality

### Why this exists

Continuous sensor sampling eventually creates large tables. A farm must also distinguish an abnormal reading from a broken sensor, old measurement or duplicate transmission.

### Step by step

1. **Inventory expected sample rates** with firmware owner: sensor type, frequency, payload size, device count.
2. **Estimate storage**: readings/day = sensors × readings/sensor/day. Example: 20 sensors reporting once/minute = 28,800 rows/day, or 864,000 per 30 days, before backups/index overhead.
3. **Define the most common queries**: latest per sensor; last 24h; all current values in a zone; device health.
4. **Test EXPLAIN ANALYZE** with a reasonably sized synthetic dataset before and after adding indexes.
5. **Define time semantics:** recorded_at is device measurement time; received_at is backend/database receipt time; use UTC.
6. **Create data-quality rules:** valid/suspect/invalid and a *freshness assessment*. Avoid equating old stored history with inherently invalid data; stale may be a derived status at query time.
7. **Define duplicate policy:** prefer a device-generated message ID or device+sequence number; duplicate delivery should not create uncontrolled repeated triggers.
8. **Plan raw-data retention and downsampling**, separately from the current implementation. Discuss expected query needs, research value and backups before deleting anything.
9. **Write a document** at docs/telemetry-quality.md explaining indexes, estimated growth, timeouts, quality rules, retention proposal and restore/export expectations.

Example:

~~~sql
EXPLAIN ANALYZE
SELECT value, recorded_at
FROM public.sensor_readings
WHERE sensor_id = 'REPLACE_WITH_REAL_SEEDED_UUID'
ORDER BY recorded_at DESC
LIMIT 100;
~~~

**Acceptance demonstration:** query-performance evidence, duplicate data behavior, explicit invalid/stale handling, and agreed retention **proposal** (not an accidental deletion job).

---

## #22 — Add actuator and automation tables

**Dependency:** #13 and #17 must establish telemetry relationships and the command protocol first. Coordinate with the automation owner.

### Why this exists

QFVT needs to answer not just "what temperature was detected?" but "what rule requested a fan change, what command was sent, did the mock controller acknowledge it, and what state did it report?"

### Step by step

1. **Read #17's command specification** and confirm field names/status lifecycle before designing tables.
2. **Create a new SQL migration** with five tables: actuators, actuator_commands, actuator_state, automation_rules, automation_events.
3. **Connect actuators** to owning device and zone; store type, enable status, and default mode.
4. **Record every command** with unique ID, actuator ID, requested action/value, command source, timestamp, expiry, priority, status and failure reason.
5. **Track reported actuator state separately** from requested commands: command completion requires device acknowledgement/feedback.
6. **Store configurable rules** with enable flag, sensor linkage, thresholds, debounce and hysteresis settings, last modification information.
7. **Link automation events** to rules, optional measured reading(s), generated command(s), result and reason.
8. **Add indexes/constraints** for recent actuator history, command expiry, outstanding commands and latest reported actuator state.
9. **Update the RLS model** from #21 for the new tables; ensure operators cannot arbitrarily write "completed" commands directly.
10. **Seed and query a complete example**: high temperature -> rule event -> FAN_ON request -> mock acknowledgement -> reported ON state.
11. **Open PR** with migration, seed, tests and a short entity-relationship diagram.

Suggested *design fields* (not final executable SQL):

| Table | Key fields |
| --- | --- |
| actuators | id, device_id, zone_id, type, enabled, mode |
| actuator_commands | id, actuator_id, action, value, source, requested_at, expires_at, priority, status, failure_reason |
| actuator_state | actuator_id, reported_value, reported_at, last_command_id |
| automation_rules | id, name, sensor_id, actuator_id, enabled, thresholds, debounce_seconds, configuration |
| automation_events | id, rule_id, reading_id, command_id, triggered_at, outcome, reason |

**Important:** A single rules event may eventually generate multiple commands; choose the correct cardinality after discussing team requirements. Treat ACK and completion as separate lifecycle states. Consider idempotency/unique command IDs for retries.

A useful query for reviewers is a command-history query that joins automation_events to rules, commands, actuators and reported state.

**Acceptance demonstration:** Reviewer can trace both a successful and a rejected action from rule condition to final recorded outcome. No remote/production migrations are required to complete the issue.

---

## Database PR checklist

- [ ] Reproducible local setup and timestamped SQL migrations.
- [ ] Seed data has no secrets or production-sensitive information.
- [ ] Primary/foreign keys, uniqueness and common indexes documented.
- [ ] UTC timestamps and stale/duplicate conventions explicit.
- [ ] Positive and negative SQL/API tests included.
- [ ] New table access protected; no unintended public writes.
- [ ] Supabase privileged credentials confined to the backend/deployment environment.
- [ ] Issue linked, test evidence supplied, another member reviews before merge.
