# Perf-Platform

> A self-hosted, full-stack performance testing platform built on Apache JMeter — manage, execute, and monitor load tests from a unified web UI without touching the command line.

---

## Overview

Perf-Platform replaces the manual JMeter CLI workflow with a modern web application. Upload your JMX files, configure thread groups and variables through the UI, run tests locally or across distributed Docker slaves, and watch results stream in real time — all from a single dashboard.

Built as a personal project to solve real pain points encountered in enterprise performance testing at Telus Digital.

---

## Features

### Execution Engine
- **Local execution** — run JMeter tests directly on the host machine
- **Distributed execution** — master-slave mode with Docker-based load generators
- **Parallel execution** — run multiple JMX files simultaneously, each on dedicated slaves
- **Sequential execution** — chain multiple JMX files in a defined order
- **Live log streaming** — real-time JMeter output via Server-Sent Events (SSE)

### JMX Configuration
- Parse JMX files and expose editable parameters (thread groups, ramp-up, duration, loops, variables, timers, CSV paths)
- Patch and deploy JMX per-run without modifying the source file
- Support for Standard, Stepping, Ultimate, BlazeMeter Concurrency thread group types

### Real-Time Monitoring
- Live KPI cards: active threads, throughput (req/s), average response time, error rate
- Over-time charts: response time (avg, P50, P90, P95, P99), throughput, active threads, error rate
- Response time histogram with colour-coded buckets (0–200ms → >5s)
- Per-transaction breakdown table with full percentile stats
- Error feed and error grouping by failure message
- Response code distribution

### Data Setup
- Verify and upload CSV test data files to local machine or distributed Docker slaves
- Per-JMX CSV slave assignment in multi-JMX parallel runs
- Slave conflict detection — blocks start if two JMX files would connect to the same slave endpoint

### Scheduling
- One-time scheduled runs (date/time picker, IST timezone)
- Recurring cron-based schedules with presets
- Schedule management dashboard with enable/disable/delete

### Projects & Profiles
- Create projects with saved execution profiles (JMX files, slave assignments, run mode)
- Run profiles directly from the project dashboard
- Per-JMX slave assignments stored per profile for parallel distributed runs

### Reporting
- Automatic HTML report generation (JMeter dashboard) after each run
- Historical run list with KPI summaries (total samples, error rate, avg response, throughput, peak threads)
- Run sharing between team members

### Load Generator Monitoring
- Real-time CPU, RAM, and JVM heap stats for Docker slave containers
- Per-slave status (online/offline) with container name mapping

### Multi-Tenancy
- Organisation-based access control
- User roles: Platform Admin, Org Admin, User
- Seat limits per organisation

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Ant Design, Recharts, Vite |
| Backend | FastAPI (Python 3.13), SQLAlchemy, APScheduler |
| Database | PostgreSQL |
| Test Engine | Apache JMeter 5.6.x |
| Load Generators | Docker (custom JMeter server image) |
| Auth | JWT (access + refresh tokens) |

---

## Architecture

```
┌─────────────────────────────────────────────┐
│              Browser (React UI)             │
└────────────────────┬────────────────────────┘
                     │ REST + SSE
┌────────────────────▼────────────────────────┐
│           FastAPI Backend (Python)          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │Execution │  │Monitoring│  │Scheduler │  │
│  │  API     │  │  API     │  │  API     │  │
│  └────┬─────┘  └────┬─────┘  └──────────┘  │
│       │             │                       │
│  ┌────▼─────────────▼────────────────────┐  │
│  │         JMeter Runner Service         │  │
│  └────────────────────┬──────────────────┘  │
└───────────────────────┼─────────────────────┘
                        │ subprocess / RMI
         ┌──────────────┼──────────────┐
         │              │              │
    ┌────▼────┐   ┌─────▼────┐   ┌────▼────┐
    │ JMeter  │   │  Docker  │   │  Docker  │
    │  Local  │   │  Slave 1 │   │  Slave 2 │
    └─────────┘   └──────────┘   └──────────┘
```

---

## Getting Started

### Prerequisites

- Python 3.11+
- Node.js 18+
- PostgreSQL
- Apache JMeter 5.6.x (extracted, not installed)
- Docker (for distributed execution)

### Backend Setup

