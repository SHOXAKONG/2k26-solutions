# System Requirements and Architecture

**Project topic:** Warehouse Robot Control Dashboard — a web-based operations console that lets warehouse staff monitor a fleet of autonomous mobile robots (AMRs) in real time, assign and reprioritize pick/move tasks, see live positions on a map of the warehouse floor, react to alerts (collisions, low battery, blocked paths), and pull historical performance reports.

---

## Stage 1. Requirements

### Functional requirements

1. **Real-time fleet visualization.** The dashboard must display every robot's live position, heading, battery level, and current task on a 2D map of the warehouse floor, with updates streamed at least once per second.
2. **Task assignment and reprioritization.** Operators must be able to create new pick/move tasks, assign them to a specific robot or to the fleet's auto-dispatcher, change task priority, and cancel in-flight tasks from the UI.
3. **Manual override / teleoperation.** An authorized operator must be able to take manual control of any single robot (stop, resume, nudge in a direction, send-home) directly from the dashboard, overriding the autonomous controller.
4. **Alerting.** The system must detect and surface critical events — collision, emergency stop, battery below threshold, robot offline > N seconds, blocked path — as visible banners in the UI and as notifications to a configured channel (email / Slack / on-call pager).
5. **Historical reporting.** Operators must be able to query historical data — tasks completed per robot per shift, average pick time, downtime by cause, battery cycles — over arbitrary date ranges and export results to CSV.

### Non-functional requirements

1. **Latency.** The end-to-end delay from a robot publishing telemetry to the value appearing on the operator's screen must be ≤ 500 ms at the 95th percentile, so manual override decisions are based on near-live data.
2. **Availability.** The dashboard must be reachable 99.9% of the time during warehouse operating hours; an outage longer than 1 minute must trigger an automatic failover to a standby instance.
3. **Scalability.** A single deployment must support at least 200 concurrent robots and 30 concurrent operator sessions without exceeding the 500 ms latency target.
4. **Security.** All operator actions (task creation, manual override, emergency stop) must require authenticated sessions with role-based authorization, and every action must be written to a tamper-evident audit log retained for at least 1 year.
5. **Safety / fail-safe behavior.** If the dashboard loses its connection to a robot for more than 2 seconds, the UI must clearly mark that robot as "stale" and the backend must instruct the robot to enter a safe-stop state until the link is restored.

---

## Stage 2. Architecture

Three architecture options are compared below.

### Option A — Monolithic Web App with WebSockets (single backend, direct robot link)

A single backend process speaks MQTT/ROS to the robot fleet on one side and serves the React dashboard plus a WebSocket channel to the browser on the other. PostgreSQL stores tasks, users, audit logs, and historical telemetry.

#### Diagram

```
┌──────────────────────┐    HTTPS + WebSocket    ┌────────────────────────────────────┐
│  Operator Browser    │ ──────────────────────▶ │  Reverse Proxy (Nginx / Caddy)     │
│  (React dashboard)   │                         │  - TLS, auth cookie check          │
│                      │ ◀────────────────────── └─────────────┬──────────────────────┘
└──────────────────────┘                                       │
                                                               ▼
                                       ┌──────────────────────────────────────────────┐
                                       │  Warehouse Control Server (single container) │
                                       │  ┌────────────────────────────────────────┐  │
                                       │  │ REST API (tasks, users, reports)       │  │
                                       │  ├────────────────────────────────────────┤  │
                                       │  │ WebSocket hub (live telemetry → UI)    │  │
                                       │  ├────────────────────────────────────────┤  │
                                       │  │ Robot connector (MQTT / ROS bridge)    │  │
                                       │  ├────────────────────────────────────────┤  │
                                       │  │ Dispatcher (assign tasks to robots)    │  │
                                       │  ├────────────────────────────────────────┤  │
                                       │  │ Alerting (rules + notifier)            │  │
                                       │  └────────────────────────────────────────┘  │
                                       └──────────┬────────────────────┬──────────────┘
                                                  │                    │
                                                  ▼                    ▼
                                      ┌────────────────────┐  ┌─────────────────────┐
                                      │  Robot fleet       │  │  PostgreSQL         │
                                      │  (MQTT topics /    │  │  - tasks, users     │
                                      │   ROS messages)    │  │  - audit log        │
                                      └────────────────────┘  │  - telemetry history│
                                                              └─────────────────────┘
```

#### Advantages

- **Simple to build and operate.** One repo, one container, one deploy. A small team can ship the first version quickly.
- **Low latency.** Robot telemetry comes in, gets fanned out to WebSocket clients in the same process — no extra hops, easy to hit the 500 ms target on a single-site deployment.
- **Easy local development.** A simulated robot publisher plus `docker compose up` is enough to bring up the whole stack.
- **Atomic deploys.** Dispatcher logic, alert rules, and UI all ship together — no version skew between services.
- **Cheap.** One VM or one ECS task plus a managed PostgreSQL covers the entire system at the early stage.

#### Disadvantages

