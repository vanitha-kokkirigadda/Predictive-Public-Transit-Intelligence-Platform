# MetroFlow — AI Platform for Metro Crowd Management \& Scheduling

MetroFlow is an AI-powered metro crowd management and scheduling platform that helps metro
authorities monitor passenger flow, predict crowd density, and optimize train scheduling in
real time. It integrates AI analytics, scheduling automation, crowd prediction with **prediction
intervals**, operational monitoring, and **real-world dataset ingestion** into one
centralized application for smart transportation systems. The live demo runs on the **real New
York City subway network** — 59 genuine MTA stations spanning the 1/2/3, 4/5/6, 7, A/C/E,
B/D/F/M, L, N/Q/R/W trunk corridors and the Staten Island Railway — with a GTFS-derived
connection graph and a full-network geographic map (496 stations / 578 rail segments).

## Key Capabilities

|Module|Features|
|-|-|
|User Management|Admin/operator login, role-based access control (RBAC), profile management|
|Crowd Monitoring|Passenger density tracking (ticketing + sensor data), heatmaps, congestion monitoring, station-wise analytics, inflow/outflow analysis — the station history chart accepts any past date/start-time (windows older than the stored history get a synthesised daily curve, so it never renders empty)|
|Connection Mapping|**Real stations, real junctions**: a connection graph built from the official MTA GTFS feed (`/crowd/connections`, 25 edges across 59 NYC stations) plus a full-network geographic map — every one of the 496 real stations and 578 rail segments (`/crowd/network`) with monitored congestion overlaid|
|Train Monitoring|Live fleet position \& status per train (in-transit / at-station / delayed...), position %, next stop, ETA, headway, projected load — streamed via Socket.IO with per-train rooms|
|Scheduling Management|Train schedule CRUD, peak-hour optimization, frequency adjustment, delay handling — headway-driven timetables (4-min peak / 8-min off-peak) with each line's departure grid staggered so arrivals don't collapse onto the same minutes|
|AI Prediction|Crowd prediction \& demand forecasting for any chosen date/time, **with lower/upper prediction intervals** (quantile ensemble when the advanced Hangzhou artifact is loaded, symmetric residual band otherwise); per-train per-stop forecasts, traffic pattern analysis, smart recommendations|
|Alert \& Notification|Overcrowding alerts, delay notifications, emergency announcements, real-time updates (Socket.IO)|
|Analytics Dashboard|Traffic analytics, station performance reports, operational monitoring, AI insight panels|
|Dataset Importers|Ingest genuine MTA / Seoul Metro / TfL CSVs into `data/` and feed the whole seed pipeline (`scripts/importers/`)|

> No computer vision / CCTV processing is used — all models are trained on transportation and
> operational passenger datasets (smart-card ticketing, entry/exit records, footfall, ridership,
> GPS/status, occupancy, delay logs).

## Tech Stack

* **Backend:** Python 3.11+, FastAPI, SQLAlchemy 2, Pydantic v2, JWT auth, Socket.IO
* **Frontend:** Next.js 14 (React 18) — **fully TypeScript**, Tailwind CSS, Recharts, lucide-react, socket.io-client
* **Databases:** PostgreSQL (core relational), MongoDB (raw ticketing/sensor events), Redis (live cache)
* **AI/Analytics:** scikit-learn (GradientBoosting, incl. **quantile regression**), pandas, NumPy, joblib model store
* **DevOps:** Docker + Docker Compose (PostgreSQL/MongoDB/Redis/backend/frontend), AWS/Azure ready
* **Real-world topology:** official MTA GTFS feed → `connections.csv` (junction graph) and `nyc\_network.json` (full 496-station / 578-segment network for the geographic map)

## Repository Layout

