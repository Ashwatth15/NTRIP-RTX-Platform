# System Architecture & Technical Specification
## "Development of NTRIP Caster, NTRIP Client and NTRIP Server on Web/Mobile Platform for RTK Correction Transmission and Reception"

---

## 1. Executive Summary & Requirement Analysis

### 1.1 Project Objective
The goal is to design and develop an end-to-end, production-grade **NTRIP (Networked Transport of RTCM via Internet Protocol)** correction distribution ecosystem. The system comprises:
1. **NTRIP Caster**: High-throughput, multi-tenant streaming router that authenticates GNSS Base Stations (Servers) and Rovers (Clients), manages dynamic Mountpoints, and multiplexes binary RTCM3 streams.
2. **NTRIP Server**: Source ingestion service that connects to physical GNSS Base Stations (via Serial RS232/USB COM ports or TCP/IP) or simulated RTCM3 streams, pushing corrections to the Caster.
3. **NTRIP Client**: Consumer service that connects to the Caster, selects mountpoints, receives RTCM correction packets, computes stream health metrics, optionally sends NMEA GGA rover coordinates, and delivers corrections to Rover receivers.
4. **Web Control & Telemetry Dashboard**: Responsive operations center offering real-time stream monitoring, sourcetable inspection, interactive rover/base mapping, RTCM packet decoding, and session logging.
5. **Persistent Store**: Relational database for authentication, access control, active session tracking, and historical audit logs.

### 1.2 Core Architectural Principles
- **Decoupled Networking vs. Management**: NTRIP streaming (binary TCP/HTTP) runs on dedicated asynchronous daemons isolated from the Web API and dashboard.
- **Hardware-Ready with Deterministic Simulation**: Native support for real hardware (Serial ports `COMx` / `/dev/ttyUSB*` and TCP sockets) alongside a realistic RTCM3/NMEA replay simulator for indoor academic defense and testing.
- **Standard Protocol Compliance**: Full compliance with **NTRIP 1.0 (ICY)** and **NTRIP 2.0 (HTTP/1.1 chunked/stream)** RFC/BKG standards.
- **Zero Mock Final Pipeline**: Telemetry metrics are computed directly from real byte counters and RTCM3 packet header decoders (Preamble `0xD3`, Message Type, Length, CRC-24Q).

---

## 2. System Architecture

```
+----------------------------------------------------------------------------------------------------+
|                                    PHYSICAL / SIMULATED HARDWARE LAYER                             |
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
|   |  - Real-time RTCM3 Header Parser (Msg 1005, 1074, 1084, 1094, 1124...)                     |       |
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
|   |     React 18 + Vite + Tailwind CSS + Lucide Icons + Leaflet GIS Engine                     |   |
|   |  - Real-time Caster/Server/Client Telemetry KPIs (Bitrate, Packets/sec, Latency)           |   |
|   |  - Interactive Map: Base Station marker (RTCM 1005/1006 ECEF->WGS84) + Rovers (NMEA GGA)   |   |
|   |  - Mountpoint Management & Live Sourcetable Inspector                                      |   |
|   |  - RTCM3 Packet Distribution Chart (1005/1006, 1074 GPS, 1084 GLO, 1094 GAL, 1124 BDS)   |   |
|   |  - Time-Tagged Event Log Streamer & CSV/JSON Export                                        |   |
|   +--------------------------------------------------------------------------------------------+   |
+----------------------------------------------------------------------------------------------------+
```

---

## 3. Technology Stack Selection

