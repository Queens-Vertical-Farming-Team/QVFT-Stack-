# QVFT Stack Workstreams

Workstreams organize the team's engineering areas. They are not Git branches and they are not necessarily one-to-one with folders.

Members should choose one or two primary workstreams, then claim individual GitHub Issues within them.

| Workstream | Purpose | Main repository areas |
|---|---|---|
| Sensors & Firmware | Read sensors, control local hardware, format telemetry | `firmware/` |
| Backend & Communications | Receive telemetry, APIs, HTTP/MQTT, validation | `backend/` |
| Database & Supabase | Schemas, migrations, permissions, historical data | `database/` |
| Dashboard | User interface, graphs, farm status and alerts | `frontend/` |
| Plant Vision | Camera capture, NIR/multispectral work, image processing | `vision/` |
| Intelligence | Rules, anomaly detection, prediction and decision support | `intelligence/` |
| Automation & Control | Safe actuator commands and closed-loop control | `automation/`, `firmware/` |
| Testing & Device Health | Calibration, validation, integration tests, diagnostics | `tests/` |
| DevOps & Infrastructure | Local setup, deployment, environments and CI | `infra/`, `.github/` |

## 1. Sensors & Firmware

Initial goals:

- establish supported ESP32/Arduino hardware
- read environmental sensors reliably
- define sensor IDs and zone IDs
- standardize telemetry messages
- implement calibration procedures
- report device health

## 2. Backend & Communications

Initial goals:

- bootstrap FastAPI
- implement `GET /health`
- implement telemetry ingestion
- validate sensor payloads
- establish HTTP/MQTT conventions
- connect backend services to Supabase

## 3. Database & Supabase

Initial goals:

- create the Supabase project
- define `devices`, `sensors`, `sensor_readings`, `zones` and `alerts`
- define indexes and retention strategy
- implement safe credentials and Row Level Security where appropriate
- document migrations

## 4. Dashboard

Initial goals:

- bootstrap React + TypeScript
- display latest measurements
- show historical graphs
- display device health
- display alerts
- expose camera/plant-health views

## 5. Plant Vision

Initial goals:

- choose camera hardware and mounting
- define capture schedule
- implement image capture/storage pipeline
- determine NIR/multispectral approach for true NDVI
- test canopy and vegetation monitoring
- later support disease/stress analysis

## 6. Intelligence

Initial goals:

- start with transparent rule-based alerts
- define acceptable environmental ranges
- detect anomalous sensor behaviour
- combine sensor and vision information
- later test prediction and optimization models

## 7. Automation & Control

Initial goals:

- define safe actuator command format
- design manual override
- define pump/fan/light/valve control interfaces
- define fail-safe states
- separate recommendations from automatic actions
- later implement closed-loop control

## 8. Testing & Device Health

Initial goals:

- sensor calibration tests
- telemetry validation tests
- disconnected-device detection
- backend/database integration tests
- actuator safety tests
- camera repeatability tests

## 9. DevOps & Infrastructure

Initial goals:

- standardize local environments
- protect secrets
- add automated checks
- document deployment
- keep production and development configuration separate

## Choosing a workstream

A member can contribute across workstreams, but should have a clear primary area.

Use GitHub Issues for actual assignments. Workstream Issues marked `[EPIC]` are planning/coordination Issues; individual task Issues are the pieces of work to claim.