```
MetroFlow/
├── backend/
│   ├── app/
│   │   ├── main.py               # FastAPI app + Socket.IO mount + public GET /api/v1/health
│   │   ├── core/                 # config, security (JWT), database sessions, time helpers
│   │   ├── models/               # SQLAlchemy models (users, stations, trains, schedules, alerts, ridership)
│   │   ├── schemas/              # Pydantic schemas (stations, connections, network, ...)
│   │   ├── api/v1/               # REST endpoints per module (incl. trains, predictions, crowd connections/network)
│   │   ├── services/             # business logic: crowd, scheduling, prediction, alerts, analytics, realtime, connection, network
│   │   └── ml/                   # feature engineering + model wrappers (59 real MTA stations, quantile intervals)
│   ├── scripts/
│   │   ├── generate\_data.py      # synthetic transportation datasets (CSV) -> data/
│   │   ├── train\_models.py       # trains legacy crowd + demand models -> models\_store/
│   │   ├── train\_quantile\_crowd.py   # trains the advanced quantile crowd model (3 models: q05/q50/q95)
│   │   ├── build\_connections.py  # MTA GTFS -> data/connections.csv (real junction graph: 25 edges)
│   │   ├── build\_network.py      # MTA GTFS -> data/nyc\_network.json (full network: 496 stations / 578 segments)
│   │   ├── seed\_db.py            # seeds stations/trains/users/schedules/history (headway-driven) + --refresh
│   │   └── importers/            # MTA / Seoul / TfL real-dataset importers -> data/ (cli.py entrypoint)
│   ├── data/                     # versioned datasets: stations.csv (59 NYC stops), connections.csv, nyc\_network.json
│   ├── models\_store/             # trained .joblib artifacts (legacy, {seoul,hangzhou,nyc}, nj delay, quantile crowd)
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── pages/                    # login + professional dashboard suite (TypeScript + Tailwind)
│   ├── components/               # layout, KPI cards, charts, heatmap, tables, modals, ConnectionList, MetroMap (schematic + geographic + connections)
│   ├── lib/                      # API client, auth context, socket client, lines/connections helpers
│   └── ...
├── docs/
│   ├── PROJECT\_PLAN.md           # master plan + API surface
│   ├── PROJECT\_GUIDE.md          # complete guided walkthrough (login → every page, data/ML story, screenshots)
│   ├── MILESTONE\_1..4.md         # weekly milestone reports
│   ├── DEPLOYMENT.md             # AWS / Azure / Kubernetes guides
│   ├── PERFORMANCE\_METRICS.md    # model + API benchmarks
│   ├── IMPORTERS.md              # real-world dataset ingestion guide
│   ├── ML\_MODELS.md              # model store, quantile intervals, training \& fallbacks
│   └── ...
├── deploy/k8s/metroflow.yaml     # Kubernetes manifests (all services)
├── postman/MetroFlow.postman\_collection.json
├── backend/tests/test\_api.py     # automated API test suite (pytest)
├── docker-compose.yml
└── README.md
```

## Quick Start (Local)

### 1\. Infrastructure

Start the databases (PostgreSQL, MongoDB, Redis):

```bash
docker compose up -d postgres mongo redis
```

### 2\. Backend

```bash
cd backend
python -m venv .venv
.\\.venv\\Scripts\\activate                    # Windows
pip install -r requirements.txt
copy .env.example .env                      # adjust if needed

python scripts/generate\_data.py             # create synthetic datasets in data/
python scripts/train\_models.py              # train legacy AI models (optional)
python scripts/train\_quantile\_crowd.py      # train the advanced quantile crowd model (optional, recommended)
python scripts/build\_connections.py --gtfs-dir <extracted mta gtfs>   # optional: rebuild data/connections.csv from the MTA GTFS feed
python scripts/build\_network.py --gtfs-dir <extracted mta gtfs>       # optional: rebuild data/nyc\_network.json (full 496-station network)
python scripts/seed\_db.py                   # seed stations, trains, users, schedules
python scripts/seed\_db.py --refresh         # anytime: roll the demo forward (7-day schedule+ridership history \& next-24h window; keeps user data)

# Model city. Default is nyc — the retrained models serve the 59 real MTA stations
# (crowd R² 0.981, demand R² 0.969; see models\_store/nyc\_train\_metrics.json).
set METROFLOW\_MODEL\_CITY=nyc               # or seoul | hangzhou | tfl | beijing

uvicorn app.main:socket\_app --reload --port 8000   # serves API + Socket.IO
```

API docs: http://localhost:8000/api/v1/docs

> macOS/Linux: use `source .venv/bin/activate` instead of `.\\\\.venv\\\\Scripts\\\\activate`,
> and `cp .env.example .env` instead of `copy`.

### 3\. Frontend

```bash
cd frontend
npm install
npm run dev
```

Dashboard: http://localhost:3000

### Demo Accounts (created by seed script)