| Subsystem | Technology | Version | Rationale |
| :--- | :--- | :--- | :--- |
| **Backend & Daemon Runtime** | Python | 3.13+ | Superior ecosystem for GNSS/RTCM/NMEA processing, native async networking, cross-platform hardware serial communication. |
| **Async Networking** | `asyncio` | Built-in | Single-threaded event loop capable of handling hundreds of concurrent low-latency TCP sockets with minimal memory overhead. |
| **Web API Framework** | FastAPI + Uvicorn | Latest | High-performance asynchronous REST API, automatic OpenAPI / Swagger documentation, native WebSocket integration. |
| **GNSS & Serial Utilities** | `pyserial`, `pyrtcm` (or custom zero-dependency decoder) | Latest | Reading physical COM ports (`COM1`, `/dev/ttyUSB0`) and parsing RTCM3 frames (Message IDs, CRC-24Q). |
| **Database & ORM** | SQLite + SQLAlchemy 2.0 (`aiosqlite`) | Latest | Zero configuration, self-contained for college presentation. Can be switched to PostgreSQL with zero code changes via connection string. |
| **Security & Auth** | Passlib (Argon2 / Bcrypt) + PyJWT | Latest | Modern password hashing, stateless JWT for Web API, standard HTTP Basic Auth for NTRIP Client/Server protocol. |
| **Frontend Framework** | React 18 + Vite + TypeScript | Latest | Ultra-fast HMR development, clean component model, type-safe telemetry contracts. |
| **Styling & Icons** | Tailwind CSS + Lucide React | Latest | Modern responsive UI, dark/light academic workstation aesthetics, responsive mobile layout. |
| **Mapping & GIS** | Leaflet + React-Leaflet | Latest | Lightweight, open-source geospatial visualization for Base Stations and Rover markers with real-time coordinate updates. |
| **Charts & Metrics** | Recharts / Chart.js | Latest | Real-time byte rate telemetry and RTCM message frequency distribution visualization. |

---

## 4. NTRIP Communication Protocols & Handshakes

### 4.1 NTRIP Server -> NTRIP Caster Handshake
The NTRIP Server pushes correction data to the Caster using either NTRIP 1.0 or NTRIP 2.0 standards:

#### NTRIP 1.0 Ingestion:
```http
SOURCE password /BASE01
Source-Agent: NTRIP MyMajorProjectServer/1.0
\r\n
```
*Caster Response:*
```http
ICY 200 OK\r\n\r\n
```
*(Immediately followed by raw continuous binary RTCM stream over the open TCP connection)*

#### NTRIP 2.0 Ingestion:
```http
POST /BASE01 HTTP/1.1\r\n
Host: 127.0.0.1:2101\r\n
User-Agent: NTRIP MyMajorProjectServer/2.0\r\n
Authorization: Basic c291cmNldXNlcjpwYXNzd29yZA==\r\n
Ntrip-Version: Ntrip/2.0\r\n
Content-Type: gnss/data\r\n
Transfer-Encoding: chunked\r\n
\r\n
```
*Caster Response:*
```http
HTTP/1.1 200 OK\r\n
Server: MajorProjectCaster/2.0\r\n
Ntrip-Version: Ntrip/2.0\r\n
Connection: close\r\n
\r\n
```

---

### 4.2 NTRIP Client -> NTRIP Caster Handshake

#### A. Sourcetable Request (Discovery):
When a client connects to root (`/`) without mountpoint or requests the sourcetable:
```http
GET / HTTP/1.1\r\n
Host: 127.0.0.1:2101\r\n
User-Agent: NTRIP MajorProjectClient/1.0\r\n
Connection: close\r\n
\r\n
```
*Caster Response:*
```http
SOURCETABLE 200 OK\r\n
Server: MajorProjectCaster/1.0\r\n
Content-Type: text/plain\r\n
Content-Length: 320\r\n
\r\n
CAS;127.0.0.1;2101;MajorProjectCaster;Academic;0;IND;0.00;0.00;0;0;\r\n
NET;MajorProjectNet;CollegeDept;B;N;http://localhost:8000;none;none;none;\r\n
STR;BASE01;CampusBase;RTCM 3.3;1005(1),1074(1),1084(1),1094(1),1124(1);2;GPS+GLO+GAL+BDS;MajorNet;IND;28.61;77.20;1;0;Septentrio;none;B;N;9600;none;\r\n
ENDSOURCETABLE\r\n
```

#### B. Mountpoint Stream Request (Data Subscription):
```http
GET /BASE01 HTTP/1.1\r\n
Host: 127.0.0.1:2101\r\n
User-Agent: NTRIP MajorProjectClient/1.0\r\n
Authorization: Basic cm92ZXIxOnBhc3M=\r\n
\r\n
```
*Caster Response (NTRIP 1.0 standard widely expected by RTK rovers):*
```http
ICY 200 OK\r\n\r\n
```
*Optional Client Position feedback (NMEA GGA transmitted by Rover to Caster every 5-10 seconds over the same connection):*
```text
$GNGGA,071422.00,2836.8123,N,07712.5432,E,1,12,1.0,215.4,M,-35.0,M,,*5A\r\n
```
*(Caster intercepts this sentence to plot the rover on the web map without interrupting the downstream RTCM binary stream)*.

