# QVFT Stack — DevOps & Infrastructure Step-by-Step Guide

**Companion:** [Implementation roadmap](implementation-roadmap.md) | [Database & Supabase guide](database-implementation-guide.md) | [Automation & Control guide](automation-control-implementation-guide.md) | [Intelligence guide](intelligence-fullstack-implementation-guide.md) | [Sensors & Firmware guide](sensors-firmware-implementation-guide.md)

**Scope:** Implementation assignments for [#19](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/19), [#41](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/41), [#42](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/42), [#43](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/43), [#44](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/44), [#45](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/45), and [#46](https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-/issues/46).

These are implementation **instructions and verification requirements**, not claims that QVFT already has a production infrastructure platform.

> **Infrastructure safety boundary:** Do not run destructive database commands, rotate shared credentials, modify production DNS, deploy to shared infrastructure, or change repository protection/settings without the appropriate owner/Stack-lead approval. All destructive/recovery testing should begin in local or isolated staging environments.

---

## What DevOps & Infrastructure means for QVFT

DevOps is the layer that makes the rest of the Stack reproducible and operational.

It connects:

~~~text
developer laptop
    ↓
repeatable local setup
    ↓
Git branch / Pull Request
    ↓
automated CI checks
    ↓
review + merge
    ↓
staging / deployment
    ↓
health checks + logs
    ↓
backup / recovery / incident response
~~~

This contributor is not expected to "deploy everything" immediately.

Their first responsibility is to make development and verification **repeatable**.

A good DevOps system answers:

- How does a new member set up the project?
- Which versions/tools are required?
- Where do secrets live?
- How do we know a PR is safe to merge?
- How do we start all required services?
- How do we know a service is healthy?
- How do we deploy without guessing?
- How do we recover when something breaks?
- Can another member reproduce all of this?

---

# Recommended task order

~~~text
#19  Reproducible local development environment
 ↓
#41  Environment configuration + secrets strategy
 ↓
#42  Local service bootstrap / orchestration
 ↓
#43  GitHub Actions CI
 ↓
#44  Deployment + staging strategy
 ↓
#45  Health checks / logging / observability
 ↓
#46  Backup / recovery / operational runbook
~~~

Some work may proceed in parallel once interfaces are stable.

---

# Engineering checks and balances

For DevOps work, **"the YAML exists" or "it deployed once" is not enough**.

Every task should pass:

~~~text
1. REPRODUCE
   Can another person follow it?

2. VERIFY
   Can we prove the expected service/check is healthy?

3. BREAK
   What happens when a dependency/configuration is wrong?

4. RECOVER
   Is the recovery or rollback documented and tested?

5. AUDIT
   Are secrets, logs, changes and operational decisions traceable?
~~~

## Evidence required in PRs

Depending on the task, include:

- exact commands run
- OS/tool versions
- screenshots or terminal logs
- CI run results
- deliberate failing CI run
- health-check output
- service startup/shutdown output
- secret-scan result
- staging deployment result
- rollback/simulated rollback result
- restore-test evidence
- known limitations
- exact reproduction steps

Use a test table:

| Test ID | Purpose | Setup | Expected | Observed | Result | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| DEV-01 | Clean setup | Fresh clone | Services start | Services started | PASS | terminal log |
| CI-02 | Broken test | PR with deliberate failure | CI red | CI red | PASS | CI run |
| OPS-03 | Backend unavailable | Stop backend | health fails clearly | failed as expected | PASS | log |

A failed test is useful evidence. Do not hide failures.

---

# #19 — Define the local development environment

## Why this exists

A project is not reproducible if each member has a private collection of commands and tools that only work on their machine.

The target is:

~~~text
clean clone
→ install documented prerequisites
→ create local config
→ start current services
→ verify health
→ run tests
~~~

without private instructions.

## Step 1 — Inventory the current components

For each component that actually exists, record:

| Component | Tool/runtime | Version | Start command | Test command | Health check |
| --- | --- | --- | --- | --- | --- |
| Backend | Python | TBD/agreed | TBD | TBD | GET /health |
| Frontend | Node/npm | TBD/agreed | TBD | TBD | browser/build |
| Supabase | Docker + CLI | agreed setup | documented | db checks | local status |
| Firmware | selected toolchain | agreed | build/flash | compile/tests | serial boot |

Do not document imaginary commands for components that have not yet been bootstrapped.

## Step 2 — Pin or document versions

At minimum document:

- Python major/minor version
- Node.js major version
- package manager
- Docker requirement
- Supabase local tooling requirement
- firmware toolchain/framework

The objective is to reduce:

> "It works on Node 24 on my machine but not Node 20 on yours."

Use version files only if the team agrees to maintain them.

Examples that may be appropriate later:

~~~text
.python-version
.nvmrc
toolchain config
package-lock.json
requirements/lock file
~~~

Do not create duplicate dependency-management systems.

## Step 3 — Create one setup guide

The setup guide should have the exact order.

Example shape:

~~~powershell
git clone https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-.git
cd QVFT-Stack-
git switch main
git pull

# backend setup when present
py -m venv .venv
.\.venv\Scripts\Activate.ps1

# frontend setup when present
npm install

# local database according to database guide
npx supabase start
~~~

Only include commands that match the actual repository.

## Step 4 — Add verification

The contributor must define how to know setup succeeded.

Examples:

~~~text
Backend:
GET /health -> expected 200

Frontend:
development server opens and build command succeeds

Database:
local Supabase reports running; a known query succeeds

Firmware:
firmware compile command succeeds
~~~

## Step 5 — Add troubleshooting

At minimum document:

- command not found
- Python virtual environment activation issue
- Node/npm version mismatch
- Docker not running
- port already in use
- missing .env
- Supabase local service failure
- firmware toolchain missing

## Verification gate for #19

### DEV-01 — Clean-clone setup

Reviewer uses a fresh directory and only the written instructions.

**Pass:** no private/undocumented commands are required.

### DEV-02 — Missing prerequisite

Temporarily simulate a missing required tool or test on a machine without it.

**Pass:** guide clearly identifies the prerequisite rather than failing mysteriously.

### DEV-03 — Missing environment file

Start without local environment configuration.

**Pass:** service fails clearly with an actionable message; it does not silently use unsafe production defaults.

### DEV-04 — Port conflict

Occupy one documented development port.

**Pass:** troubleshooting explains how to identify and resolve the conflict.

### DEV-05 — Second-person reproduction

Another member sets up the project.

**Pass:** they can reach the same health checks.

**Acceptance demonstration:** Second member records OS, versions, commands, PASS/FAIL, and any undocumented step discovered.

---

# #41 — Environment configuration and secrets strategy

## Why this exists

QVFT will have configuration values that are safe to expose and credentials that are not.

These must not be treated the same.

Examples of **configuration**:

~~~text
API host
API port
frontend API URL
feature flags
development mode
~~~

Examples of **secrets**:

~~~text
database password
Supabase privileged secret key
MQTT password
private API token
deployment credential
~~~

A frontend/browser variable is not secret merely because it came from an environment file. Anything sent to browser code should be assumed visible to the user.

## Step 1 — Create a configuration inventory

Create a table:

| Variable | Component | Secret? | Local source | CI source | Deployment source | Owner |
| --- | --- | --- | --- | --- | --- | --- |
| SUPABASE_URL | backend | no/safe config | local env | CI env | deployment config | DB owner |
| privileged DB key | backend only | YES | local secret | CI secret if required | secret store | DB owner |
| FRONTEND_API_URL | frontend | no | local env | build config | environment config | frontend |

Use current platform/key naming agreed by the Database owner rather than hard-coding obsolete names into new infrastructure.

## Step 2 — Update `.env.example`

The example file should contain:

- variable names
- comments
- placeholder values only

Never:

~~~text
real passwords
real private tokens
shared production credentials
~~~

## Step 3 — Define boundaries

### Frontend

Only browser-safe configuration.

### Backend

May access server-side secrets.

### Firmware

Should not contain privileged database keys. Device credentials, if used later, must be scoped for the device use case.

### CI

Secrets should only be provided to jobs that require them.

### Staging/production

Use the deployment platform's protected secret/configuration mechanism.

## Step 4 — Rotation/revocation

Document:

~~~text
credential suspected leaked
→ revoke/rotate at provider
→ update authorized secret store
→ restart/redeploy affected service
→ verify old credential fails
→ verify new credential succeeds
→ document incident
~~~

Never commit a "new" secret to fix a leaked committed secret.

## Verification gate for #41

### SEC-01 — Repository secret scan

Search repository/history as appropriate for obvious credential patterns and use an approved scanner if adopted.

**Pass:** no known real secret appears in current tracked files.

### SEC-02 — Frontend exposure check

Build/run frontend and inspect what configuration is shipped to the browser.

**Pass:** no privileged backend/database secret is exposed.

### SEC-03 — Missing secret

Remove one required backend secret locally.

**Pass:** backend fails clearly; no unsafe fallback.

### SEC-04 — Rotation simulation

Use a safe development credential/configuration.

**Pass:** contributor can explain and demonstrate how a value is replaced without editing source code.

### Lead questions

- Which values are safe in the browser?
- Which values must remain server-side?
- If this key leaks tonight, what exactly do we do?
- Where does CI obtain it?
- Who owns it?

---

# #42 — Local Stack bootstrap and service orchestration

## Why this exists

When the Stack has several components, contributors should not have to remember five terminals and an undocumented startup order.

The objective is:

~~~text
one documented bootstrap path
→ required services start
→ readiness checks pass
→ clean shutdown works
~~~

This does not mean every component must be containerized.

Use containers/orchestration only where they improve reproducibility.

## Step 1 — Draw the local service topology

Example:

~~~text
React frontend :5173
        ↓
FastAPI backend :8000
        ↓
Supabase/PostgreSQL local services
~~~

Optional services such as MQTT should only appear if actually used.

## Step 2 — Define ports

Maintain a table:

| Service | Local port | Required? | Health/readiness |
| --- | ---: | --- | --- |
| frontend | agreed | yes when developing UI | page/build |
| backend | agreed | yes | /health |
| database | managed local ports | yes for data tasks | Supabase status |
| MQTT | agreed | optional | broker connection |

## Step 3 — Choose orchestration

Possible options:

- documented scripts
- Make/PowerShell scripts
- Docker Compose for team-owned container services
- existing Supabase local tooling for Supabase services

Do not wrap Supabase's local tooling in a conflicting duplicate database stack.

## Step 4 — Add readiness checks

"Process started" is not equal to "service ready."

Examples:

~~~text
backend process exists
≠ backend /health is ready

database container exists
≠ migration/schema is usable
~~~

## Step 5 — Clean shutdown/reset

Document:

- normal stop
- clean restart
- local reset where safe
- which reset commands destroy local data

## Verification gate for #42

### ORCH-01 — One-command/documented bootstrap

**Pass:** contributor starts all currently required services using the documented flow.

### ORCH-02 — Dependency missing

Stop database, then start backend.

**Pass:** dependency failure is clear and diagnosable.

### ORCH-03 — Readiness

Attempt health check immediately during startup.

**Pass:** readiness behavior is meaningful; system does not falsely claim readiness.

### ORCH-04 — Shutdown/restart

Stop all services, then restart.

**Pass:** no orphaned/conflicting service prevents a clean restart.

### ORCH-05 — Port collision

Force a collision.

**Pass:** error/troubleshooting identifies the port problem.

### ORCH-06 — Second-member reproduction

Another member uses the bootstrap flow.

**Pass:** same service topology and health results.

---

# #43 — GitHub Actions CI for Pull Requests

## Why this exists

Human reviewers should not have to remember every build/test command manually.

CI should automatically answer:

> Does this PR satisfy the checks we currently know how to automate?

CI should grow with the actual project.

Do not create fake checks that always pass.

## Initial workflow structure

A future workflow might contain jobs such as:

~~~text
repository/docs check
backend test
frontend build/test
firmware compile
database/migration validation
secret/security check
~~~

Only enable jobs when the component and command actually exist.

## Example minimal GitHub Actions shape

Illustrative only:

~~~yaml
name: PR Checks

on:
  pull_request:
    branches: [main]

jobs:
  backend-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.x"
      - name: Install backend dependencies
        run: echo "Replace with actual project install command"
      - name: Run tests
        run: echo "Replace with actual project test command"
~~~

Do not merge placeholder `echo` jobs as proof that CI exists. Replace them with real checks before marking the Issue complete.

## Step by step

1. List commands developers already run locally.
2. Choose deterministic checks.
3. Add GitHub Actions workflow.
4. Install dependencies in a clean runner.
5. Run real test/build commands.
6. Keep jobs scoped by component where useful.
7. Avoid exposing secrets to PR code unless absolutely necessary.
8. Make failure logs understandable.
9. Document how to reproduce CI failures locally.
10. Later, with lead approval, consider making reliable checks required before merge.

## Verification gate for #43

### CI-01 — Green path

Open/update PR with valid code.

**Pass:** expected jobs are green.

### CI-02 — Deliberate test failure

Temporarily change a test/assertion in a dedicated test branch so it fails.

**Pass:** CI turns red and identifies the failing check.

Revert the deliberate failure afterward.

### CI-03 — Build failure

Introduce a controlled syntax/build failure in a disposable branch.

**Pass:** corresponding CI job fails.

### CI-04 — Missing dependency/lock consistency

Test clean runner installation.

**Pass:** CI does not rely on globally installed developer dependencies.

### CI-05 — Secret output

Review logs.

**Pass:** no privileged secret is printed.

### CI-06 — Local reproduction

Contributor uses the same documented test/build command locally.

**Pass:** failure can be debugged without guessing what CI did.

### Stack-lead rule

Do not require a CI job for merges until it is stable enough that failures actually mean something.

---

# #44 — Deployment architecture and staging strategy

## Why this exists

Deployment should not mean:

> "I ran something from my laptop and now the site changed."

Define a repeatable path.

## Step 1 — Inventory deployable components

For each component:

| Component | Deployable now? | Build artifact | Host/platform | Config owner |
| --- | --- | --- | --- | --- |
| frontend | TBD | static/app build | TBD | frontend/devops |
| backend | TBD | Python service/container | TBD | backend/devops |
| database | Supabase migrations/config | SQL migrations | Supabase | DB/devops |

Do not mark components deployable before they exist.

## Step 2 — Define environments

Recommended concepts:

~~~text
local
staging/demo
production
~~~

Not every student project needs a permanent production environment immediately.

Staging/demo should be the place to validate deployments without affecting the real farm/production users.

## Step 3 — Define deployment source

Examples:

- merge to a designated branch
- versioned release/tag
- manually approved workflow

Do not deploy arbitrary unreviewed feature branches to production.

## Step 4 — Database migration process

Before application deploy:

- know whether migration is backward-compatible
- test locally/staging
- back up where required
- run migration through an agreed process
- verify schema
- deploy dependent code
- perform smoke test

Avoid automatically running destructive migrations without safeguards.

## Step 5 — Smoke test

After deployment check:

~~~text
service starts
/health responds
database dependency works
frontend reaches expected backend
one safe read path works
logs contain no immediate critical error
~~~

## Step 6 — Rollback

Document:

- previous application version
- how to redeploy it
- whether database changes are reversible
- what to do if migration cannot simply be rolled back
- who approves rollback

## Verification gate for #44

### DEP-01 — Staging deploy

**Pass:** authorized second member can reproduce the documented staging/demo deployment.

### DEP-02 — Wrong configuration

Deploy/start staging with one intentionally wrong non-secret config value.

**Pass:** health/smoke tests reveal the problem clearly.

### DEP-03 — Failed application release

Use a safe staged broken build/release.

**Pass:** deployment is blocked or smoke test fails.

### DEP-04 — Rollback

Return staging to previous known-good version.

**Pass:** service health returns.

### DEP-05 — Migration dry run/test

Apply migrations to isolated local/staging data.

**Pass:** expected schema/data checks succeed.

Do not use shared production data for destructive practice.

---

# #45 — Health checks, logging and observability

## Why this exists

When a service fails, QVFT should be able to answer:

- Is the process running?
- Is it ready to serve requests?
- Can it reach its dependencies?
- Which request/device failed?
- What happened immediately before failure?

## Health concepts

### Liveness

Is the service process alive?

### Readiness

Can it currently do the work it claims to provide?

For a backend, readiness may depend on required services such as the database.

Do not create an expensive health endpoint that performs a full system test on every request.

## Logging conventions

At minimum consider:

~~~text
timestamp
level
component
message
request/correlation ID
device ID where relevant
error code
~~~

Do not log:

~~~text
passwords
authorization tokens
private keys
privileged connection strings
full secret environment dumps
~~~

## Example conceptual log

~~~text
2026-11-22T20:15:01Z INFO backend.telemetry request_id=req-83 device=qvft-zone-01 telemetry accepted
2026-11-22T20:15:05Z ERROR backend.database request_id=req-91 database unavailable
~~~

## Step by step

1. Define health endpoint behavior.
2. Define dependency checks.
3. Define log levels:
   - DEBUG
   - INFO
   - WARNING
   - ERROR
4. Add request/correlation ID strategy where useful.
5. Add device IDs for device-related backend logs.
6. Define where local/staging logs are viewed.
7. Propose useful metrics:
   - request error rate
   - request latency
   - device heartbeat age
   - database errors
   - telemetry rejection count
8. Delay strict production SLO/alert thresholds until enough operational data exists.

## Verification gate for #45

### OBS-01 — Healthy service

**Pass:** health returns expected status and useful information.

### OBS-02 — Database dependency failure

Stop/unavailable database in local/staging.

**Pass:** readiness/logging shows dependency problem rather than generic success.

### OBS-03 — Invalid request

Send invalid telemetry/API request.

**Pass:** log identifies failure without leaking sensitive payload/secrets.

### OBS-04 — Correlation

Trace one request through relevant logs.

**Pass:** reviewer can identify related entries.

### OBS-05 — Secret redaction

Use a test value shaped like a credential.

**Pass:** secrets are not written to normal logs.

### Lead demo

Ask:

> "The dashboard isn't receiving new telemetry. Show me how you determine whether the backend, database, or device path is the problem."

The contributor should use health/log evidence rather than guessing.

---

# #46 — Backup, recovery and operational runbook

## Why this exists

Infrastructure is incomplete if the only plan for failure is:

> "Hopefully it doesn't happen."

The runbook should define common failure responses.

## Required runbook scenarios

### Service unavailable

- verify health
- inspect recent deployment/config change
- inspect logs
- rollback/restart according to documented policy
- verify recovery

### Database unavailable

- verify provider/local service state
- stop unsafe dependent writes if required
- avoid repeated destructive recovery attempts
- restore/reconnect according to tested procedure

### Migration failure

- stop deployment
- record current migration state
- do not blindly rerun destructive SQL
- follow migration-specific recovery/rollback procedure

### Credential compromise

~~~text
revoke/rotate
→ update secret store
→ redeploy/restart
→ verify old credential fails
→ verify new credential succeeds
→ investigate exposure
→ document incident
~~~

### Bad deployment

- identify previous known-good version
- rollback application
- verify health/smoke tests
- decide how database compatibility affects rollback

## Backup design questions

Document:

- what is backed up?
- who owns backup configuration?
- how often?
- retention?
- where?
- how is access protected?
- has restoration been tested?
- what data loss window is acceptable later?

Do not claim backups are useful merely because a provider says backups exist. Restoration must be understood and periodically tested on safe data.

## Incident note template

~~~text
Incident:
Date/time:
Affected component:
Detected by:
User/farm impact:
Initial symptoms:
Root cause:
Recovery steps:
Data loss:
Credentials affected:
Follow-up Issues:
Preventive action:
~~~

## Verification gate for #46

### REC-01 — Backup existence/config check

**Pass:** contributor can identify what backup/recovery capability exists for the current environment.

### REC-02 — Safe restore test

Use local/staging/test data.

**Pass:** selected data/service can be restored according to procedure.

### REC-03 — Bad deployment exercise

Simulate a failed staged deployment.

**Pass:** runbook returns service to known-good state.

### REC-04 — Credential compromise tabletop

Pretend one development/staging credential leaked.

**Pass:** contributor can state exact revoke/rotate/update/verify steps.

### REC-05 — Runbook handoff

Another member follows one scenario without the author coaching them.

**Pass:** recovery path is understandable.

---

# Cross-workstream DevOps checks

DevOps should not work in isolation.

## Database owner

Coordinate:

- migrations
- local Supabase
- secret boundaries
- backup/restore
- staging schema

## Backend owner

Coordinate:

- dependency installation
- start/test commands
- /health
- logs
- deployment artifact

## Frontend owner

Coordinate:

- Node/package manager
- build command
- browser-safe environment variables
- API URL configuration
- deployment build

## Firmware owner

Coordinate:

- firmware build tooling
- CI compile checks
- configuration/secrets boundary
- device health integration

## Automation/Intelligence owners

Coordinate:

- test commands
- service dependencies
- logs/health for their modules
- safe staging/synthetic tests

---

# End-to-end DevOps verification

After initial tasks are complete, run this exercise.

## OPS-E2E-01 — New contributor setup

A member who did not build the infrastructure:

~~~text
clone repo
→ follow setup docs
→ configure local environment
→ start available services
→ run tests
→ verify health
~~~

**Pass:** no hidden instructions.

## OPS-E2E-02 — PR quality gate

Create a test PR.

**Pass:** CI runs the expected checks.

Then deliberately introduce a failing test/build in the test branch.

**Pass:** CI blocks/flags the change.

Revert the failure.

## OPS-E2E-03 — Dependency failure

Stop one dependency.

**Pass:** service health/logs explain the dependency problem.

## OPS-E2E-04 — Secret/config failure

Remove one required local/staging value.

**Pass:** service fails clearly and securely.

## OPS-E2E-05 — Staging deploy

Deploy a safe version to staging/demo if a deployable service exists.

**Pass:** smoke test succeeds.

## OPS-E2E-06 — Rollback

Deploy or simulate a bad staged release.

**Pass:** documented rollback returns known-good health.

## OPS-E2E-07 — Recovery exercise

Run one backup/restore or operational tabletop.

**Pass:** second member can follow the runbook.

---

# Stack-lead sign-off sheet

## Local development

- [ ] Fresh clone setup is documented.
- [ ] Tool/runtime versions are documented.
- [ ] Second member reproduced setup.
- [ ] Startup and shutdown are documented.
- [ ] Troubleshooting covers common failures.

## Configuration/secrets

- [ ] Config inventory exists.
- [ ] Public vs secret values are distinguished.
- [ ] No privileged secret is committed.
- [ ] Frontend receives no backend secret.
- [ ] Rotation/revocation is documented.

## Orchestration

- [ ] Current services can be started reproducibly.
- [ ] Ports are documented.
- [ ] Readiness checks exist.
- [ ] Missing dependency behavior tested.
- [ ] Restart works cleanly.

## CI

- [ ] PR workflow runs real checks.
- [ ] Good change produces green CI.
- [ ] Deliberate failure produces red CI.
- [ ] CI failure is reproducible locally.
- [ ] CI logs expose no secrets.

## Deployment

- [ ] Environment separation documented.
- [ ] Deployment source/approval documented.
- [ ] Smoke test exists.
- [ ] Migration process documented.
- [ ] Rollback tested in safe environment.

## Observability

- [ ] Health check exists where service exists.
- [ ] Dependency failure is visible.
- [ ] Logs are useful and structured consistently.
- [ ] Sensitive values are not logged.
- [ ] Contributor can diagnose a simulated failure.

## Recovery

- [ ] Runbook exists.
- [ ] Backup/recovery capability documented.
- [ ] Safe restore or recovery exercise completed.
- [ ] Credential compromise procedure exists.
- [ ] Another member can follow one runbook scenario.

## Learning

- [ ] Contributor can explain why CI exists.
- [ ] Contributor can distinguish config from secrets.
- [ ] Contributor can explain readiness vs liveness.
- [ ] Contributor can describe staging vs production.
- [ ] Contributor can perform a rollback without blindly deleting resources.
- [ ] Contributor can diagnose a failure from evidence.
- [ ] Contributor states what AI helped with and what they personally verified.

If these cannot be demonstrated, the infrastructure may be a useful prototype, but it should not yet be treated as reliable operational infrastructure.

---

# Learning review questions

Ask the DevOps contributor:

- If I clone this on a clean computer, what is the first thing I do?
- Which tool versions matter and why?
- Which environment variables are safe in the frontend?
- If a secret is committed accidentally, why isn't deleting the line enough?
- What exact PR change will make CI fail?
- How can I reproduce a CI failure locally?
- What is the difference between a process being alive and a service being ready?
- How do you know a deployment succeeded?
- What do you do if the deployment succeeds but the health check fails?
- What is the rollback path?
- What happens if a database migration is not backward-compatible?
- Where would you look first if telemetry suddenly stops reaching the database?
- What is actually backed up?
- Have we tested a restore?
- What did AI generate or help with?
- What did you personally verify?
- Can you change a service port, CI check, or environment value without asking AI where it is configured?

The objective is to use automation and AI to reduce repetitive work while still learning reproducibility, configuration management, CI/CD, deployment safety, debugging, observability, and recovery.