|Role|Email|Password|
|-|-|-|
|Admin|admin@metroflow.io|Admin@123|
|Operator|operator@metroflow.io|Operator@123|
|Viewer|viewer@metroflow.io|Viewer@123|

## Ingesting Real-World Datasets

MetroFlow ships first-class importers for genuine transit-agency CSVs. Each importer parses the
real export format, normalizes it to the demo's hourly schema (`ridership\_hourly.csv` +
`stations.csv`), re-anchors it to *now* so the live dashboards stay fresh, and hands the result
straight to `seed\_db.py`:

```bash
cd backend

# MTA turnstile (the weekly open-data turnstile\_\*.txt cumulative counters)
python scripts/importers/cli.py --city mta --input path/to/turnstile\_240515.txt --output-dir data

# Seoul Metro card-swipe CSVs (English or Korean columns, hourly or daily totals)
python scripts/importers/cli.py --city seoul --input path/to/seoul\_metro.csv --output-dir data

# TfL Entry \& Exit annual counts
python scripts/importers/cli.py --city tfl --input path/to/tfl\_entry\_exit.csv --output-dir data

python scripts/seed\_db.py --refresh   # load the imported data into the database
```

Station lines and fleet fallbacks are handled automatically on refresh. Full documentation of
formats, column aliases, the normalization pipeline, and how to add a new city live in
[`docs/IMPORTERS.md`](docs/IMPORTERS.md).

## Real Connections \& Full-Network Map

The connection map is built from the **official MTA GTFS feed**, not hand-drawn art. Two derived
assets are versioned in `backend/data/`:

|Asset|Content|Endpoint|
|-|-|-|
|`connections.csv`|**Junction graph** between the 59 monitored stations — 25 real edges (21 consecutive-stop along-line links + 4 walking interchanges like the 42 St Shuttle and Queensboro Plaza same-concourse)|`GET /api/v1/crowd/connections`|
|`nyc\_network.json`|**Complete rail network** for the geographic base map — all 496 real MTA stations (real `stop\_lat`/`stop\_lon`) and 578 real segments (stops genuinely consecutive on the same trip), tagged with the 59 monitored codes|`GET /api/v1/crowd/network`|

### How real are the "NYC metro paths"?

Real stations, in real positions, connected in the real order — with one deliberate simplification
worth stating plainly:

* **Real:** the 496 station positions come straight from MTA `stops.txt` coordinates; the 578
segments are consecutive stops on the *same GTFS trip*, so the adjacency is the network's real
adjacency; each segment is tagged with the routes that actually serve it (`1;2;3`).
* **Simplified:** the line between two stops is drawn **straight**. MTA's optional `shapes.txt`
track polylines are not consumed, so curves and elevated/underground distinctions are not shown.
Stops sharing a `parent\_station` (the separate platforms inside one complex) collapse to a single
junction dot.

It is therefore a real-topology map with straight-line rendering — not surveyed track geometry, and
not a live GPS trace of vehicles. Full detail in `docs/PROJECT\_GUIDE.md` §4.1.

`scripts/build\_connections.py` matches GTFS `stop\_times` through the `parent\_station` map (platform
ids like `120N` → station `120`) and hard-codes two genuine-feed corrections: the phantom
`718↔R09` artifact is dropped and the same-complex transfers `718↔R05` / `127↔631` are added.
`scripts/build\_network.py` snapshots the full station/segment topology (monitored: 59/59 codes match).

On the Crowd dashboard all three views live in one toggle. The **schematic** view (one trunk per
track) is untouched; the **geographic** toggle draws the complete 496-station real network as a
faint base layer, overlays the 59 monitored stations with live congestion dots, and clicking a
station brightens exactly its one-hop real connections (with dashed walking interchanges); the
**connections** toggle uses the same true lat/lng projection but strips it back to the monitored
graph so the one-to-one links are unmistakable — click a station and the map **explodes into two
depth planes**: the selection and everything it links to rise onto a bright, glowing layer with
animated dashes, route chips (`L`, `4/5`), white rings and live occupancy badges, while every
other station and all of its remaining connection paths drop onto a thinner, blurred floor below,
tied to the raised layer by vertical risers so the junction is read in the context of the whole
network rather than in isolation. Only links genuinely incident to the selected station are lifted
(the tested `linkTouches` predicate), so a path that merely passes through a neighbour stays down
on the floor. With nothing selected the graph is a single flat plane.

