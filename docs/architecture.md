# QVFT Stack System Architecture

This document describes the planned system architecture for the Queen's Vertical Farming Team Stack platform.

The Stack connects physical farm hardware with software infrastructure so that environmental conditions, device status, and farm data can be collected, stored, monitored, and eventually used for automated control.

## System Overview

The QVFT Stack is designed around the following data flow:

Farm Sensors & Actuators  
↓  
ESP32 / Arduino Devices  
↓  
HTTP / MQTT Communication  
↓  
Python / FastAPI Backend  
↓  
PostgreSQL / Supabase Database  
↓  
React + TypeScript Dashboard

Each layer has a specific responsibility and should remain as independent as possible from the other layers.

## 1. Farm Hardware

The physical layer contains the sensors, actuators, microcontrollers, and other devices installed throughout the vertical farm.

Possible sensors include:

- Air temperature
- Relative humidity
- Water temperature
- pH
- Electrical conductivity (EC)
- Water or reservoir level
- Light intensity
- Moisture
- Flow rate
- Pressure
- Plant or canopy monitoring sensors

Possible actuators include:

- Pumps
- Solenoid valves
- Fans
- Grow lights
- Dosing systems
- Relays
- Other environmental control devices

Hardware will be organized by farm zone or subsystem so that measurements can be associated with specific areas of the farm.

## 2. ESP32 and Arduino Devices

ESP32 and Arduino-based devices interface directly with sensors and actuators.

Their responsibilities may include:

- Reading sensors
- Performing basic validation
- Adding device identification
- Adding zone information
- Formatting telemetry
- Sending readings to the backend
- Receiving commands
- Reporting device health
- Controlling local actuators
- Recovering from connection failures

ESP32 devices are preferred where Wi-Fi connectivity or more advanced processing is required.

Arduino devices may be used where simpler local hardware control is sufficient.

## 3. Telemetry Format

Sensor readings should use a consistent format across the system.

Example:

{
  "device_id": "qvft-zone-01",
  "sensor": "air_temperature",
  "value": 23.4,
  "unit": "C",
  "zone": "growth_zone_1",
  "timestamp": "2026-10-19T20:00:00Z"
}

Additional fields may be introduced as the system develops.

Possible future fields include:

- sensor_id
- firmware_version
- device_status
- quality flag
- plot identifier
- greenhouse identifier
- battery voltage
- signal strength

Using a standard telemetry format allows firmware, backend, database, and frontend systems to communicate consistently.

## 4. Device Communication

Farm devices will communicate with the software platform using HTTP and/or MQTT.

### HTTP

HTTP may be used for:

- Sending sensor readings
- Registering devices
- Reporting device health
- Requesting configuration
- Sending occasional events

Example flow:

ESP32  
↓  
HTTP POST  
↓  
FastAPI Endpoint  
↓  
Validation  
↓  
Database

### MQTT

MQTT may be used for:

- Continuous telemetry
- Real-time device communication
- Device status messages
- Control commands
- Automation events

Example topic structure could include:

qvft/telemetry/zone-01  
qvft/status/zone-01  
qvft/commands/zone-01  
qvft/alerts/zone-01

The final MQTT topic structure will be standardized as the firmware and backend systems are developed.

## 5. Backend

The backend acts as the main software interface between farm devices, the database, and the dashboard.

The planned backend technology is:

Python + FastAPI

Backend responsibilities include:

- Receiving telemetry
- Validating incoming data
- Authenticating devices
- Rejecting invalid readings
- Writing data to the database
- Retrieving historical data
- Providing dashboard API endpoints
- Monitoring device status
- Supporting alerts
- Supporting automation logic
- Logging errors
- Providing system health information

Example endpoint structure may include:

GET /health

GET /devices

GET /devices/{device_id}

GET /telemetry

GET /telemetry/latest

POST /telemetry

GET /zones

GET /alerts

Additional endpoints will be added as system requirements develop.

## 6. Database

The planned database system is:

PostgreSQL through Supabase

The database will store information such as:

- Sensor readings
- Device information
- Farm zones
- Plots
- Sensor definitions
- Device status
- Alerts
- Maintenance records
- Environmental history
- Configuration information

Possible database tables include:

devices

sensors

sensor_readings

zones

plots

alerts

maintenance_logs

device_status

Exact database schemas will be documented separately in:

docs/database.md

## 7. Frontend Dashboard

The frontend will provide a visual interface for monitoring the farm.

The planned technologies are:

React  
TypeScript

The dashboard may display:

- Current air temperature
- Relative humidity
- pH
- EC
- Water temperature
- Reservoir level
- Device status
- Historical sensor graphs
- Environmental trends
- Alerts
- Zone information
- Maintenance information
- System health