```bash
cd backend

# Create virtual environment
python -m venv .venv
.venv\Scripts\activate          # Windows
# source .venv/bin/activate     # Linux / Mac

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env — set DATABASE_URL, SECRET_KEY, JMETER_HOME
```

**.env example:**
```env
DATABASE_URL=postgresql://user:password@localhost:5432/perfplatform
SECRET_KEY=your-secret-key-here
JMETER_HOME=C:/path/to/apache-jmeter-5.6.3
RESULTS_FOLDER=C:/perf-platform/results
APP_ENV=development
```

```bash
# Start the backend
uvicorn app.main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
npm run dev
```

The UI will be available at `http://localhost:5173`.

### Docker Slave Setup (Distributed Execution)

```bash
# Build the JMeter slave image
docker build -t jmeter-slave ./docker

# Run slave 1 (port 1099)
docker run -d \
  --name jmeter-slave1 \
  -p 1099:1099 \
  -v C:/perf-data/slave1:/test-data \
  jmeter-slave

# Run slave 2 (port 1100)
docker run -d \
  --name jmeter-slave2 \
  -p 1100:1099 \
  -v C:/perf-data/slave2:/test-data \
  jmeter-slave
```

Then register the slaves in **Settings → Load Generators**:

| Field | Slave 1 | Slave 2 |
|---|---|---|
| JMeter Host | `172.18.0.2` | `172.18.0.3` |
| JMeter Port | `1099` | `1099` |
| Connection Host | `localhost` | `localhost` |
| Connection Port | `1099` | `1100` |
| Slave Type | Docker | Docker |

> **Important:** Each Docker slave must have a unique Connection Port matching its host port mapping. Using the same port for two slaves causes both JMeter masters to connect to the same container.

---

## Usage

### Running a Local Test

1. Go to **Execution** → upload or select a JMX file
2. Choose **Local Execution**
3. Edit thread groups, ramp-up, duration as needed
4. Verify CSV data files in the **Data Setup** step
5. Click **Start Now**
6. Watch live logs and switch to **Monitoring** for real-time metrics

### Running a Distributed Test

1. Choose **Distributed Execution** in Step 1
2. Assign a slave to the JMX in **Select Slaves**
3. Verify CSV files are present on the slave in **Data Setup**
4. Start the test

### Running Parallel Tests (Multiple JMX)

1. Upload multiple JMX files
2. Select **Parallel** run mode
3. Assign a **different** slave to each JMX file (UI enforces this)
4. Start — both tests fire simultaneously, each monitored on its own tab

---

## Project Structure

```
perf-platform/
├── backend/
│   ├── app/
│   │   ├── api/          # FastAPI route handlers
│   │   │   ├── execution.py
│   │   │   ├── monitoring.py
│   │   │   ├── projects.py
│   │   │   ├── scheduler.py
│   │   │   └── ...
│   │   ├── core/         # Config, DB, security
│   │   ├── models/       # SQLAlchemy models
│   │   └── services/     # JMeter runner, JMX patcher, scheduler
│   └── requirements.txt
├── frontend/
│   └── src/
│       ├── pages/
│       │   └── dashboard/
│       │       ├── Execution.jsx
│       │       ├── Monitoring.jsx
│       │       ├── Projects.jsx
│       │       ├── Scheduler.jsx
│       │       └── ...
│       ├── components/
│       └── services/
├── docker/
│   ├── Dockerfile          # JMeter slave image
│   └── jmeter-slave-start.sh
└── README.md
```

---

## Key Design Decisions

**JMX patching at run time** — source JMX files are never modified. Each run gets its own patched copy written to the run's result folder, allowing concurrent runs without interference.

**JTL streaming parse** — the monitoring API re-parses the JTL file on every poll, computing all metrics (percentiles, histograms, over-time buckets) on the fly. No separate aggregation process needed.

**Per-JMX slave isolation** — in parallel distributed mode, the UI enforces that no two JMX files share the same slave endpoint, preventing JMeter masters from sending results to each other's JTL.

**RMI hostname via JVM_ARGS** — `java.rmi.server.hostname` must be injected through the `JVM_ARGS` environment variable (picked up by JMeter's bat/sh before JVM startup), not as a JMeter `-D` arg which is processed after the JVM and RMI have already initialized.

---

## Author

**Vishal Singh**
Performance Test Engineer
[vshalvsngh@gmail.com](mailto:vshalvsngh@gmail.com)

