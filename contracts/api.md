# API Contracts

## Backend REST API

Base URL: `http://backend:8080/api/v1`

### Endpoints

| Method | Path                    | Description              |
|--------|-------------------------|--------------------------|
| GET    | /sensors                | List all sensors         |
| GET    | /sensors/{id}/data      | Get sensor readings      |
| GET    | /sensors/{id}/latest    | Get latest reading       |
| POST   | /sensors/{id}/data      | Ingest sensor data       |
| GET    | /health                 | Health check             |

### Response Format

```json
{
  "success": true,
  "data": {},
  "error": null
}
```

## MQTT → Backend Bridge

- Edge publishes to MQTT broker
- Backend subscribes and persists to DB
- Frontend subscribes for real-time updates via WebSocket

## WebSocket (Frontend)

`ws://backend:8080/ws`

Events: `sensor.update`, `sensor.alert`

---
_Update this file when adding or changing API endpoints._
