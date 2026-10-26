# GridBricks — Project Development Log

> **Client**: UK Energy Utility — Day-Ahead Market Procurement Team  
> **Goal**: Web app to visualise wind power actuals vs. forecasts, and quantify forecast accuracy   
> **Started**: 2026-10-26

---

## Tech Stack (Finalised)

| Layer | Technology |
|---|---|
| Frontend | React (Vite) |
| Backend | Python (FastAPI) |
| Containerisation | Docker + Docker Compose |
| Frontend Deploy | Vercel |
| Backend Deploy | Heroku |
| Data Source | Elexon BMRS API |

---

## SDLC Model

**Lean Agile — Time-Boxed Single Sprint**

```
Phase 1 │ Requirements & Architecture   │ ~1.5h
Phase 2 │ System Design & API Contract  │ ~1.5h
Phase 3 │ Backend Development           │ ~4h
Phase 4 │ Frontend Development          │ ~5h
Phase 5 │ Integration & Dockerisation   │ ~2h
Phase 6 │ Deployment & Documentation    │ ~2h
```

---

## Phase 1 — Requirements & Architecture

**Status**: COMPLETE
**Date**: 2026-10-26

### 1.1 Stakeholder & Context

- **User**: Trading/procurement team at a UK energy utility
- **Pain point**: They must decide how much power to buy from the UK day-ahead market ONE DAY before delivery
- **Core dependency**: Wind generation forecasts (renewable output is weather-dependent and uncertain)
- **Data scope**: National-level wind power only, from Elexon BMRS, January 2026

---

### 1.2 User Stories (Epics -> Stories)

#### Epic 1 — Understand Latest Forecasts
"As a procurement analyst, I need to see the most recent wind generation forecasts so I can estimate additional power needed from thermal sources."

| ID | User Story | Priority |
|---|---|---|
| US-01 | View the latest wind generation forecast for each 30-min settlement period on a given day | Must |
| US-02 | Compare the latest forecast against all earlier forecasts for the same period (versioned forecasts) | Should |
| US-03 | Select any date in January 2026 to view its forecast timeline | Must |
| US-04 | Visually distinguish different forecast publication times on a chart | Must |

#### Epic 2 — Evaluate Forecast Accuracy
"As a procurement analyst, I need to quantify how accurate past forecasts have been so I can factor that uncertainty into my buy decisions."

| ID | User Story | Priority |
|---|---|---|
| US-05 | View actual vs. forecast generation overlaid on a single chart | Must |
| US-06 | See key error metrics: MAE, RMSE, MAPE per day | Must |
| US-07 | Compare accuracy across multiple days (trend of error over January 2026) | Should |
| US-08 | Filter by forecast horizon (e.g., 12h ahead, 24h ahead, 36h ahead) | Could |

---

### 1.3 MoSCoW Prioritisation Summary

| Priority | Items |
|---|---|
| Must | US-01, US-03, US-04, US-05, US-06 |
| Should | US-02, US-07 |
| Could | US-08 |
| Won't (this sprint) | Multi-fuel support, alerts/notifications, user auth |

---

### 1.4 Data Sources

#### Actuals — FUELHH
- Endpoint: https://bmrs.elexon.co.uk/api-documentation/endpoint/datasets/FUELHH/stream
- Filter: fuelType = "WIND"
- Key fields: startTime (30-min resolution), generation (MW)
- Scope: January 2026

#### Forecasts — WINDFOR
- Endpoint: https://bmrs.elexon.co.uk/api-documentation/endpoint/datasets/WINDFOR/stream
- Key fields: startTime, publishTime, generation (MW)
- Scope: January 2026

---

### 1.5 High-Level Architecture