Future versions may also support:

- Actuator controls
- Automation settings
- Treatment recommendations
- Plant monitoring
- Yield information
- Image-based monitoring
- Predictive analytics

## 8. System Data Flow

A typical sensor reading should move through the system as follows:

Sensor  
↓  
ESP32 reads measurement  
↓  
ESP32 creates telemetry message  
↓  
HTTP or MQTT transmission  
↓  
FastAPI receives message  
↓  
Backend validates message  
↓  
Reading is stored in PostgreSQL / Supabase  
↓  
Dashboard requests data through API  
↓  
React dashboard displays reading

This separation prevents farm hardware from needing direct access to the database.

## 9. Database Security

Farm devices should not connect directly to PostgreSQL or Supabase using privileged database credentials.

Instead, devices should communicate with the backend.

Preferred architecture:

ESP32  
↓  
Authenticated API  
↓  
FastAPI  
↓  
Database

Avoid:

ESP32  
↓  
Direct database connection

This reduces the risk of exposing database credentials inside firmware.

## 10. Environment Variables

Sensitive configuration should be stored using environment variables.

Examples include:

DATABASE_URL

SUPABASE_URL

SUPABASE_ANON_KEY

MQTT_BROKER

MQTT_PORT

MQTT_USERNAME

MQTT_PASSWORD

API_HOST

API_PORT

FRONTEND_API_URL

Real credentials must never be committed to the public repository.

The repository should contain only:

.env.example

Developers should create their own local:

.env

file when configuring the project.

## 11. Device Identification

Every farm-connected device should eventually have a unique identifier.

Example:

qvft-zone-01

Other possible identifiers include:

qvft-reservoir-01

qvft-canopy-01

qvft-lighting-01

qvft-zone-02

A consistent naming convention will make it easier to identify devices, diagnose failures, and associate telemetry with physical locations.

## 12. Device Health

Devices should periodically report their operating status.

Possible health information includes:

- Last connection time
- Firmware version
- Wi-Fi signal strength
- Uptime
- Error state
- Sensor availability
- Last telemetry transmission
- Battery voltage, where applicable

The backend can use this information to determine whether a device is online, offline, degraded, or experiencing an error.

## 13. Alerts

The Stack should eventually support alerts when conditions exceed acceptable limits.

Examples include:

- High temperature
- Low temperature
- High humidity
- Low humidity
- pH outside target range
- EC outside target range
- Low reservoir level
- Sensor failure
- Device offline
- Communication failure
- Pump failure

Alert thresholds should eventually be configurable rather than permanently hard-coded.

## 14. Automation

Automation will be added gradually once reliable sensing, communication, and monitoring infrastructure is established.

Possible future automation includes:

- Irrigation control
- Nutrient dosing
- pH adjustment
- Lighting schedules
- Fan control
- Pump scheduling
- Reservoir management
- Environmental control

Safety-critical automation should include manual overrides and appropriate fail-safe behaviour.

## 15. Reliability

The system should be designed so that temporary network or backend failures do not immediately compromise farm operation.

Future reliability features may include:

- Local buffering of sensor readings
- Automatic reconnection
- Retry logic
- Device watchdog timers
- Backend health checks
- Database backups
- Offline-safe actuator states
- Error logging
- Device heartbeat messages

## 16. Development Architecture

The repository separates the Stack into several areas:

firmware/  
Embedded software for ESP32, Arduino, sensors, and actuators.

backend/  
FastAPI application, APIs, telemetry processing, and backend logic.

frontend/  
React and TypeScript monitoring dashboard.

database/  
Database schemas, migrations, and database-related configuration.

docs/  
Architecture, onboarding, API, hardware, database, and deployment documentation.

scripts/  
Development, testing, data processing, and maintenance utilities.

sample-data/  
Synthetic public data used for testing and demonstration.

## 17. Development Principles

The QVFT Stack should follow several general principles:

### Modular

Hardware, backend, database, and frontend components should remain separated where practical.

### Documented

Important architecture decisions and systems should be documented for future QVFT members.

### Secure

Credentials and sensitive information must not be committed to the repository.

### Testable

Components should be capable of being tested independently where possible.

### Maintainable

The system should be understandable by future students who were not involved in its original development.

### Open Source

Public components should be designed so that other students, researchers, and engineering teams can understand and reuse the work.

## 18. Planned Architecture Development

The current architecture represents the intended direction of the project and will evolve as the physical farm and Stack platform are developed.

Major architecture changes should be documented through:

- GitHub Issues
- Pull Requests
- Updates to this document
- Supporting technical documentation

The goal is to maintain a clear record of how the QVFT Stack system develops over time.
