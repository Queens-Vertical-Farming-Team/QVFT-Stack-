# Frontend

The frontend contains the user-facing dashboard for the Queen's Vertical Farming Team's farm monitoring and automation platform.

## Responsibilities

The frontend will be responsible for:

- Displaying live and historical farm data
- Showing sensor readings and trends
- Visualizing environmental conditions
- Monitoring device and system status
- Displaying alerts and warnings
- Providing clear views of farm zones and plots
- Supporting operators and team members with decision-making tools

## Planned Technologies

- React
- TypeScript
- JavaScript
- HTML
- CSS
- Chart.js, Recharts, or a similar visualization library

## Planned Dashboard Features

The dashboard may include:

- Temperature monitoring
- Relative humidity monitoring
- Soil or root-zone moisture data
- pH readings
- Electrical conductivity (EC)
- Reservoir information
- Device health
- Historical graphs
- Alert history
- Farm status summaries

## Data Flow

```text
PostgreSQL / Supabase
        ↓
FastAPI Backend
        ↓
REST API
        ↓
React Dashboard
