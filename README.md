# Queen's Vertical Farming Team — Stack

The QVFT Stack Team develops the software, embedded systems, data infrastructure, and monitoring tools that support the Queen's Vertical Farming Team's automated vertical farming system.

QVFT is a multidisciplinary student design team at Queen's University focused on developing accessible, automated, and open-source vertical farming technology.

## Our Goal

Our goal is to connect the physical farm with a reliable digital platform that allows the team to collect data, monitor growing conditions, understand system performance, and eventually support automated farm control.

By making our software and documentation open source, we hope our work can also be useful to students, researchers, growers, and other engineering teams.

## System Architecture

Our planned Stack architecture follows this general structure:

Farm Sensors & Actuators  
↓  
ESP32 / Arduino  
↓  
HTTP / MQTT  
↓  
Python / FastAPI  
↓  
PostgreSQL / Supabase  
↓  
React + TypeScript Dashboard

## What the Stack Team Works On

The Stack Team is responsible for software and digital infrastructure across the farm, including:

- Embedded sensor and actuator firmware
- ESP32 and Arduino development
- Farm telemetry
- Device communication
- Backend APIs
- Database infrastructure
- Environmental monitoring
- Historical data visualization
- Device health monitoring
- Alerts and diagnostics
- Farm dashboards
- Automation infrastructure
- Technical documentation

## Repository Structure

```text
qvft-stack/
│
├── firmware/
│   └── Embedded software for sensors, actuators, ESP32s, and Arduino devices
│
├── backend/
│   └── Python backend, APIs, telemetry processing, and system logic
│
├── frontend/
│   └── React and TypeScript farm monitoring dashboard
│
├── database/
│   └── PostgreSQL / Supabase schemas, migrations, and database documentation
│
├── docs/
│   └── Architecture, onboarding, API, sensor, and system documentation
│
├── scripts/
│   └── Development, testing, maintenance, and data utility scripts
│
├── sample-data/
│   └── Public synthetic data for development and testing
│
├── .github/
│   └── GitHub contribution and Pull Request configuration
│
├── .env.example
├── CONTRIBUTING.md
└── README.md
```

## Technologies

### Embedded Systems

C / C++  
Arduino  
ESP32

### Backend

Python  
FastAPI  
REST APIs  
MQTT

### Database

PostgreSQL  
Supabase

### Frontend

React  
TypeScript  
JavaScript  
HTML / CSS

### Development

Git  
GitHub  
GitHub Issues  
Pull Requests  
GitHub Actions

## Example Farm Telemetry

A standard sensor message may look like:

```json
{
  "device_id": "qvft-zone-01",
  "sensor": "air_temperature",
  "value": 23.4,
  "unit": "C",
  "zone": "growth_zone_1",
  "timestamp": "2026-10-19T20:00:00Z"
}
```

Maintaining a consistent telemetry format makes it easier for firmware, backend, database, and frontend systems to work together.

## Development Workflow

QVFT Stack development follows a branch and Pull Request workflow.

GitHub Issue  
↓  
Feature / Fix Branch  
↓  
Development  
↓  
Testing  
↓  
Pull Request  
↓  
Code Review  
↓  
Merge to Main

Contributors should avoid committing directly to `main` once the initial repository setup is complete.

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for our development and review guidelines.

## Documentation

Technical documentation is maintained inside the [`docs/`](docs/) directory.

Documentation will cover areas including:

- System architecture
- Developer onboarding
- Sensors and hardware
- Backend APIs
- Database design
- Deployment
- Troubleshooting

The current architecture documentation can be found at:

[`docs/architecture.md`](docs/architecture.md)

## Security

This is a public repository.

Never commit:

- Passwords
- API keys
- Database credentials
- Private keys
- `.env` files
- Personal information
- Sensitive farm information

Use environment variables for sensitive configuration.

The repository contains `.env.example` only as a reference for required configuration.

## Project Status

The QVFT Stack platform is currently under active development.

The team is developing the infrastructure required to connect farm hardware, data collection, backend services, database systems, and monitoring tools into one integrated platform.

## Contributing

QVFT members and external contributors are encouraged to use GitHub Issues and Pull Requests.

Before contributing, please read:

[`CONTRIBUTING.md`](CONTRIBUTING.md)

## Queen's Vertical Farming Team

Queen's University  
Kingston, Ontario, Canada

Instagram: @qvft_qu

Contact: director.qvft@engsoc.queensu.ca

## Open Source

QVFT believes that research and engineering knowledge surrounding vertical farming should be accessible.

By documenting and publishing our work, we hope future QVFT members and the wider community can learn from, improve, and build upon the systems we develop.