## ML Models \& Confidence Intervals

`GET /predictions/crowd` (and `/predictions/demand`) returns `lower` / `upper` bounds for every
forecast point alongside the point prediction. With the advanced quantile artifact loaded
(`models\_store/hangzhou\_crowd\_quantile\_model.joblib`, trained by `train\_quantile\_crowd.py`), these
bounds come from an **ensemble of three GradientBoostingRegressors trained on quantile loss**
(α = 0.05 / 0.50 / 0.95) — a genuine uncertainty estimate. Legacy and synthetic artifacts fall
back to a symmetric residual-std band, so the API shape never changes.

* Model store, per-city selection (`METROFLOW\_MODEL\_CITY`), and training/fallback details:
[`docs/ML\_MODELS.md`](docs/ML_MODELS.md)
* Measured metrics (R² / MAE / coverage): [`docs/PERFORMANCE\_METRICS.md`](docs/PERFORMANCE_METRICS.md)

## Docker (Full Stack)

```bash
docker compose up --build
```

Services: `frontend` (:3000), `backend` (:8000), `postgres` (:5432), `mongo` (:27017), `redis` (:6379).

## Operational Resilience (no Docker / partial services)

The backend runs and degrades gracefully even when an optional dependency is
down — end users never see raw stack traces (DB failures return a clean `503`):

|Dependency down|Behavior|
|-|-|
|PostgreSQL|DB-backed endpoints return `503 {"detail": "Database is temporarily unavailable..."}`; realtime loop backs off exponentially (5s → 120s)|
|Redis|Auto-falls back to an in-memory cache, re-probes every 60s|
|MongoDB|Sensor-event storage is skipped/disabled; no impact on APIs|
|No Docker at all|Use SQLite: `DATABASE\_URL=sqlite:///./metroflow\_dev.db`, then `python scripts/seed\_db.py`|

`METROFLOW\_ENABLE\_REALTIME=0` disables the Socket.IO broadcast loop (used in tests/CI).

A public `GET /api/v1/health` endpoint backs the login page and Kubernetes probes — it is
**unauthenticated** and reports reachability without leaking data. The login screen pings it to
decide "online / offline" instead of probing an auth-protected route.

## Milestone Mapping (PRD)

* **Milestone 1 (Wk 1-2):** architecture, DB schema, auth + RBAC, crowd monitoring dashboard - [docs/MILESTONE\_1.md](docs/MILESTONE_1.md)
* **Milestone 2 (Wk 3-4):** datasets, AI crowd/demand models, backend integration - [docs/MILESTONE\_2.md](docs/MILESTONE_2.md)
* **Milestone 3 (Wk 5-6):** real-time monitoring, alerts, schedule optimization - [docs/MILESTONE\_3.md](docs/MILESTONE_3.md)
* **Milestone 4 (Wk 7-8):** analytics, testing, Docker/cloud deployment, docs - [docs/MILESTONE\_4.md](docs/MILESTONE_4.md)

See [docs/PROJECT\_GUIDE.md](docs/PROJECT_GUIDE.md) for a complete guided tour of the application — from the login page
through every dashboard, explaining data provenance, the ML techniques and where each is used, and
what every chart/table represents (with screenshots). For the full week-wise breakdown see
[docs/PROJECT\_PLAN.md](docs/PROJECT_PLAN.md), [docs/PERFORMANCE\_METRICS.md](docs/PERFORMANCE_METRICS.md)
for measured model/API benchmarks, and [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) for AWS/Azure/K8s deployment.

## Testing

```bash
cd backend
pip install -r requirements-dev.txt
pytest tests -v        # 179 tests (API, models, ML artifacts, connections, network, timezone, cache, scheduling advice, simulation)
```

Frontend: `cd frontend \&\& npm test` runs the Vitest suite (**72 tests** — lines, connections,
`linkTouches` one-hop link predicate, the `mapDepth` plane model plus a render-level suite that
asserts the connections view really stacks two planes, `CrowdEstimateNote`, `ModelBadge`, and the
NYC/IST timezone helpers).

CI runs on every branch push and pull request (see `.github/workflows/ci.yml`): a Python job runs
the backend suite with dev dependencies, a separate job verifies the backend Docker image builds,
and the frontend runs `lint`, `tsc`, unit tests and a production build.

## 

