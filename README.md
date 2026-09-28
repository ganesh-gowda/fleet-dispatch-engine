# Fleet Dispatch Engine

Real-time, event-driven fleet dispatch and recovery platform built on a realistic Bengaluru road network.

**Stack:** Java 21, Spring Boot, MongoDB Atlas, Redis Cloud, Confluent Cloud Kafka, Next.js, TypeScript, Leaflet, WebSockets.

## What it does

The engine continuously manages drivers and delivery orders and handles normal dispatch as well as driver failures during active deliveries.

Core capabilities include:

- Dijkstra and A* routing over the Bengaluru road graph
- Spatial driver discovery with bounded Top-K candidate selection
- Pluggable dispatch scoring and route-aware selection
- Global Hungarian assignment for batch dispatch
- Route insertion and delivery consolidation
- Concurrent assignment protection
- Live driver location tracking with sequence ordering and stale/duplicate rejection
- Heartbeat-based failure detection
- Picked-up order recovery and replacement-driver selection
- Per-driver recovery serialization and a recovery queue/worker
- Kafka event transport with an outbox and consumer idempotency
- MongoDB persistence and Redis live-location TTL state
- REST APIs, OpenAPI documentation, readiness/liveness probes, metrics, and a Next.js command-center dashboard

## Failure recovery lifecycle

The main end-to-end scenario is:

```text
AVAILABLE
   ↓
ORDER CREATED
   ↓
ASSIGNED
   ↓
PICKED UP
   ↓
DRIVER FAILURE
   ↓
OFFLINE / RECOVERY REQUIRED
   ↓
NEAREST FEASIBLE REPLACEMENT
   ↓
HANDOFF
   ↓
PICKUP
   ↓
COMPLETED
```

The recovery flow is executed by the backend runtime; the dashboard visualizes the resulting state and events.

## Architecture

```text
                Bengaluru Road Network
                         │
                         ▼
                Routing + Dispatch Engine
                         │
              ┌──────────┼───────────┐
              ▼          ▼           ▼
           MongoDB     Redis       Kafka
          persistence  live state  event stream
              │          │           │
              └──────────┼───────────┘
                         ▼
                  Spring Boot APIs
                         │
                  WebSocket servers
                         │
                         ▼
                Next.js Command Center
                         │
                         ▼
                   Leaflet map/UI
```

## Repository layout

```text
backend/    Spring Boot application, routing, dispatch, recovery, APIs and persistence
dashboard/  Next.js + TypeScript operations dashboard
data/       Local road-network data (when present locally)
docs/       Architecture and deployment documentation
tools/      Development and simulation utilities
pom.xml     Root Maven build
```

## Running locally

### Backend

Requirements: Java 21, Maven, and access to the configured MongoDB/Redis/Kafka services.

```powershell
mvn spring-boot:run
```

### Dashboard

```powershell
cd dashboard
npm install
npm run dev
```

The dashboard reads its backend and WebSocket URLs from `NEXT_PUBLIC_*` environment variables. Example templates are provided in `.env.example` and `dashboard/.env.example`.

## Validation

Backend tests:

```powershell
mvn -B test
```

Dashboard checks:

```powershell
cd dashboard
npm run typecheck
npm run build
```

GitHub Actions runs the backend test/package pipeline and the dashboard typecheck/build pipeline on pushes and pull requests to `main`.

## Performance & benchmarking

A **30.39× improvement** was recorded during local optimization of driver candidate discovery. This is a local benchmark result, not a production latency or throughput guarantee; results depend on the machine, JVM, and workload. The repository contains benchmark harnesses to inspect and reproduce the relevant comparisons.

| Benchmark | What it measures | Workload / configuration |
|---|---|---|
| Driver candidate discovery ([source](backend/src/main/java/com/ganesh/fleetdispatch/benchmark/DriverCandidateDiscoveryBenchmark.java)) | Full scan vs. grid-backed lookup; reports median and p95 and checks that both approaches return the same candidate IDs | 10,000 and 100,000 drivers; fixed seed; 1 km radius; top 20 candidates; 10 warm-up and 30 measured iterations |
| Concurrent candidate discovery ([JMH source](backend/src/main/java/com/ganesh/fleetdispatch/benchmark/DriverCandidateDiscoveryJmhBenchmarkFixed.java)) | Full scan vs. grid lookup, read-only concurrency, location updates, and mixed lookup/update activity | 10,000 and 100,000 drivers; JMH sample-time mode; 3 × 1 s warm-up, 5 × 1 s measurement, 2 forks; mixed group uses 7 lookup threads and 1 update thread |
| Spatial index comparison ([JMH source](backend/src/main/java/com/ganesh/fleetdispatch/benchmark/SpatialIndexComparisonJmhBenchmark.java)) | Grid vs. H3 nearby-driver lookup | 10,000 and 100,000 drivers; 2 km radius; top 20 candidates; JMH sample-time mode |
| Batch dispatch ([JMH source](backend/src/main/java/com/ganesh/fleetdispatch/benchmark/BatchDispatchJmhBenchmark.java)) | Repeated greedy dispatch vs. global minimum-cost batch assignment | Batch sizes of 10, 50, and 100; synthetic routing fixture |
| Nearest-node lookup ([source](backend/src/main/java/com/ganesh/fleetdispatch/graph/NodeLocatorComparisonBenchmark.java)) | Brute-force lookup vs. KD-tree lookup; reports index build time and lookup average, p50, p95, and p99 | Real road graph supplied as `nodes.csv` and `edges.csv`; 10 fixed geographic queries |

Run the deterministic driver-discovery comparison from the repository root:

```powershell
mvn -B -DskipTests compile exec:java -Dexec.mainClass=com.ganesh.fleetdispatch.benchmark.DriverCandidateDiscoveryBenchmark
```

The JMH harnesses are available in the backend benchmark package. The repository does not currently publish a production load-test result for end-to-end assignment throughput or API p95 latency; those should be measured separately rather than inferred from microbenchmarks.

## API and operations endpoints

Useful endpoints include:

```text
GET  /api/v1/health
GET  /api/v1/dashboard/summary
GET  /api/v1/events/recent?limit=N
GET  /api/v1/map/geojson
GET  /actuator/health/readiness
GET  /swagger-ui.html
GET  /v3/api-docs
```

## Deployment

The recommended public-demo deployment is:

```text
Next.js dashboard  → Vercel
Spring Boot        → Railway
MongoDB            → MongoDB Atlas
Redis              → Redis Cloud
Kafka              → Confluent Cloud
```

See [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) for the exact environment variables, Railway/Vercel settings, WebSocket exposure, and post-deploy validation checklist.

The project is intended to be a **production-deployable demonstration environment**. It should not be presented as a production fleet serving real customers without additional authentication/authorization, rate limiting, stronger multi-instance coordination, and a formal security/load review.

## Engineering focus

This project is intentionally built as a systems-and-algorithms project rather than a CRUD application. The emphasis is on routing, optimization, concurrent state transitions, event-driven processing, failure recovery, persistence, observability, correctness testing, deterministic simulation, and reproducible benchmarks.
