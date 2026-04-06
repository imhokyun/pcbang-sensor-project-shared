# System Architecture

## Overview

```
[Sensors / Devices]
        ↓ MQTT publish
[Edge - Home Assistant + go2rtc + Mosquitto MQTT Broker]
        ↓ MQTT → HTTP/WebSocket
[Backend - API Server + DB]
        ↓ REST API + WebSocket
[Frontend - Dashboard UI]
```

## Components

### Edge (`edge/`)
- **Home Assistant**: Device automation and sensor integration
- **go2rtc**: Camera stream handling
- **Mosquitto**: MQTT broker for sensor data relay
- Runs on Raspberry Pi or local edge device

### Backend (`backend/`)
- REST API server
- MQTT subscriber → DB writer
- WebSocket server for real-time frontend updates
- Database: (TBD)

### Frontend (`frontend/`)
- Dashboard UI
- Displays real-time sensor data
- Connects to backend via REST + WebSocket

## Network

All services communicate within Docker network.
See `docker-compose.yml` in project root.

## Shared Submodule

Each component includes this repo as `shared/` submodule.
See `contracts/` for data format agreements between components.