```
+-------------------------------------------------------------+
|                        CLIENT (Browser)                      |
|                    React App (Vite) — Vercel                 |
|  +--------------+  +--------------+  +------------------+  |
|  |  Forecast    |  |  Actuals vs  |  |  Accuracy        |  |
|  |  Explorer   |  |  Forecast    |  |  Dashboard       |  |
|  |  (Chart)    |  |  Overlay     |  |  (MAE/RMSE)      |  |
|  +--------------+  +--------------+  +------------------+  |
+------------------------+------------------------------------+
                         | REST (JSON)
                         v
+-------------------------------------------------------------+
|               Python FastAPI Backend — Heroku                |
|  +------------------------------------------------------+   |
|  |  /api/forecasts     -> WINDFOR data (filtered/shaped)|   |
|  |  /api/actuals       -> FUELHH WIND data              |   |
|  |  /api/accuracy      -> MAE, RMSE, MAPE calculations  |   |
|  +------------------------------------------------------+   |
|  +------------------------------------------------------+   |
|  |  Elexon API Client  (httpx, async)                   |   |
|  |  In-memory cache (TTL: 5 min)                        |   |
|  +------------------------------------------------------+   |
+------------------------+------------------------------------+
                         | HTTPS
                         v
+-------------------------------------------------------------+
|                    Elexon BMRS API                           |
|    FUELHH (actuals)        WINDFOR (forecasts)               |
+-------------------------------------------------------------+
```

---

### 1.6 Non-Functional Requirements

| NFR | Requirement |
|---|---|
| Performance | Page load < 3s; API response < 5s (Elexon can be slow) |
| Availability | Heroku eco dyno; cold start acceptable |
| Caching | Backend caches Elexon responses (5 min TTL) to avoid rate limits |
| Responsiveness | UI works on desktop (1280px+); tablet acceptable |
| Security | No auth required (internal tool prototype); CORS locked to frontend domain |

---

### 1.7 Folder Structure (Planned)

```
gridbricks/
├── backend/
│   ├── app/
│   │   ├── main.py            # FastAPI app entry
│   │   ├── routers/
│   │   │   ├── forecasts.py
│   │   │   ├── actuals.py
│   │   │   └── accuracy.py
│   │   ├── services/
│   │   │   ├── elexon_client.py
│   │   │   └── metrics.py
│   │   └── models/
│   │       └── schemas.py
│   ├── Dockerfile
│   ├── requirements.txt
│   └── Procfile               # Heroku
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── services/          # API calls to backend
│   │   └── main.jsx
│   ├── Dockerfile
│   └── package.json
├── docker-compose.yml
├── PROJECT_LOG.md             # <- this file
└── README.md
```

---

## Phase 2 — System Design & API Contract

**Status**: COMPLETE
**Date**: 2026-10-26

---

### 2.1 Elexon API — Real Data Shapes (Verified Live)

#### WINDFOR (Forecasts) — Sample Record
```json
{
  "dataset": "WINDFOR",
  "publishTime": "2026-10-26T04:30:00Z",
  "startTime": "2026-10-26T06:00:00Z",
  "generation": 13295
}
```
- Resolution: 1-hour slots (startTime)
- publishTime: when forecast was published (multiple per startTime = versioned forecasts)
- generation: MW (integer)
- No pagination — stream endpoint returns full result set
- Query params: `from`, `to` (ISO8601), `format=json`

#### FUELHH (Actuals) — Sample Record
```json
{
  "dataset": "FUELHH",
  "publishTime": "2026-10-26T06:30:00Z",
  "startTime": "2026-10-26T06:00:00Z",
  "settlementDate": "2026-10-26",
  "settlementPeriod": 13,
  "fuelType": "WIND",
  "generation": 10624
}
```
- Resolution: 30-minute slots (startTime)
- settlementPeriod: 1..48 (48 per day)
- publishTime ≈ startTime + 30 min (near-real-time actual)
- Must filter by `fuelType=WIND`
- Query params: `from`, `to` (ISO8601), `fuelType=WIND`, `format=json`

#### Key Observations
| Item | Observation |
|---|---|
| WINDFOR resolution | 1-hour (startTime steps by 1h) |
| FUELHH resolution | 30-minute (startTime steps by 30m) |
| Alignment strategy | Aggregate FUELHH to 1-hour by averaging adjacent 30-min periods |
| Multiple forecasts | WINDFOR returns all publishTime versions for same startTime — pick latest for "current forecast" |
| Jan 2026 date range | `from=2026-01-01T00:00:00Z&to=2026-02-01T00:00:00Z` |

---

### 2.2 Backend API Contract (FastAPI)

Base URL: `https://<heroku-app>.herokuapp.com`

#### GET /api/forecasts
Returns all wind generation forecasts for a given delivery date, grouped by publishTime.