- **Single point of failure.** A crash in the dispatcher or the robot connector takes the operator UI down too, violating the 99.9% availability NFR.
- **Vertical scaling only.** All 200 robots and 30 operators share one process; the only way to grow is a bigger box.
- **Mixed workloads share resources.** Heavy historical-report queries can slow down the WebSocket hub and push live telemetry over the 500 ms latency budget.
- **Hard to extend.** Adding a second site, a different robot vendor, or a separate analytics pipeline means cramming more code into the same process.
- **Failover is coarse.** Hot-standby for one big process is harder than for small stateless services; warm-standby + DB replication is the realistic option, with a longer recovery time.

---

### Option B — Microservices with a Streaming Bus (Kafka/NATS + specialized services)

The system is split into small services connected by a streaming bus. Robots publish telemetry to the bus; specialized services (live state, dispatcher, alerting, reporting) consume from it. The UI talks to a thin API gateway over REST and to a WebSocket gateway for live data.

#### Diagram

```
┌──────────────────────┐    HTTPS + WebSocket    ┌────────────────────────────────┐
│  Operator Browser    │ ──────────────────────▶ │  API Gateway (REST + WS)       │
│  (React dashboard)   │                         │  - AuthN/AuthZ, rate limiting  │
└──────────────────────┘ ◀────────────────────── └──────┬────────────────┬────────┘
                                                       │ REST           │ WebSocket
                                                       ▼                ▼
                                            ┌────────────────┐  ┌─────────────────────┐
                                            │ Task Service   │  │ Live-State Service  │
                                            │ (CRUD tasks)   │  │ (in-memory fleet    │
                                            │                │  │  state, fans out    │
                                            │                │  │  to WS clients)     │
                                            └───────┬────────┘  └─────────┬───────────┘
                                                    │                     ▲
                                                    │                     │
                                                    ▼                     │
                                      ┌──────────────────────────────────────────────┐
                                      │   Streaming Bus (Kafka / NATS JetStream)     │
                                      │   topics: telemetry, commands, alerts, tasks │
                                      └──┬──────────────┬───────────────┬────────────┘
                                         ▲              │               │
                                         │              ▼               ▼
                            ┌────────────┴───────┐ ┌──────────────┐ ┌─────────────────┐
                            │ Robot Gateway      │ │ Dispatcher   │ │ Alert Service   │
                            │ (MQTT/ROS bridge)  │ │ (assigns,    │ │ (rules engine + │
                            │ telemetry ─► bus   │ │  rebalances) │ │  notifier)      │
                            │ bus ─► commands    │ └──────────────┘ └─────────────────┘
                            └─────────┬──────────┘
                                      │
                                      ▼
                              ┌─────────────────┐
                              │  Robot fleet    │
                              │  (AMRs)         │
                              └─────────────────┘

                            ┌─────────────────────┐   ┌─────────────────────┐
                            │ Reporting Service   │   │ Storage             │
                            │ (consumes bus,      │ ─▶│ - PostgreSQL (OLTP) │
                            │  builds aggregates) │   │ - TimescaleDB /     │
                            └─────────────────────┘   │   ClickHouse        │
                                                      │   (telemetry, OLAP) │
                                                      └─────────────────────┘
```

#### Advantages

- **Failure isolation.** If the reporting service crashes mid-report, live telemetry and manual override keep working. Each service can be restarted independently.
- **Independent scaling.** Live-State and the WebSocket gateway scale with operator count; the Robot Gateway scales with fleet size; the Reporting Service scales with query load.
- **Multi-site / multi-vendor ready.** Add a second Robot Gateway for a second site or vendor; the rest of the system doesn't care because it consumes from the bus.
- **Replayable history.** The bus retains telemetry, so a new consumer (e.g. ML model for predictive maintenance) can be added later by replaying past events.
- **Cleaner audit/security boundaries.** Each service has its own credentials, its own IAM-style role on the bus, and its own log stream.

#### Disadvantages

- **Operational complexity.** Running Kafka/NATS, multiple services, schema registry, and observability for all of them is real ops work. Wrong tool for a 5-person team or a 30-robot pilot.
- **Harder local development.** `docker compose up` now spins up 6+ services; debugging an end-to-end issue spans multiple log streams.
- **Eventual consistency gotchas.** A task created in the Task Service is "live" only after the Dispatcher consumes the event — UI needs to handle "pending" states explicitly.
- **More expensive infra.** A managed Kafka cluster (or self-hosted with replication) plus several service containers plus an OLAP store costs an order of magnitude more than Option A.
- **Higher network latency between hops.** Robot → Gateway → Bus → Live-State → WS → UI is more hops than Option A; achievable under 500 ms but needs care.

---

### Option C — Edge + Cloud Hybrid (on-site edge controller, cloud dashboard)

A small edge box inside the warehouse handles the safety-critical, low-latency path: it talks directly to robots, runs the dispatcher and the fail-safe logic, and serves a local LAN dashboard. A cloud component handles long-term storage, reporting, multi-site rollups, alert delivery to off-site channels, and remote access for managers. The two are connected by an authenticated tunnel; the edge keeps working when the cloud link is down.

