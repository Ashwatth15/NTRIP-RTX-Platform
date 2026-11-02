# Major Project: Development of NTRIP Caster, NTRIP Client and NTRIP Server on Web/Mobile Platform for RTK Correction Transmission and Reception

![License](https://img.shields.io/badge/License-MIT-blue.svg)
![Python](https://img.shields.io/badge/Python-3.13%2B-blue.svg)
![React](https://img.shields.io/badge/React-18-cyan.svg)
![NTRIP](https://img.shields.io/badge/NTRIP-1.0%20%7C%202.0-emerald.svg)
![RTCM](https://img.shields.io/badge/RTCM-3.3%20MSM4-orange.svg)

---

## 1. Project Overview

This repository contains the complete, production-grade implementation of an end-to-end **NTRIP (Networked Transport of RTCM via Internet Protocol)** correction distribution ecosystem developed for academic and engineering demonstration.

Unlike UI prototypes, this system executes **real binary TCP networking** adhering to official NTRIP 1.0 (ICY) and NTRIP 2.0 (HTTP chunked) specifications, complete with binary RTCM 3.x frame framing (preamble `0xD3`), Qualcomm CRC-24Q validation, Antenna Reference Point (ARP) coordinate decoding, and real-time NMEA-0183 `$GNGGA` rover tracking.

### Core Ecosystem Components:
1. **NTRIP Caster (TCP Port 2101)**: High-throughput streaming router that handles dynamic mountpoints, sourcetable discovery (`SOURCETABLE 200 OK`), client/server authentication, and real-time multiplexing ring buffers.
2. **NTRIP Server**: Source ingestion service that connects to physical GNSS Base Stations (via RS232/USB Serial COM ports or TCP sockets) or simulated RTCM3 streams, pushing corrections upstream.
3. **NTRIP Client**: Consumer service that connects to the Caster, subscribes to mountpoints, decodes RTCM3 packets, forwards corrections to Rover receivers, and reports rover coordinates via NMEA GGA.
4. **Web Control & Telemetry Dashboard**: Responsive operations center built with React 18, Tailwind CSS, Lucide icons, and Leaflet GIS mapping, driven by 1 Hz real-time WebSockets.
5. **Persistence Layer**: Relational database (SQLite/PostgreSQL) tracking users, mountpoints, active/historic sessions, and time-tagged event logs with CSV/JSON export.
6. **Dual Hardware / Simulation Engine**: Native support for physical GNSS receivers alongside a built-in multi-GNSS RTCM3 generator and moving rover kinematics simulator for indoor laboratory demonstrations.

---

## 2. System Architecture & Communication Flow

```
+----------------------------------------------------------------------------------------------------+
|                                PHYSICAL / SIMULATED HARDWARE LAYER                                 |
|                                                                                                    |
|   +------------------------------------+              +------------------------------------+       |
|   |   GNSS Base Station / Receiver     |              |    GNSS Rover Receiver (Target)    |       |
|   |   (u-blox ZED-F9P, Septentrio,     |              |    (u-blox ZED-F9P / RTK Rover     |       |
|   |    Serial COM / TCP Socket)        |              |     Serial COM / TCP Socket)       |       |
|   |         OR RTCM3 Replay            |              |         OR NMEA Simulator          |       |
|   +-----------------+------------------+              +-----------------+------------------+       |
+---------------------|---------------------------------------------------|--------------------------+
                      | Raw RTCM3 Stream                                  ^ Injected RTCM3 Stream
                      |                                                   | & NMEA $GPGGA strings
                      v                                                   |
+-------------------------------------------------------------------------|--------------------------+
|                                    NTRIP STREAMING SERVICES LAYER       |                          |
|                                                                         |                          |
|   +------------------------------------+              +-----------------+------------------+       |
|   |           NTRIP SERVER             |              |            NTRIP CLIENT            |       |
|   |  - Hardware Ingest (COM/TCP)       |              |  - Caster Connect (HTTP/ICY)       |       |
|   |  - Source Reconnection Engine      |              |  - Mountpoint Discovery            |       |
|   |  - RTCM Frame Validation           |              |  - RTCM Frame Demux & Stats        |       |
|   |  - NTRIP 1.0/2.0 Push Engine       |              |  - Serial/TCP Out to Hardware Rover|       |
|   +-----------------+------------------+              +-----------------+------------------+       |
|                     |                                                   ^                          |
|                     | NTRIP Source Push                                 | NTRIP Client Pull        |
|                     | (SOURCE / POST)                                   | (GET /mountpoint)        |
|                     v                                                   |                          |
|   +---------------------------------------------------------------------+------------------+       |
|   |                                     NTRIP CASTER                                       |       |
|   |  - Port 2101 / Custom Port TCP Async Engine                                            |       |
|   |  - Dynamic Mountpoint Registry & Sourcetable Generator (SOURCETABLE 200 OK)            |       |
|   |  - Client & Server Basic / Token Authentication                                        |       |
|   |  - Fan-out Broadcast Hub (1 Source -> N Clients Ring Buffer)                           |       |
|   |  - Real-time RTCM3 Header Parser (Msg 1005, 1074, 1084, 1094, 1124...)                 |       |
|   |  - GGA Extraction Engine (Extracts Rover Lat/Lon from incoming Client stream)          |       |
|   +-----------------+---------------------------------------------------+------------------+       |
+---------------------|---------------------------------------------------|--------------------------+
                      | Event Bus & Telemetry Ring (Internal Queue)       |
                      v                                                   v
+----------------------------------------------------------------------------------------------------+
|                                    APPLICATION & DATA PERSISTENCE LAYER                            |
|                                                                                                    |
|   +------------------------------------+              +------------------------------------+       |
|   |      FastAPI Core Application      |<--- JWT ----->|    SQLite / PostgreSQL Database    |       |
|   |  - REST Endpoints (CRUD, Config)   |              |  - Users & Roles (Admin, Rover)    |       |
|   |  - Real-time WebSocket Hub         |              |  - Mountpoints Registry            |       |
|   |  - Background Worker Tasks         |              |  - Active & Historic Sessions      |       |
|   |  - System Health & Event Logger    |              |  - System Logs & Audit Trail       |       |
|   +-----------------+------------------+              +------------------------------------+       |
+---------------------|------------------------------------------------------------------------------+
                      | JSON REST + Binary/JSON WebSockets
                      v
+----------------------------------------------------------------------------------------------------+
|                                    USER INTERFACE LAYER (DASHBOARD)                                |
|                                                                                                    |
|   +--------------------------------------------------------------------------------------------+   |
|   |     React 18 + Vite + TypeScript + Tailwind CSS + Leaflet GIS Engine                       |   |
|   |  - Real-time Caster/Server/Client Telemetry KPIs (Bitrate, Packets/sec, Latency)           |   |
|   |  - Interactive Map: Base Station marker (RTCM 1005/1006 ECEF->WGS84) + Rovers (NMEA GGA)   |   |
|   |  - Mountpoint Management & Live Sourcetable Inspector                                      |   |
|   |  - RTCM3 Packet Distribution Chart (1005/1006, 1074 GPS, 1084 GLO, 1094 GAL, 1124 BDS)   |   |
|   |  - Time-Tagged Event Log Streamer & CSV/JSON Export                                        |   |
|   +--------------------------------------------------------------------------------------------+   |
+----------------------------------------------------------------------------------------------------+
```

---

## 3. Technology Stack

- **Backend & Networking**: Python 3.13+, `asyncio` TCP Server, `pyserial`
- **Web API & WebSockets**: FastAPI, Uvicorn, WebSockets
- **Database**: SQLite (WAL Mode) with SQLAlchemy 2.0 Async (`aiosqlite`)
- **Security**: Passlib / bcrypt password hashing, PyJWT Bearer Authentication
- **Frontend Dashboard**: React 18, Vite, TypeScript, Tailwind CSS, Lucide Icons
- **GIS Engine**: Leaflet & React-Leaflet with CartoDB Voyager tiles
- **Test Suite**: Pytest, Pytest-asyncio, HTTPX

---

## 4. Quickstart Guide (Running the Project)

### Prerequisites:
- Python 3.10+ (Python 3.13 verified)
- Node.js 18+ (Node.js v22 verified)

### Step 1: Clone and Set Up Virtual Environment
```powershell
# Open terminal in project root
python -m venv venv
.\venv\Scripts\Activate.ps1

# Install backend dependencies
pip install -r backend/requirements.txt
```

### Step 2: Build or Run the Frontend
```powershell
cd frontend
npm install
npm run build
cd ..
```
*(The production build in `frontend/dist` is automatically served directly by FastAPI!)*

### Step 3: Launch the Full Application
```powershell
$env:PYTHONPATH="."
.\venv\Scripts\python.exe backend/main.py
```

Now open your web browser:
- **Web Dashboard**: `http://localhost:8000`
- **Interactive REST API Docs (Swagger)**: `http://localhost:8000/docs`
- **Raw NTRIP Sourcetable**: `http://localhost:8000/api/v1/sourcetable`
- **NTRIP TCP Caster**: Listening directly on `127.0.0.1:2101`

---

## 5. End-to-End Academic Demonstration Procedure

Follow these simple steps to demonstrate the complete RTCM transmission and reception pipeline during a college project presentation:

### 1. Log In to the Dashboard
- Open `http://localhost:8000`.
- Click the **Autofill** button or enter credentials:
  - **Username**: `admin`
  - **Password**: `admin123`
- Click **Sign In to Dashboard**.

### 2. Inspect the Initial Standby State
- The top KPI cards show:
  - **NTRIP Caster Status**: `ONLINE` on TCP port `2101`.
  - **Throughput Bitrate**: `0 bps`.
  - **Connected Rovers**: `0`.
  - **Decoded RTCM Frames**: `0 pkts`.
- The interactive map displays the default campus coordinates.

### 3. Start the Academic Multi-GNSS Simulator
- In the **Correction Source Controller** card, click **START SIMULATOR**.
- Watch the live telemetry spring to life:
  - The real **NTRIP Server** connects to Caster port `2101` with handshake `SOURCE basepass123 /CAMPUS_BASE`.
  - The Caster replies with `ICY 200 OK`.
  - The Server begins streaming binary RTCM 3.3 packets (Msg 1005 ARP, Msg 1074 GPS, Msg 1084 GLONASS, Msg 1094 Galileo, Msg 1124 BeiDou) at 1 Hz.
  - The real **NTRIP Client** connects with HTTP Basic Auth `admin:admin123`, receives `ICY 200 OK`, and continuously receives the stream.
  - The Client decodes each packet, verifies CRC-24Q integrity, and emits NMEA `$GNGGA` coordinates of a moving rover back to the Caster.

### 4. Observe Real-Time Visualizations
- **Interactive Map**:
  - The **Base Station** marker (📡) appears at the exact location decoded from RTCM Message 1005 ARP.
  - The **Rover** marker (🚗) moves along a kinematic orbit in real-time, showing `RTK Fixed (Q=4)` and 18 tracking satellites.
- **RTCM 3.x Message Distribution**:
  - Real-time progress bars illustrate the distribution of packets across GPS, GLONASS, Galileo, and BeiDou constellations.
- **Mountpoints Table**:
  - Shows `/CAMPUS_BASE` transitioned to `STREAMING` state with byte and packet counters increasing.
- **Live Event & Audit Logs**:
  - Watch time-tagged events stream in real-time (`CONNECT`, `RTCM DECODE`, `GGA UPLINK`).

### 5. Export Time-Tagged Logs
- In the **Time-Tagged Event & Audit Logs** panel, click **CSV** or **JSON**.
- The browser downloads the audit history file (`ntrip_system_logs.csv`), showing timestamps, components, levels, and messages.

### 6. Stop the Simulator
- Click **STOP SIMULATOR**. The streams disconnect cleanly, and the session disconnection events are logged into the database.

---

## 6. How to Connect Real Physical GNSS Hardware Later

The application is completely hardware-ready and decoupled from the simulator. You can connect physical RTK base stations and rovers:

### Connecting a Physical GNSS Base Station:
1. Connect your RTK Base receiver (e.g., **u-blox ZED-F9P**, Septentrio Mosaic-X5, or Trimble BD980) to your computer via USB/Serial or Ethernet LAN.
2. In the dashboard's **Correction Source Controller**, click the **Physical GNSS Receiver** tab.
3. Select **Serial COM Port** (e.g. `COM3` at `115200` baud) or **TCP Socket** (e.g. `192.168.1.100:5017`).
4. Click **Connect Physical Receiver**.
5. The NTRIP Server opens the COM port via `pyserial`, reads raw binary RTCM bytes, and forwards them directly to the Caster.

### Connecting a Physical GNSS Rover Receiver or External NTRIP Client:
You can connect industry-standard NTRIP clients like **RTKLIB**, **QGIS**, or **Lefebure NTRIP Client**:
- **Caster Host**: `127.0.0.1` (or local LAN IP)
- **Caster Port**: `2101`
- **Mountpoint**: `CAMPUS_BASE`
- **Username**: `admin`
- **Password**: `admin123`

---

## 7. Running the Automated Test Suite

To verify all components across Phase 1 through Phase 7:
```powershell
$env:PYTHONPATH="."
.\venv\Scripts\pytest.exe backend/tests/ -v
```

Expected output:
```
backend/tests/test_full_system.py::test_complete_end_to_end_system PASSED
backend/tests/test_phase1.py::test_rtcm_crc_and_decoder PASSED
backend/tests/test_phase1.py::test_rtcm_msm4_frames PASSED
backend/tests/test_phase1.py::test_nmea_gga_parsing PASSED
backend/tests/test_phase1.py::test_sourcetable PASSED
backend/tests/test_phase1.py::test_caster_tcp_sourcetable PASSED
backend/tests/test_phase2_3.py::test_end_to_end_ntrip_streaming PASSED
backend/tests/test_phase4_5.py::test_api_and_database_integration PASSED
============================== 8 passed in 6.87s ==============================
```

---

## 8. REST & WebSocket API Reference

| Endpoint | Method | Description |
| :--- | :---: | :--- |
| `/api/v1/health` | `GET` | Health check and system operational state |
| `/api/v1/auth/login` | `POST` | Authenticate and issue JWT Bearer token |
| `/api/v1/auth/me` | `GET` | Get current user profile and role |
| `/api/v1/sourcetable` | `GET` | Fetch standard text or JSON NTRIP Sourcetable |
| `/api/v1/mountpoints` | `GET` | List all mountpoints and real-time streaming state |
| `/api/v1/mountpoints` | `POST` | Register a new mountpoint (Admin only) |
| `/api/v1/mountpoints/{id}` | `DELETE` | Deactivate a mountpoint (Admin only) |
| `/api/v1/sessions/active` | `GET` | Real-time list of connected sources and rovers |
| `/api/v1/sessions/history` | `GET` | Historical connection sessions from database |
| `/api/v1/logs` | `GET` | Filtered time-tagged event logs |
| `/api/v1/logs/export` | `GET` | Export logs as CSV or JSON file |
| `/api/v1/simulator/start` | `POST` | Start Multi-GNSS RTCM3 & Rover simulator |
| `/api/v1/simulator/stop` | `POST` | Stop simulator |
| `/api/v1/control/server/start` | `POST` | Start physical Serial/TCP NTRIP Server |
| `/ws/telemetry` | `WS` | 1 Hz real-time WebSocket telemetry feed |

---

## 9. Technical Limitations & Engineering Assumptions

1. **Academic Demonstration vs. Multi-Gigabit Scale**: Python `asyncio` handles 200–500 simultaneous Rover client streams at 1 Hz RTCM3 (~5–15 kbps per stream), which exceeds academic requirements.
2. **Indoor Demonstration Assumption**: Physical GNSS receivers require open sky to acquire satellites. The built-in **RTCM3 Simulator/Replay Mode** ensures a 100% dependable indoor laboratory presentation, while physical Serial COM ports are fully supported for outdoor hardware demonstrations.
3. **Database Concurrency**: SQLite with WAL (Write-Ahead Logging) is used by default for zero external database dependencies. Switching to PostgreSQL requires changing just the `DATABASE_URL` in `.env`.
4. **Cellular Field Rovers**: To connect rovers over 4G/5G mobile networks in the field, port `2101` on the host machine must be port-forwarded, or tunneled via Cloudflare Tunnel / ngrok / VPS.