---

## 5. End-to-End RTCM3 Data Flow

```
+------------------------------------------------------------------------------------+
| 1. GENERATION / ACQUISITION                                                        |
|   Base Receiver generates RTCM3 messages:                                          |
|   - Type 1005/1006: Stationary Antenna Reference Point (ARP) Coordinates (ECEF)    |
|   - Type 1074: GPS MSM4 Pseudorange & Phase                                        |
|   - Type 1084: GLONASS MSM4 Pseudorange & Phase                                    |
|   - Type 1094: Galileo MSM4 Pseudorange & Phase                                    |
|   - Type 1124: BeiDou MSM4 Pseudorange & Phase                                     |
+-----------------------------------------+------------------------------------------+
                                          |
                                          v
+------------------------------------------------------------------------------------+
| 2. INGESTION (NTRIP Server)                                                        |
|   - Reads bytes from Serial COM / TCP socket / Simulator File                      |
|   - Verifies 0xD3 preamble frame sync                                              |
|   - Buffers and transmits upstream to Caster over TCP via SOURCE or POST           |
+-----------------------------------------+------------------------------------------+
                                          |
                                          v
+------------------------------------------------------------------------------------+
| 3. ROUTING & INSPECTION (NTRIP Caster)                                             |
|   - Authenticates Base Server mountpoint credentials                               |
|   - Reads incoming byte stream into an asynchronous Broadcast RingBuffer           |
|   - In-memory Lightweight Decoder extracts:                                        |
|     * Packet ID (e.g. 1005, 1074)                                                  |
|     * Base Station Lat/Lon/Alt (converted from ECEF X, Y, Z in Msg 1005)           |
|     * Packet length, count, and byte bitrate                                       |
|   - Multiplexes binary stream to all active subscribed Rover client sockets        |
+--------------------+------------------------------------+--------------------------+
                     |                                    |
                     v                                    v
+------------------------------------+   +-------------------------------------------+
| 4. RECEPTION (NTRIP Client)        |   | 5. TELEMETRY & VISUALIZATION (Web UI)     |
|   - Connects to Caster mountpoint  |   |   - Caster pushes telemetry via WebSocket |
|   - Receives continuous RTCM bytes |   |   - Interactive Map plots:                |
|   - Verifies CRC-24Q integrity     |   |     * Base Station (Msg 1005 coords)      |
|   - Streams out to Rover Receiver  |   |     * Active Rovers (from NMEA GGA)       |
|     via COM port / Virtual TCP     |   |   - Packet frequency live graphs          |
|   - Computes packet reception stats|   |   - Downloadable logs & sourcetable export|
+------------------------------------+   +-------------------------------------------+
```

---

## 6. Database Schema Design (ER Specifications)