#### Diagram

```
┌─────────────── Warehouse site (LAN) ────────────────────┐
│                                                         │
│  ┌───────────────────┐    HTTPS + WebSocket             │
│  │ On-site operator  │ ────────────────────────────┐    │
│  │ (browser, tablet) │                             │    │
│  └───────────────────┘                             ▼    │
│                                       ┌──────────────────────────────┐   ┌──────────── Cloud ─────────────┐
│  ┌────────────────┐  MQTT / ROS       │  Edge Controller (on-prem)   │   │                                │
│  │ Robot fleet    │ ◀────────────────▶│  - Robot connector           │   │  ┌──────────────────────────┐  │
│  │ (AMRs)         │                   │  - Dispatcher                │   │  │  Cloud Dashboard         │  │
│  └────────────────┘                   │  - Live-state + WS hub       │   │  │  (multi-site, remote     │  │
│                                       │  - Local fail-safe logic     │   │  │   operators, managers)   │  │
│                                       │  - Local SQLite (last 24 h)  │   │  └──────────┬───────────────┘  │
│                                       │  - Outbound sync agent       │   │             │                  │
│                                       └────────────┬─────────────────┘   │             ▼                  │
│                                                    │ TLS tunnel          │  ┌──────────────────────────┐  │
└────────────────────────────────────────────────────┼─────────────────────┘  │  Cloud Backend           │
                                                     └────────────────────────▶│  - Long-term storage     │
                                                                              │  - Reporting / analytics │
                                                                              │  - Off-site alerts       │
                                                                              │  - Multi-site rollup     │
                                                                              └──────────────────────────┘
```

#### Advantages

- **Best latency for the safety-critical path.** Live telemetry, manual override, and the 2-second safe-stop fail-safe all run on the LAN, well under 500 ms and unaffected by WAN jitter.
- **Survives internet outages.** If the cloud link or the ISP drops, the warehouse keeps running — only remote viewing and long-term reporting are degraded.
- **Natural multi-site scaling.** Each site gets its own edge controller; the cloud aggregates them. Adding site #2 doesn't touch site #1.
- **Strong separation of concerns.** Real-time control stays on-prem (where safety needs determinism); analytics and access management stay in the cloud (where elasticity matters).
- **Smaller blast radius.** A cloud-side compromise can't directly drive the robots — commands originate from the edge controller, which authenticates them locally.

#### Disadvantages

- **Two deployments to maintain.** Edge software and cloud software now have their own release cycles, and they have to stay compatible across upgrades.
- **On-site hardware to own.** The edge box is physical infrastructure: provisioning, monitoring, spare units, firmware updates — all real costs that Option A and Option B avoid.
- **Sync logic is non-trivial.** The outbound sync agent must handle reconnects, backfill after outages, conflict resolution, and order-of-events for reporting accuracy.
- **Harder to develop.** Engineers need an edge simulator and a cloud staging environment; "run it all on my laptop" is no longer free.
- **Overkill for a single small site.** If there is only one warehouse with a stable internet link, the edge tier adds complexity without much payoff.

---

### Which option is better and why

**Option C (Edge + Cloud Hybrid) is the better choice for a real warehouse robot control dashboard.**

Reasoning:

- The safety and latency requirements are the dominant constraints. A 500 ms p95 latency budget and a 2-second safe-stop fail-safe are *physical* requirements — robots move, people work around them — and they should not depend on the public internet. Putting the control loop on an edge controller inside the warehouse satisfies these by design, not by tuning.
- The 99.9% availability NFR is hard to meet honestly with Option A (one process, one machine, WAN-dependent), and is overkill-expensive to meet with Option B alone if a site loses internet. Option C degrades gracefully: cloud features disappear, the warehouse keeps running.
- Reporting, off-site alerting, and multi-site rollup are genuinely better in the cloud — elastic storage, easier remote access, central audit log. Option C keeps those there instead of forcing them onto an on-prem box.
- The main weaknesses of Option C — two deployments, on-site hardware, sync logic — are operational costs that scale well: they are paid once per site and amortized over the lifetime of the deployment. The benefits (safety, offline resilience, multi-site) compound with every additional site.

**However**, the right starting point depends on scope:

- For a **single small pilot (one site, ≤ 30 robots, one shift)**, start with **Option A** to validate the product. Keep the dispatcher and robot connector as cleanly separated modules so they can later be extracted.
- For a **production single-site deployment**, move to **Option C** before going live, even if the cloud component is minimal at first. The fail-safe and offline-survival benefits are too important to skip.
- **Option B** is the right intermediate target if the deployment is single-site but needs to ingest a high-volume historical telemetry stream for analytics or ML, *before* multi-site is on the roadmap.

So the recommended path is: **prototype on Option A, productionize on Option C, and adopt Option B's streaming bus inside the cloud tier of Option C when analytics demand it.**
