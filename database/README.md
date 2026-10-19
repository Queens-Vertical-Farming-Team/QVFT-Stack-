# Database

The database stores and organizes the operational data collected by the Queen's Vertical Farming Team's automated farm.

## Responsibilities

The database will be responsible for storing:

- Sensor readings
- Device information
- Farm zones and plots
- Environmental measurements
- System status
- Alerts and warnings
- Historical farm data
- Maintenance records
- User-generated notes where appropriate

## Planned Technologies

- PostgreSQL
- Supabase

## Example Data

The database may store information such as:

- Air temperature
- Relative humidity
- pH
- Electrical conductivity (EC)
- Water temperature
- Reservoir level
- Soil or root-zone moisture
- Light measurements
- Device health
- Timestamps
- Zone or plot identifiers

## Example Sensor Record

```json
{
  "device_id": "qvft-zone-01",
  "sensor": "air_temperature",
  "value": 23.4,
  "unit": "C",
  "zone": "growth_zone_1",
  "timestamp": "2026-10-19T20:00:00Z"
}