```sql
-- 1. Users Table (Authentication and Role Management)
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'client', -- 'admin', 'server', 'client'
    is_active BOOLEAN DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. Mountpoints Table (Stream Registry)
CREATE TABLE mountpoints (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    identifier VARCHAR(50) UNIQUE NOT NULL,     -- e.g., 'BASE01'
    mount_password VARCHAR(100) NOT NULL,      -- Secret for NTRIP Server ingestion
    format VARCHAR(30) DEFAULT 'RTCM 3.3',     -- e.g., 'RTCM 3.3', 'RTCM 3.0'
    format_details VARCHAR(100),               -- '1005(1),1074(1),1084(1),1094(1),1124(1)'
    carrier INTEGER DEFAULT 2,                 -- 0=None, 1=L1, 2=L1+L2
    nav_system VARCHAR(50) DEFAULT 'GPS+GLO+GAL+BDS',
    country VARCHAR(10) DEFAULT 'IND',
    latitude DOUBLE PRECISION DEFAULT 0.0,
    longitude DOUBLE PRECISION DEFAULT 0.0,
    altitude DOUBLE PRECISION DEFAULT 0.0,
    is_active BOOLEAN DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 3. Connection Sessions (Tracking active & historical Server & Client connections)
CREATE TABLE connection_sessions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_type VARCHAR(20) NOT NULL,         -- 'SERVER' or 'CLIENT'
    mountpoint_id INTEGER REFERENCES mountpoints(id),
    user_id INTEGER REFERENCES users(id),
    client_ip VARCHAR(45) NOT NULL,
    client_port INTEGER NOT NULL,
    user_agent VARCHAR(150),
    connected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    disconnected_at TIMESTAMP,
    bytes_transferred BIGINT DEFAULT 0,
    packets_transferred BIGINT DEFAULT 0,
    last_known_lat DOUBLE PRECISION,           -- From Rover NMEA GGA
    last_known_lon DOUBLE PRECISION,
    disconnect_reason VARCHAR(100)
);

-- 4. System & Security Event Logs (Audit & Time-tagged logs)
CREATE TABLE system_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    log_level VARCHAR(10) NOT NULL,            -- 'INFO', 'WARNING', 'ERROR', 'CRITICAL'
    component VARCHAR(30) NOT NULL,            -- 'CASTER', 'SERVER', 'CLIENT', 'AUTH'
    mountpoint VARCHAR(50),
    message TEXT NOT NULL,
    metadata_json TEXT                         -- JSON details (IP, packet type, etc.)
);

-- 5. RTCM Packet Telemetry Snapshots (Periodic aggregated telemetry for fast UI charts)
CREATE TABLE telemetry_snapshots (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    mountpoint_id INTEGER REFERENCES mountpoints(id),
    bytes_per_second INTEGER DEFAULT 0,
    active_clients INTEGER DEFAULT 0,
    msg_1005_count INTEGER DEFAULT 0,
    msg_1074_count INTEGER DEFAULT 0,
    msg_1084_count INTEGER DEFAULT 0,
    msg_1094_count INTEGER DEFAULT 0,
    msg_1124_count INTEGER DEFAULT 0,
    other_msg_count INTEGER DEFAULT 0
);
```

---

## 7. REST & WebSocket API Specification

### 7.1 REST Endpoints (`/api/v1`)
| Method | Endpoint | Description | Auth Required |
| :--- | :--- | :--- | :--- |
| `POST` | `/auth/login` | Authenticate user, returns JWT bearer token | Public |
| `GET` | `/auth/me` | Fetch authenticated user profile & permissions | Bearer JWT |
| `GET` | `/caster/status` | Get Caster state, uptime, active listeners, total throughput | Bearer JWT |
| `GET` | `/sourcetable` | Retrieve raw or JSON-formatted NTRIP Sourcetable | Public / JWT |
| `GET` | `/mountpoints` | List all configured mountpoints and live status | Bearer JWT |
| `POST` | `/mountpoints` | Register new mountpoint | Admin JWT |
| `DELETE` | `/mountpoints/{id}` | Remove mountpoint | Admin JWT |
| `GET` | `/sessions/active` | List real-time active Server & Client connections | Bearer JWT |
| `POST` | `/sessions/{id}/disconnect` | Forcefully drop an active connection | Admin JWT |
| `GET` | `/logs` | Query time-tagged logs with filtering (component, level) | Bearer JWT |
| `GET` | `/logs/export` | Export logs as CSV or JSON format | Bearer JWT |
| `POST` | `/simulator/start` | Trigger virtual Base Station & Rover simulation | Admin JWT |
| `POST` | `/simulator/stop` | Terminate active simulation | Admin JWT |