**Request**
```
GET /api/forecasts?date=2026-01-15
```
| Param | Type | Required | Description |
|---|---|---|---|
| date | string (YYYY-MM-DD) | Yes | The delivery date to fetch forecasts for |

**Response**
```json
{
  "date": "2026-01-15",
  "forecasts": [
    {
      "publishTime": "2026-01-14T10:30:00Z",
      "isLatest": true,
      "series": [
        { "startTime": "2026-01-15T00:00:00Z", "generation": 4200 },
        { "startTime": "2026-01-15T01:00:00Z", "generation": 4350 }
      ]
    }
  ]
}
```

#### GET /api/actuals
Returns actual wind generation for a given delivery date (30-min resolution, aligned to hours).

**Request**
```
GET /api/actuals?date=2026-01-15
```

**Response**
```json
{
  "date": "2026-01-15",
  "actuals": [
    { "startTime": "2026-01-15T00:00:00Z", "generation": 4150 },
    { "startTime": "2026-01-15T01:00:00Z", "generation": 4400 }
  ]
}
```

#### GET /api/accuracy
Returns error metrics (MAE, RMSE, MAPE) comparing latest forecast vs actuals, per day.

**Request**
```
GET /api/accuracy?from=2026-01-01&to=2026-01-31
```

**Response**
```json
{
  "from": "2026-01-01",
  "to": "2026-01-31",
  "daily": [
    {
      "date": "2026-01-01",
      "mae": 312.5,
      "rmse": 418.2,
      "mape": 7.4,
      "n_points": 24
    }
  ],
  "summary": {
    "mae": 298.1,
    "rmse": 401.7,
    "mape": 6.9
  }
}
```

#### GET /health
Simple liveness check for Heroku.
```json
{ "status": "ok" }
```

---

### 2.3 React Component Tree

```
App
├── Layout
│   ├── Header (logo, nav)
│   └── Sidebar (date picker, controls)
├── Pages
│   ├── ForecastExplorer (route: /)
│   │   ├── DateSelector
│   │   ├── ForecastVersionList (publish times, toggle visibility)
│   │   └── ForecastChart (Recharts LineChart)
│   │       ├── Each publishTime → coloured line
│   │       └── Tooltip with generation value + publish time
│   └── AccuracyDashboard (route: /accuracy)
│       ├── DateRangePicker
│       ├── MetricsSummaryCards (MAE / RMSE / MAPE)
│       ├── ActualsVsForecastChart (LineChart — 2 series)
│       └── DailyErrorChart (BarChart — MAE per day over Jan 2026)
└── shared/
    ├── LoadingSpinner
    ├── ErrorBanner
    └── StatCard
```

---

### 2.4 State Management

- **Local state** (useState/useEffect) for each page — no Redux needed
- **React Query (TanStack Query)** for data fetching, caching, and loading states
- **URL-driven state**: selected date stored in query params (`?date=2026-01-15`) for shareability

---

### 2.5 Metrics Definitions

| Metric | Formula | Unit |
|---|---|---|
| MAE | mean(|forecast - actual|) | MW |
| RMSE | sqrt(mean((forecast - actual)^2)) | MW |
| MAPE | mean(|forecast - actual| / actual) * 100 | % |

All computed server-side in `services/metrics.py`.


---

## Phase 3 — Backend Development

**Status**: PENDING

---

## Phase 4 — Frontend Development

**Status**: PENDING

---

## Phase 5 — Integration & Dockerisation

**Status**: PENDING

---

## Phase 6 — Deployment & Documentation

**Status**: PENDING

---

## Decision Log

| Date | Decision | Rationale |
|---|---|---|
| 2026-10-26 | FastAPI over Flask | Async support (better for Elexon API calls), auto-generated OpenAPI docs |
| 2026-10-26 | Vercel for frontend | Zero-config React/Vite deployment |
| 2026-10-26 | Heroku for backend | Simple Docker container deployment, eco tier sufficient |
| 2026-10-26 | In-memory cache (no Redis) | Keeps stack simple; Elexon data does not change frequently |
| 2026-10-26 | Recharts for charts | Lightweight, React-native, good time-series support |

---

## Issues & Blockers

None yet.

---

*This log is auto-maintained throughout the SDLC process.*
