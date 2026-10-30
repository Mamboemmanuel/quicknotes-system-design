### `docs/architecture.md`

```markdown
# QuickNotes System Architecture Specification

This document details the scalability architecture for QuickNotes to support 1 million users.

## 1. Functional & Non-Functional Requirements

### Functional Requirements
- Users can create, view, update, and delete text notes.
- Users can tag and filter notes.
- Notes list loads in under 200ms.

### Non-Functional Requirements
- **Scalability:** System supports 1 million registered users.
- **Availability:** 99.9% uptime (max ~8.76 hours downtime/year).
- **Data Durability:** Zero note data loss across hardware failures.
- **Low Latency:** Global read latency < 100ms; write latency < 300ms.

---

## 2. Capacity Planning & Load Estimates

Assuming **1,000,000 active users**:
- **Daily Active Users (DAU):** 200,000 (20% of total)
- **Read Operations:** Average 10 note fetches/user/day = 2,000,000 reads/day.
  - **Reads per Second (Average):** $2,000,000 / 86,400 \approx 23.15$ RPS.
  - **Peak Read RPS (3x average):** $\approx 70$ RPS.
- **Write Operations:** Average 2 notes created/user/day = 400,000 writes/day.
  - **Writes per Second (Average):** $400,000 / 86,400 \approx 4.63$ WPS.
  - **Peak Write WPS (3x average):** $\approx 14$ WPS.
- **Storage Estimates per Year:**
  - Average note size: 1 KB (title + body metadata).
  - New notes per day: 400,000.
  - Daily storage: $400,000 \times 1\text{ KB} = 400\text{ MB/day}$.
  - Yearly storage: $400\text{ MB} \times 365 = 146\text{ GB/year}$.

---

## 3. System Architecture Diagram

[ Client (Browser/App) ]
│
▼
[ DNS ]
│
▼
[ CDN ] ──(Static Assets)
│
▼
[ Load Balancer (NGINX) ]
│             │
▼             ▼
[ App Server 1 ] [ App Server 2 ]
│    │          │    │
│    ├──────────┘    │
│    ▼               ▼
│  [ Redis Cache ] [ Message Queue (RabbitMQ) ]
│         │                        │
▼         │                        ▼
[ Primary DB (SQL) ] ◄── Read Sync ── [ Background Worker ]
│
▼ (Replication)
[ Read Replica DB ]


---

## 4. Component Justifications

- **Client:** Provides user interface and client-side validation.
- **DNS:** Resolves domain names to load balancer IPs via latency-based routing.
- **CDN:** Caches static assets (`index.html`, `style.css`, static JS) close to users globally.
- **Load Balancer (NGINX):** Distributes incoming HTTPS traffic across stateless application instances.
- **App Servers (Stateless Node.js/Go):** Handles business logic, input validation, and API serialization.
- **Redis Cache:** Caches frequently accessed user note lists in memory to drastically reduce read queries on SQL databases.
- **Primary Database (SQL Primary):** Handles all write queries (INSERT, UPDATE, DELETE) and maintains authoritative transactional state.
- **Read Replica DB:** Offloads read-only queries from the primary database to handle high scale.
- **Message Queue (RabbitMQ/Kafka):** Buffers heavy asynchronous workloads (e.g., sending email alerts, generating statistics).
- **Background Worker:** Consumes tasks from the queue and processes background jobs asynchronously.

---

## 5. End-to-End Request Flows

### Flow 1: `GET /notes`
1. Client requests `GET /api/v1/notes` via HTTP to Load Balancer.
2. Load Balancer routes the request to an available Stateless App Server.
3. App Server checks **Redis Cache** for key `user:{id}:notes`.
4. *Cache Hit:* App Server returns cached JSON immediately to client (<20ms).
5. *Cache Miss:* App Server queries **Read Replica DB**, stores result in **Redis Cache**, and returns response to client.

### Flow 2: `POST /notes`
1. Client sends payload to `POST /api/v1/notes`.
2. Load Balancer routes payload to App Server.
3. App Server validates input (Title length $\le 100$).
4. App Server writes new record to **Primary SQL DB**.
5. App Server invalidates/updates key `user:{id}:notes` in **Redis Cache**.
6. App Server pushes a background job event (`NoteCreated`) to the **Message Queue**.
7. App Server immediately returns `201 Created` status code to client.
8. **Worker** asynchronously consumes the queue event for secondary processing (e.g., search indexing).

---

## 6. System Trade-Offs & Single Points of Failure (SPOF)

### System Trade-Offs
1. **Cache Consistency vs Read Performance:** We invalidate the cache on every write. This guarantees high read performance at the expense of brief cache invalidation overhead on updates.
2. **Synchronous Writes vs Eventual Consistency for Processing:** Note creation writes synchronously to the primary database to guarantee durability, while secondary tasks (indexing, emails) run asynchronously via queues to minimize write latencies.

### Avoiding Single Points of Failure (SPOF)
- **App Server Failures:** Multiple stateless instances run behind the Load Balancer. If one crashes, health checks route traffic to surviving nodes.
- **Database Failures:** The SQL Primary continuously replicates to Read Replicas. Automatic failover (e.g., AWS Aurora / Orchestrator) promotes a Read Replica to Primary if the main node fails.