### 7.2 WebSocket Real-Time Telemetry (`/ws/telemetry`)
- **Protocol**: JSON-encoded bi-directional WebSocket.
- **Heartbeat & Telemetry Rate**: 1 Hz streaming broadcast.
- **Payload Structure**:
```json
{
  "type": "TELEMETRY_UPDATE",
  "timestamp": "2026-10-08T12:45:00Z",
  "caster": {
    "status": "ONLINE",
    "uptime_seconds": 3600,
    "total_in_bps": 9840,
    "total_out_bps": 29520,
    "active_sources": 1,
    "active_clients": 3
  },
  "mountpoints": [
    {
      "mount": "BASE01",
      "status": "STREAMING",
      "source_ip": "127.0.0.1",
      "connected_clients": 3,
      "bps": 9840,
      "base_position": { "lat": 28.6139, "lon": 77.2090, "alt": 216.5 },
      "last_packet_time": "2026-10-08T12:44:59.950Z",
      "packet_rates": { "1005": 0.2, "1074": 1.0, "1084": 1.0, "1094": 1.0, "1124": 1.0 }
    }
  ],
  "rovers": [
    {
      "session_id": 42,
      "username": "rover_field_01",
      "mount": "BASE01",
      "ip": "192.168.1.105",
      "lat": 28.6145,
      "lon": 77.2102,
      "alt": 215.1,
      "fix_quality": 4,
      "satellites": 18
    }
  ]
}
```

---

## 8. Proposed Project Folder Structure

```
major-project/
│
├── .env.example                     # Environment template (ports, secrets, DB URL)
├── README.md                        # Project documentation & run guide
├── ARCHITECTURE.md                  # Comprehensive architectural specification
│
├── backend/                         # Core Python Application
│   ├── requirements.txt             # Python dependencies
│   ├── main.py                      # Main entrypoint launching API & Caster workers
│   │
│   ├── config.py                    # Pydantic Settings & environment loader
│   ├── database.py                  # SQLAlchemy async engine & sessionmaker
│   ├── models/                      # Database ORM models
│   │   ├── user.py
│   │   ├── mountpoint.py
│   │   ├── session.py
│   │   └── log.py
│   │
│   ├── ntrip/                       # Autonomous NTRIP Networking Core
│   │   ├── __init__.py
│   │   ├── caster.py                # NTRIP Caster TCP Server & Client Router
│   │   ├── server.py                # NTRIP Server (Ingest from Serial/TCP/File)
│   │   ├── client.py                # NTRIP Client (Subscriber & Serial/TCP forwarder)
│   │   ├── sourcetable.py           # Sourcetable formatting & parser
│   │   ├── rtcm_decoder.py          # Zero-dep RTCM3 frame parser & CRC24Q validator
│   │   ├── nmea_parser.py           # NMEA GGA sentence parser
│   │   └── buffer.py                # Ring buffer for fan-out broadcasting
│   │
│   ├── simulator/                   # Realistic Academic Test Suite & Generator
│   │   ├── __init__.py
│   │   ├── rtcm_generator.py        # Generates valid RTCM3 1005/1074/1084 frames
│   │   ├── sample_data/             # Pre-recorded real-world RTCM3 dump samples
│   │   └── rover_simulator.py       # Simulates moving rover emitting NMEA GGA
│   │
│   ├── api/                         # FastAPI REST Endpoints & WebSocket
│   │   ├── v1/
│   │   │   ├── auth.py
│   │   │   ├── mountpoints.py
│   │   │   ├── sessions.py
│   │   │   ├── logs.py
│   │   │   └── simulator.py
│   │   └── websocket.py             # Live telemetry broadcaster
│   │
│   └── services/                    # Business Logic Layer
│       ├── auth_service.py
│       ├── telemetry_service.py
│       └── logger_service.py
│
└── frontend/                        # Web Dashboard (React + TypeScript)
    ├── package.json
    ├── vite.config.ts
    ├── tailwind.config.js
    ├── tsconfig.json
    └── src/
        ├── main.tsx
        ├── App.tsx
        ├── components/
        │   ├── Navbar.tsx
        │   ├── Sidebar.tsx
        │   ├── StatCard.tsx
        │   ├── MapView.tsx          # Leaflet GIS showing Base and Rovers
        │   ├── RtcmPacketChart.tsx  # Message breakdown chart
        │   ├── MountpointTable.tsx  # Sourcetable & mountpoint list
        │   ├── LiveLogViewer.tsx    # Live terminal log streaming & CSV export
        │   └── SimulatorControl.tsx # Toggle simulator/real hardware mode
        ├── hooks/
        │   ├── useWebSocket.ts      # WebSocket connection & telemetry hook
        │   └── useAuth.ts           # Authentication & token manager
        ├── pages/
        │   ├── Dashboard.tsx        # Central monitoring screen
        │   ├── MountpointsPage.tsx  # Mountpoint CRUD
        │   ├── SessionsPage.tsx     # Active rovers & base stations
        │   ├── LogsPage.tsx         # Audit logs & export
        │   └── LoginPage.tsx        # Authentication screen
        ├── types/
        │   └── telemetry.ts         # TypeScript definitions for telemetry payloads
        └── utils/
            ├── formatters.ts
            └── coords.ts            # Coordinate conversion utilities
```

