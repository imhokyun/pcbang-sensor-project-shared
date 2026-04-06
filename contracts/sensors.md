# Sensor Data Contracts

## MQTT Topic Structure

```
pcbang/sensors/{sensor_id}/{data_type}
```

## Sensor Data Format (JSON)

```json
{
  "sensor_id": "string",
  "timestamp": "ISO8601",
  "type": "temperature | humidity | motion | co2",
  "value": "number",
  "unit": "string"
}
```

## Sensor Types

| type        | unit  | range        |
|-------------|-------|--------------|
| temperature | °C    | -10 ~ 60     |
| humidity    | %     | 0 ~ 100      |
| motion      | bool  | 0 or 1       |
| co2         | ppm   | 0 ~ 5000     |

---
_Update this file when adding new sensor types or changing data format._
