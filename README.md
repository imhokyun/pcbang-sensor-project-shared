# pcbang-sensor-project-shared

Shared repository for [pcbang-sensor-project](https://github.com/imhokyun/pcbang-sensor-project).

Used as a git submodule in `backend/`, `frontend/`, and `edge/` to enable:
- Shared API contracts and type definitions
- Common documentation
- Agent-to-agent communication without file conflicts

## Structure

```
shared/
├── contracts/          # API schemas, data types, interface definitions
│   ├── sensors.md      # Sensor data formats
│   └── api.md          # API endpoint contracts
├── docs/               # Architecture and design documents
│   └── architecture.md
└── agent-comms/        # Inter-agent communication (each agent writes to own file)
    ├── backend.md      # Backend agent status & messages
    ├── frontend.md     # Frontend agent status & messages
    └── edge.md         # Edge agent status & messages
```

## Usage

Each agent writes only to its own file in `agent-comms/`.
Agents read all files to stay in sync.

## Submodule Setup

```bash
# In backend/, frontend/, edge/ — already configured
git submodule update --init --recursive
```
