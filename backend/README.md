# Backend

The backend contains the server-side software for the Queen's Vertical Farming Team's farm monitoring and automation platform.

## Responsibilities

The backend will be responsible for:

- Receiving data from ESP32 and Arduino-based devices
- Validating incoming sensor readings
- Providing API endpoints for the dashboard
- Communicating with the database
- Processing farm telemetry
- Managing device status information
- Supporting alerts and automation logic

## Planned Technologies

- Python
- FastAPI
- PostgreSQL
- Supabase
- REST APIs
- MQTT where appropriate

## Planned Data Flow

```text
Sensors
   ↓
ESP32 / Arduino
   ↓
HTTP / MQTT
   ↓
FastAPI Backend
   ↓
PostgreSQL / Supabase
   ↓
Dashboard
