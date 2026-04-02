# Apex SAPIENT Middleware – AI Coding Instructions

## Project Overview
Apex is a Python middleware (v4.x) implementing the SAPIENT standard (BSI Flex 335). It routes, validates, archives, and optionally converts messages between:
- **Child nodes (ASMs/Edge Nodes)** – sensor systems connecting on configurable ports
- **Parent/Peer nodes (FN/DMM)** – fusion/decision-management nodes
- **Recorders** – passive message loggers

## Architecture: Key Components

| Package | Role |
|---|---|
| `sapient_apex_server/` | Core TCP server, message parsing, routing, SQLite persistence |
| `sapient_apex_api/` | FastAPI REST server + Elasticsearch interface for message querying |
| `sapient_apex_gui/` | Optional PySide6 desktop GUI (`apex_gui` entry point) |
| `sapient_apex_replay/` | Replay recorded SQLite databases to a live system |
| `sapient_msg/` | Protobuf definitions for all SAPIENT protocol versions |
| `sapient_apex_qt_helpers/` | Reusable Qt widget utilities |

**Entry points** (see `pyproject.toml`):
- `apex` → `sapient_apex_server/apex.py:serve_apex`
- `apex_gui` → `sapient_apex_gui/apex_gui.py:main`
- `apex_replay` → `sapient_apex_replay/replay.py:main`

## Data Flow
1. `apex.py` reads `apex_config.json`, starts `ApexServer` (trio async) + `SqliteThread` (dedicated OS thread) + FastAPI/uvicorn (separate thread).
2. Per TCP connection, `ApexServer.serve()` reads size-prefixed binary frames or newline-delimited XML, then calls `parse_proto` / `parse_xml`.
3. Parsed `MessageRecord` objects are passed to `ConnectionCreator`-managed `Connection` subclasses (`ChildConnection`, `ParentConnection`, etc.) in `connection.py` for routing logic.
4. Messages are forwarded via `ConnectionWriter` (wraps `BufferedWriter` from `trio_util.py`) which handles on-the-fly protocol version translation (`message_io.py`).
5. `SqliteThread` serialises all records into SQLite via SQLAlchemy ORM models in `sqlite_schema.py`.
6. If Elasticsearch is enabled, `sapient_apex_api/manager.py` also indexes received messages.

## Protocol Versions & `sapient_msg`
- Three versions: `VERSION6` (XML only), `BSI_FLEX_335_V1_0`, `BSI_FLEX_335_V2_0`.
- Proto bindings live under `sapient_msg/<version>/`. **Always import from `sapient_msg.latest.*`** in application code to stay version-agnostic.
- After modifying `.proto` files, regenerate bindings: `poetry run python run_protoc.py` (or `pre-commit run protobuf --all`).
- Cross-version translation is in `sapient_apex_server/translator/`: `proto_to_proto_translator.py` (v1↔v2), `bsi_flex_v1_to_xml.py` (proto→XML), `xml_to_bsi_flex_v1.py` (XML→proto).

## Configuration (`apex_config.json`)
- Each entry in `"connections"` has `type` (`Child`/`Peer`/`Parent`/`Recorder`), `port`, `format` (`XML`/`PROTO`), `icd_version`, and optional `outbound`/`forwardAll`.
- `"enableMessageConversion": true` lets nodes on different protocol versions interoperate.
- `"autoAssignSensorIDInRegistration"` controls whether Apex rewrites sensor IDs on registration.
- Elasticsearch is opt-in via `"elasticConfig.enabled": true`; REST API runs on `apiConfig.host:port` (default `127.0.0.1:8080`).

## Developer Workflows

### Setup
```bash
python -m venv venv && source venv/bin/activate
poetry install --all-extras   # installs all deps including PySide6 GUI
pre-commit install             # enforces black, flake8, proto regeneration on commit
```

### Run
```bash
apex                           # start middleware (reads apex_config.json)
apex_gui                       # start optional desktop GUI
cd tests && docker compose up  # start local Elasticsearch + Kibana for API tests
```

### Test
```bash
pytest                         # all tests (trio_mode=true; --import-mode=importlib)
pytest tests/test_routing.py   # integration routing tests (spin up real ApexMain)
pytest tests/test_msg_parsing.py  # unit tests for proto parse/validate
```
Tests use `pytest-trio` with `trio_mode = true` (set in `pyproject.toml`). The `conftest.py` fixture `server` spins up a real `ApexServer` against `MockStream` objects — prefer this pattern for new server-level tests.

### Linting / Formatting
```bash
black --check .                # line-length=100, target py39
flake8 .
```

## Key Conventions
- **Concurrency model**: trio async for all network I/O; a separate `SqliteThread` (with `threading.Condition`) for all DB writes; FastAPI runs in its own thread.  Do not mix asyncio with trio.
- **`MessageRecord`** is the central data structure flowing through the pipeline — see `structures.py`.
- **Error taxonomy** in `structures.py`: `SilentError` (log only), `NoisyError` (send error back), `FatalError` (close connection).
- **Proto wire format**: 4-byte big-endian size prefix followed by serialised protobuf bytes (see `receive_size_prefixed` in `trio_util.py`). XML uses newline delimiting.
- **ID generation**: `IdGenerator` in `translator/id_generator.py` manages node ID assignment and is shared across the server instance.
- **Validation** is opt-in per `ValidationType` enum and configured via `"validationOptions"` in `apex_config.json`; see `validate_proto.py`.
- Code style: `Copyright (c) 2019-2024 Roke Manor Research Ltd` header on all source files.
