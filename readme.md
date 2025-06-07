# Grapfity System Architecture

## Table of Contents
1. [Architecture Overview](#architecture-overview)  
2. [Components](#components)  
3. [Request Flows](#request-flows)  
   - [1. Recommendation (`/api/recommendation`)](#1-recommendation-apirecommendation)  
   - [2. Event Tracking (`/api/event_tracking`)](#2-event-tracking-apievent_tracking)  
   - [3. Delete Track (`/api/tracks/:id`)](#3-delete-track-apitracksid)  
   - [4. Create Track (`/api/tracks/create-track`)](#4-create-track-apitrackscreate-track)  
   - [5. Other Backend Requests (`/api/*`)](#5-other-backend-requests-apis)  
4. [Containerization & Deployment](#containerization--deployment)  
5. [Running the Project](#running-the-project)  

---

## Architecture Overview

Grapfity is composed of the following key components:

- **Client**: ReactJS application providing the user interface.  
- **API Gateway**: Express Gateway acting as a single entry point, routing requests to the Backend Server and Recommender System Server.  
- **Backend Server**: Implements core business logic and interacts with MSSQL Server.  
- **Recommender System Server**: Provides song recommendations and event tracking, backed by PostgreSQL.  
- **Load Balancer**: Nginx distributes requests across two instances of the Recommender System Server (round-robin).  
- **Databases**:  
  - **PostgreSQL** for recommendation data and event logs.  
  - **MSSQL Server** for application data (users, metadata).  
- **Containerization**: All services are Dockerized and orchestrated via Docker Compose.

---

## Components

### 1. Client (ReactJS)
- Built with ReactJS.  
- Sends API requests to `http://localhost:8080/api/...`, which the API Gateway handles.

### 2. API Gateway (Express Gateway)
- Routes client requests to the correct downstream service based on path.  
- Applies policies (proxy, transformation, custom logic).  
- Configured via `gateway.config.yml`.

### 3. Backend Server
- Core functions: song management (create, delete, etc.).  
- CRUD operations against MSSQL Server.  
- Receives all `/api/*` requests (except those explicitly routed to recommendation/event).

### 4. Recommender System Server
- Generates song recommendations and logs user behavior events.  
- Interacts with PostgreSQL.  
- Deployed in two instances behind Nginx for load balancing.

### 5. Load Balancer (Nginx)
- Balances incoming recommendation requests across Recommender instances.  
- Ensures high availability and performance under load.

### 6. Databases
- **PostgreSQL**  
  - Stores user events (`events` table) and track feature data (`tracks` table).  
- **MSSQL Server**  
  - Stores application data: user profiles, song metadata, invoices, etc.

---

## Request Flows

### 1. Recommendation (`/api/recommendation`)
1. Client → `GET /api/recommendation?user_id=<id>`  
2. Gateway applies `recommendation-pipeline-policy`:  
   - Validate `user_id` (400 if missing).  
   - **Recommender Service**:  
     - `GET http://nginx:80/api/recommendation?user_id=<id>` → returns `track_ids`.  
   - **Backend Service**:  
     - `POST http://backend:8001/api/tracks/getTracksById`  
     - Payload: `{ "track_ids": [ ... ] }`  
   - Returns combined JSON to client (200).  
   - On error → log and return 500.

### 2. Event Tracking (`/api/event_tracking`)
1. Client → `POST /api/event_tracking`  
2. Gateway applies `event_tracking` pipeline → proxies to Recommender via `http://nginx:80`.

### 3. Delete Track (`/api/tracks/:id`)
1. Client → `DELETE /api/tracks/:id`  
2. Gateway applies `deleteTrack-pipeline-policy`:  
   - Validate `trackId` (400 if missing).  
   - **Backend Service**: `DELETE http://backend:8001/api/tracks/:id` → returns status.  
   - If successful, **Recommender Service**:  
     - `DELETE http://nginx:80/api/delete_track/:id`  
     - Retry up to 3 times (500 ms delay).  
   - Return final result or error to client.

### 4. Create Track (`/api/tracks/create-track`)
1. Client → `POST /api/tracks/create-track`  
2. Gateway applies `createTrack-pipeline-policy`:  
   - Proxy to Backend: `POST http://backend:8001/api/tracks/create-track`.  
   - Read `res.locals.proxyResponse.body`.  
   - Return 200 + data to client.  
   - In background, notify Recommender:  
     - `POST http://nginx:80/api/add_track` with `{ track_id, track_file_name }`  
     - Retry up to 3 times (500 ms delay).

### 5. Other Backend Requests (`/api/*`)
- All other paths are proxied directly to `http://backend:8001`.

---

## Containerization & Deployment

- All components are packaged as Docker containers.  
- **Docker Compose** defines and runs the multi-container application.  
- Simplifies scaling, management, and full-system deployments.

| Service       | Internal Address      | External Address        |
|---------------|-----------------------|-------------------------|
| postgres      | `postgres:5432`       | —                       |
| redis         | `redis:6379`          | —                       |
| nginx         | `nginx:80`            | —                       |
| mssql         | `mssql:1433`          | —                       |
| backend       | `backend:8001`        | —                       |
| api_gateway   | `api_gateway:8080`    | `localhost:8080`        |
| frontend      | `frontend:5173`       | `localhost:5173`        |
| zookeeper     | `zookeeper:2181`      | —                       |
| kafka         | `kafka:9092`          | —                       |
| recommender1  | `recommender1:8000`   | —                       |
| recommender2  | `recommender2:8000`   | —                       |

---

## Running the Project

1. Place `extractor_model.pkl` in `recommender/static/matrix/`.  
2. Replace `database/mssql/script.sql` with your data-generation script (keep the name `script.sql`).  
3. Replace `database/postgre/init-db.sql` with your initialization script (keep the name `init-db.sql`).  
4. Run:
   ```bash
   docker-compose up
