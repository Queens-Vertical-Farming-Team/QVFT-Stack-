# Sample Data

This folder contains public, non-sensitive example data for testing and demonstrating the Queen's Vertical Farming Team Stack system.

## Purpose

Sample data allows contributors to develop and test the dashboard, backend, database, and analytics tools without requiring access to the live farm.

This helps keep development safe, reproducible, and accessible to external contributors.

## Example Data

Sample datasets may include:

- Air temperature
- Relative humidity
- pH
- Electrical conductivity (EC)
- Water temperature
- Reservoir level
- Moisture readings
- Light measurements
- Device status
- Timestamps
- Zone or plot identifiers

## Example Record

```json
{
  "device_id": "qvft-zone-01",
  "sensor": "air_temperature",
  "value": 23.4,
  "unit": "C",
  "zone": "growth_zone_1",
  "timestamp": "2026-10-19T20:00:00Z"
}
