# AFRICA RESQ — AI Fire & Smoke Detection System (Egypt)

## Overview

AFRICA RESQ is an AI-powered fire and smoke detection system designed to support
emergency response and search-and-rescue operations in **Egypt**.

The system combines computer vision, hazard analysis, risk assessment, survivor
detection, **nearest emergency-shelter search** and decision-support components,
all exposed through a FastAPI backend that feeds a real-time **dashboard**.

> **One repo, two builds.** This repository hosts the whole project: the AI /
> data-engineering / API backend in the repo root, and the fully-built command
> dashboard (its own frontend, server and rover simulator) under
> [`dashboard/`](dashboard/). See [Project Structure](#project-structure).

## What's inside (hackathon prototype)

| Feature | Description |
|---|---|
| 🏠 Nearest shelter search | Real shelter dataset for **13 Egyptian cities** (Cairo, Giza, Alexandria, Port Said, Ismailia, Suez, Luxor, Aswan, Sharm El-Sheikh, Hurghada, Mansoura, Tanta, New Admin Capital). Distances (haversine), direction, walk/drive estimates, city filter. |
| 📡 Fake sensor simulator | Continuous simulated telemetry (temperature, gas, smoke density, humidity, wind, battery, GPS, fire/smoke/survivor detections, risk, priority, recommendation). Runs automatically so dashboards always have live data. |
| 🔌 API contract for the dashboard | The backend exposes every endpoint the frontend dashboard team needs (`/api/nearest_shelter`, `/api/sensor`, `/api/full`, `/api/route`, ...). The frontend dashboard itself is built by the frontend teammate — a basic reference page is served at `/map`. |
| 🗄️ Data engineering | Every detection — from the camera (YOLO) **and** the fake sensor simulator — is automatically persisted to a SQLite store (`data/africa_resq.db`) plus an append-only JSONL export (`data/events.jsonl`), queryable via `/api/events*`. |
| 🚁 Perception (optional) | YOLO fire/smoke detection pipeline (`src/perception.py`, `live_pipeline.py`, `webcam.py`) for real camera input. |

## Demo

The command dashboard in [`dashboard/`](dashboard/) runs against a rover simulator, so the whole chain (camera, thermal, fusion, risk, safe route) can be seen without hardware. The screenshot below is the dashboard on simulated sensor data.

![AFRICA RESQ command dashboard on simulated data](dashboard/og.jpg)

What the dashboard shows in that frame: a person detected by the camera at 84% and cross-checked against thermal, a risk score of 33/100 (medium) broken down by cause (carbon monoxide, combustible gas, heat, particulates, survivor in D4), and a recommended next action with the safest route.

### Simulator run (from `dashboard/README.md`)

| t | what happens |
|---|---|
| 0 s | rover enters at A1, sweeps east along row 1 |
| 15 s | turns north at D1 and holds to observe |
| 17 s | camera picks up something person-shaped, far off, logged as an unconfirmed contact |
| 22 s | thermal array finishes warming, finds a heat signature in the same place |
| 27 s | fire in D3, directly between the rover and the contact; the gas front arrives |
| 34 s | D3 is now impassable, the planner offers the long way round |
| 40 s | rover takes the safe corridor, east then north |
| 62 s | closes on the survivor; confidence rises as the range falls |
| 70 s | a second, partly buried contact appears to the west, and ranks below the first |

### Rover design

A low-cost tracked rover sketch (LiDAR, camera, GPS, gas and temperature sensors, Raspberry Pi) is in [`design/`](design/).

![AFRICA RESQ rover sketch](design/rover_sketch.png)

## System Architecture

```text
Camera / Sensors
       ↓
Perception (optional YOLO)
       ↓
Fusion / Hazard / Survivor Analysis
       ↓
Risk / Priority / Safe Path
       ↓
FastAPI Backend  ←——  Fake Sensor Simulator (prototype)
       ↓
Frontend Dashboard (static/map.html)
```

## Quick Start (backend + dashboard)

```powershell
# 1. Create a virtual environment (Python 3.10+)
python -m venv .venv
.venv\Scripts\Activate.ps1

# 2. Install dependencies
pip install -r requirements-api.txt

# 3. Start the server
python run_api.py
```

Then open:

- **API docs:**  http://localhost:8000/docs
- **Health:**    http://localhost:8000/api/health
- **Reference page:** http://localhost:8000/map (basic page — the real dashboard
  is built by the frontend teammate)