---

## 9. Development Phases & Implementation Roadmap

| Phase | Milestone | Deliverables | Verification Strategy |
| :---: | :--- | :--- | :--- |
| **Phase 1** | **Core NTRIP Engine & RTCM Decoder** | - Asynchronous NTRIP Caster TCP Server (port 2101)<br>- Sourcetable generator & validator<br>- RTCM3 Preamble `0xD3` & CRC24Q parser<br>- Multiplexing broadcast ring buffer | Test with `curl -v http://localhost:2101/` to fetch sourcetable; unit tests on RTCM packet decoder. |
| **Phase 2** | **NTRIP Server & Source Ingestion** | - NTRIP Server connecting to serial COM port or RTCM file/simulator<br>- Handshake authentication (`SOURCE` / `POST`)<br>- Reconnect backoff engine | Verify stream ingest from Server into Caster with zero packet drop. |
| **Phase 3** | **NTRIP Client & Rover Relay** | - NTRIP Client connecting to Caster mountpoint<br>- Receiving RTCM frames and forwarding to Serial COM / TCP<br>- Periodic NMEA GGA coordinates sender | Test with open-source RTKLIB (`str2str` / `rtkrcv`) or QGIS NTRIP client to verify standard compatibility. |
| **Phase 4** | **FastAPI Core & Database Integration** | - SQLAlchemy SQLite models & migrations<br>- Authentication (JWT + Basic Auth)<br>- REST API for mountpoint and session management<br>- Time-tagged event logger | Automated pytest suite for REST API endpoints and database operations. |
| **Phase 5** | **Real-Time Telemetry & WebSocket Bus** | - 1 Hz telemetry aggregator<br>- WebSocket broadcaster pushing byte rates, packet counts, active sessions, and rover coordinates | WebSocket client script verifying JSON telemetry delivery and latency < 100ms. |
| **Phase 6** | **Responsive Web Dashboard** | - React + Vite + Tailwind UI<br>- Interactive Leaflet Map (Base station & Rover markers)<br>- Packet distribution charts<br>- Log viewer & CSV exporter<br>- Hardware / Simulator toggle controls | End-to-end user testing across desktop and mobile screen viewports. |
| **Phase 7** | **Physical Hardware Testing & Academic Polish** | - Connect real GNSS/RTK receiver (if available) or multi-point simulator<br>- Document setup instructions and academic project report | End-to-end demonstration readiness with zero errors. |

---

## 10. Technical Limitations & Engineering Assumptions

1. **Academic Demonstration vs. Multi-Gigabit Scale**:
   - The Python `asyncio` architecture comfortably handles 200–500 simultaneous Rover client streams at 1 Hz RTCM3 (typically ~5-15 kbps per stream), which exceeds academic requirements.
2. **Indoor Demonstration Assumption**:
   - Physical GNSS receivers require an open sky view to track satellites and emit valid RTCM3 corrections. For indoor college laboratory presentations, the built-in **RTCM3 Replay / Simulator Mode** ensures that the demonstration is 100% dependable without satellite signal dropouts.
3. **Database Concurrency**:
   - SQLite in WAL (Write-Ahead Logging) mode is used by default for zero-friction setup on Windows without installing heavy database servers. The architecture uses standard SQLAlchemy ORM, enabling an immediate switch to PostgreSQL via the `.env` file if desired.
4. **Network & NAT Traversal**:
   - For rovers connecting over cellular 4G/5G in field deployments, the Caster must be hosted on a public IP or port-forwarded/tunneled (e.g. ngrok / cloudflare tunnel / VPS). On local LAN or single-machine demonstration, `127.0.0.1` or the local Wi-Fi IP (`192.168.x.x`) is used.
