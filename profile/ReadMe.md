# 🏊 Swimeo

> **The open platform for competitive swimming events.**  
> From event creation to live tracking and automated results – all in one place.

---

## What is Swimeo?

Swimeo is an all-in-one event operations platform for managing competitive long-distance swimming events. It digitises every phase of a swim event: planning, live execution and automated evaluation – replacing paper-based processes with a real-time, role-aware system accessible from any device.

Built for swim clubs, schools, companies and event organisers who need a reliable, transparent and modern solution.

---

## Repositories

| Repository | Description | Stack |
|---|---|---|
| [`lanepilot-app`](https://github.com/) | Mobile & web client for all user roles | Expo · React Native · TypeScript |
| [`lanepilot-api`](https://github.com/) | REST API, WebSocket and SSE backend | Python · FastAPI · PostgreSQL · Redis |

---

## Architecture

```mermaid
graph TD
    subgraph Clients
        A[📱 Expo App\nOrganiser · Staff]
        B[🌐 Web App\nParticipants · Sponsors]
    end

    subgraph Identity
        C[🔐 Authentik\nOIDC / OAuth2]
    end

    subgraph Backend
        D[⚡ FastAPI\nREST · WebSocket · SSE]
        E[🔄 Celery\nAsync Task Queue]
    end

    subgraph Data
        F[(🗄️ PostgreSQL\nPrimary Store)]
        G[(⚡ Redis\nPub/Sub · Cache)]
        H[📁 File Storage\nPDF Certificates]
    end

    A -->|PKCE Login| C
    B -->|PKCE Login| C
    C -->|JWT + Groups| A
    C -->|JWT + Groups| B
    A -->|Bearer Token| D
    B -->|Bearer Token| D
    D -->|Verify| C
    D -->|Read / Write| F
    D -->|Publish / Subscribe| G
    G -->|Broadcast| D
    D -->|Enqueue| E
    E -->|Generate PDF| H
    E -->|Write result| F
```

---

## Real-time Data Flow

Three parallel channels serve different roles simultaneously.

```mermaid
sequenceDiagram
    participant Staff as 👷 Staff (App)
    participant API as ⚡ FastAPI
    participant Redis as 🔄 Redis
    participant Org as 📊 Organiser (WS)
    participant Part as 🏊 Participant (SSE)

    Staff->>API: POST /laps { lap, distance, client_uuid }
    API->>API: Persist to PostgreSQL
    API-->>Staff: 201 Created
    API->>Redis: PUBLISH event:{id} { lap_data }
    Redis-->>Org: WebSocket push (< 100ms)
    Redis-->>Part: SSE push (personal dashboard)
```

---

## Role Model

```mermaid
graph LR
    subgraph Authentik Groups
        AD[swimtrack-admin]
        OR[swimtrack-organiser]
        HE[swimtrack-staff]
        PA[swimtrack-participant]
    end

    subgraph Permissions
        AD -->|Full access| P1[Manage platform]
        OR -->|Event scope| P2[Create & manage events\nAssign staff · View all data]
        HE -->|Event scope| P3[Record laps\nView own shift]
        PA -->|Own data| P4[View dashboard\nDownload certificate]
    end
```

---

## Core Data Model

```mermaid
erDiagram
    EVENTS {
        uuid id PK
        text name
        text organiser_id
        text location
        timestamptz start_time
        text status
    }
    LANES {
        uuid id PK
        uuid event_id FK
        int lane_number
        text label
    }
    PARTICIPANTS {
        uuid id PK
        uuid event_id FK
        text auth_user_id
        text full_name
        text category
        text bib_number
    }
    REGISTRATIONS {
        uuid id PK
        uuid participant_id FK
        uuid lane_id FK
        timestamptz registered_at
        text status
    }
    EVENT_STAFF {
        uuid id PK
        uuid event_id FK
        text auth_user_id
        uuid lane_id FK
        timestamptz shift_start
        timestamptz shift_end
    }
    LAPS {
        uuid id PK
        uuid client_uuid
        uuid participant_id FK
        uuid recorded_by FK
        int lap_number
        int distance_m
        timestamptz recorded_at
        timestamptz synced_at
    }
    EVENTS ||--o{ LANES : contains
    EVENTS ||--o{ PARTICIPANTS : registers
    EVENTS ||--o{ EVENT_STAFF : employs
    PARTICIPANTS ||--o{ REGISTRATIONS : has
    LANES ||--o{ REGISTRATIONS : holds
    PARTICIPANTS ||--o{ LAPS : swims
    EVENT_STAFF ||--o{ LAPS : records
```

---

## Key Features

**Event Management**
- Create and configure events with location, timing, sponsors and lane setup
- Define participant categories (school, club, company, private)
- Staff assignment with shift and break scheduling

**Live Execution**
- Real-time lap recording by staff via mobile app
- Offline-capable – laps are stored locally and synced when connectivity returns
- Free lane registration and transfer for participants during the event
- Live organiser dashboard via WebSocket (< 100ms latency)

**Evaluation & Results**
- Automatic ranking by category using PostgreSQL window functions
- Personal dashboards for participants via SSE push
- PDF certificate generation (async, WeasyPrint + Jinja2 templates)
- Exportable result data per event

---

## Tech Stack

| Layer | Technology |
|---|---|
| Mobile & Web | Expo · React Native · TypeScript |
| API | Python 3.12 · FastAPI · Pydantic v2 |
| Auth | Authentik (self-hosted OIDC/OAuth2) |
| Database | PostgreSQL 16 |
| Realtime | Redis 7 (Pub/Sub) |
| Task Queue | Celery |
| PDF | WeasyPrint · Jinja2 |
| Infrastructure | Docker · Docker Compose |

---

## Offline Sync Strategy

Staff members operate in environments with unreliable Wi-Fi (indoor swimming halls). Every lap recorded by a staff member is written immediately to local storage (`expo-sqlite`) with a locally generated `client_uuid`. A background sync queue pushes unsynced records (`synced_at = null`) to the API when connectivity is available. The `client_uuid` unique constraint on the server prevents duplicates in all scenarios.

---

## Project Status

> 🚧 **In active development.** Not yet production-ready.

- [x] Architecture design
- [x] Data model
- [ ] Docker Compose local setup
- [ ] Authentik OIDC integration (Expo PKCE flow)
- [ ] Core API endpoints
- [ ] Expo app scaffolding
- [ ] Real-time WebSocket / SSE layer
- [ ] PDF certificate generation
- [ ] Offline sync queue

---

## License

MIT