> The fake sensor simulator starts automatically with the server. `state.json`
> at the project root is continuously updated with live telemetry.

## Data Engineering

Every time the system **detects something**, the event is persisted automatically:

- **Camera / YOLO pipeline** (`src/perception.py`) → logs `fire`, `smoke`,
  `survivor`, `person` detections with confidence + bounding box (throttled to
  ~1 row per class every 2s so a constant fire doesn't flood the store).
- **Fake sensor simulator** (`src/simulator.py`) → logs a `telemetry` record
  every tick plus one record per active detection.

Storage is dual-purpose and requires **no extra dependencies** (stdlib only):

```
data/africa_resq.db    # SQLite - queryable, indexed (type, ts, source)
data/events.jsonl      # append-only JSON lines - ready for ETL / pipelines
```

Schema of each event row:

```json
{
  "id": 1,
  "event_type": "fire",
  "source": "camera_yolo",
  "detected": true,
  "confidence": 0.92,
  "latitude": 30.0444,
  "longitude": 31.2357,
  "payload": {"zone": "B3", "bbox": [1, 2, 3, 4]},
  "ts": 1760000000.0,
  "ts_iso": "2026-08-28T10:00:00"
}
```

Example:

```http
GET /api/events/stats
GET /api/events?type=fire&start=2026-08-28&limit=100
POST /api/events   # ingest from external systems
```

You can enable/disable event throttling with the `DATA_THROTTLE` env var
(seconds between identical events, default `2.0`; set `0` to record everything).

### Connecting the dashboard (frontend teammate)

The backend exposes everything the dashboard needs over HTTP + Server-Sent
Events — **no polling required** for live data:

```http
GET /api/events/latest     # last events + latest snapshot + counts
GET /api/events/stats      # aggregated counts (by type / source / detected)
GET /api/events/:type,source,start,end,limit,offset   # query history
GET /api/events/export     # dump every stored event
GET /api/events/stream     # LIVE feed (SSE) - pushed detections + state
GET /api/sensor            # latest fake sensor snapshot
GET /api/full              # complete current system state
POST /api/events           # ingest external detections
```

Live feed example (browser):

```javascript
const stream = new EventSource('http://localhost:8000/api/events/stream');

stream.onmessage = (e) => {
    const msg = JSON.parse(e.data);
    // msg.kind === 'hello'  -> connection greeting + stats
    // msg.kind === 'event'  -> freshly stored detection/telemetry row
    // msg.kind === 'state'  -> latest full sensor snapshot
    if (msg.kind === 'event' && msg.event_type === 'fire') {
        drawFireMarker(msg.latitude, msg.longitude, msg.confidence);
    }
};
```

The event rows use the schema shown above (`event_type`, `source`, `detected`,
`confidence`, `latitude`, `longitude`, `payload`, `ts_iso`), so the dashboard
can plot markers, timeline charts and risk stats directly from them.

Also create events from the frontend when a camera pipeline is unavailable:

```http
POST /api/events
Content-Type: application/json

{ "type": "fire", "source": "dashboard", "detected": true,
  "confidence": 0.9, "latitude": 30.0477, "longitude": 31.2336,
  "payload": {"zone": "A1"} }
```

### One-click start (Windows)

Double-click `start_backend.bat`.

### To use real camera detection instead

```powershell
pip install -r requirements.txt
cd src
python live_pipeline.py      # or python webcam.py
```

## API Endpoints

| Endpoint | Purpose |
|---|---|
| `GET /` | System info + available endpoints |
| `GET /api/status` | Backend status + timestamp |
| `GET /api/health` | Health check (incl. simulator state) |
| `GET /api/full` | Complete system state (detections, risk, sensors, history) |
| `GET /api/detection` | Fire / smoke / survivor detections |
| `GET /api/risk` | Current risk score + level |
| `GET /api/priority` | Rescue priority |
| `GET /api/location?latitude&longitude` | Validate a position is inside Egypt |
| `GET /api/route?start_lat&start_lon&end_lat&end_lon` | Driving route (OSRM, with straight-line fallback) |
| `GET /api/shelters?city=` | List shelters (all, or one city) |
| `GET /api/shelters/cities` | List cities that have shelters |
| `GET /api/nearest_shelter?latitude&longitude&limit=` | Nearest shelters, sorted by distance |
| `GET /api/shelter/{shelter_id}` | Shelter details by id |
| `GET /api/sensor` | Latest fake sensor snapshot |
| `GET /api/sensor/history?limit=` | Recent sensor history |
| `POST /api/events` | Ingest a detection event (body: type, source, detected, confidence, latitude, longitude, payload) — used by camera pipeline / external systems |
| `GET /api/events?type&source&start&end&limit&offset` | Query stored events |
| `GET /api/events/latest` | Latest events + current snapshot + counts |
| `GET /api/events/stats` | Aggregated stats (counts by type / source / detected) |
| `GET /api/events/export` | Export every stored event as JSON |
| `GET /api/events/stream` | Live feed (SSE) — pushes new events + state, no polling |
| `POST /api/simulation/start` | Start the fake sensor simulator |
| `POST /api/simulation/stop` | Stop the fake sensor simulator |
| `GET /api/simulation/status` | Simulator state |
| `POST /api/incident?latitude&longitude` | Recenter the simulation on an incident |
| `GET /map` | Dashboard page |
| `GET /static/...` | Static assets |

## Example — nearest shelter

```http
GET /api/nearest_shelter?latitude=30.0477&longitude=31.2336&limit=3
```

```json
{
  "success": true,
  "latitude": 30.0477,
  "longitude": 31.2336,
  "nearest": [
    {
      "shelter": {
        "id": "CAI-03",
        "name": "Tahrir Square (Egyptian Museum)",
        "city": "Cairo",
        "type": "open_square",
        ...
      },
      "distance_km": 0.31,
      "direction": "SE",
      "estimated_walk_minutes": 4,
      "estimated_drive_minutes": 1
    }
  ]
}
```

## Project Structure

```text
AFRICA-RESQ/
├── run_api.py              # server launcher
├── start_backend.bat       # one-click Windows start
├── requirements-api.txt    # lightweight backend deps (deploy)
├── requirements.txt        # full pipeline deps (perception)
├── state.json              # live system state (updated by simulator)
├── pytest.ini
├── conftest.py             # makes `src` importable in tests
├── src/
│   ├── api.py              # FastAPI backend
│   ├── shelter_data.py     # real Egyptian shelter dataset
│   ├── shelter_engine.py   # nearest-shelter search (haversine)
│   ├── fake_sensor.py      # simulated telemetry engine
│   ├── simulator.py        # background simulator thread
│   ├── location_engine.py  # Egypt geo-validation
│   ├── route_engine.py     # OSRM routing + fallback
│   ├── data_engine.py      # persistence (SQLite + JSONL event store)
│   ├── event_bus.py        # live pub/sub (SSE push to dashboards)
│   ├── safe_path.py / safe_route_engine.py / zone_mapper.py / hazard_grid.py
│   ├── perception.py / live_pipeline.py / webcam.py   # YOLO pipeline
│   ├── fusion.py / survivor.py / thermal.py / sensor_data.py
│   ├── hazard_engine.py / risk_engine.py / priority_engine.py
│   └── recommendation_engine.py / shared_state.py / main_pipeline.py
├── static/
│   └── map.html            # dashboard prototype
├── model/
│   └── best.pt             # YOLO fire/smoke model
├── dashboard/              # command dashboard app (self-contained, see its own README)
│   ├── index.html          # live rescue command dashboard (frontend)
│   ├── server/             # command server (uvicorn main:app)
│   ├── rover/              # rover simulator shown in the dashboard
│   ├── models/             # fire_smoke + yolov8n (.pt / .onnx)
│   └── Dockerfile, docker-compose.yml, deploy/ ...
└── Tests/                  # pytest suite (68 tests)
```

## Testing

```powershell
pip install -r requirements-api.txt
python -m pytest
```

47 tests run against the detection engines, shelter search, location validation,
the fake sensor simulator and the full API (including live sensor telemetry).

## Deployment notes (for the backend teammate)

- The API only needs the `requirements-api.txt` packages.
- The dashboard is fully client-side (`static/map.html`); it only needs a route
  to the API. When serving from a different domain, enable CORS (already
  configured to `*`).
- The fake sensor simulator can be disabled with
  `POST /api/simulation/stop` or re-centered with `POST /api/incident`.
- OSRM routing uses the public service (rate-limited for demos); the backend
  automatically falls back to a straight-line estimate if it is unreachable.

## Technologies

- Python / FastAPI / uvicorn
- Leaflet + OpenStreetMap (dashboard)
- OSRM public routing API
- YOLO / Ultralytics / OpenCV (optional perception pipeline)

## Team Project

AFRICA RESQ is developed as an AI-powered decision-support system for emergency
response and search-and-rescue operations, hackathon build targeting Egypt.
