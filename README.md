# Attribyte

**Mobile install attribution platform.** Answers the question every mobile team asks: *where did this install come from — organic search, paid campaign, or referral link?* Kotlin Android SDK captures the Play Install Referrer + in-app events, streams them to a Kafka-backed pipeline, and surfaces them in a React admin dashboard with segmentation by OS, device, app version, and geography. Multi-tenant from day one.

---

## Architecture — 3 components across 3 repos

```
                            ┌────────────────────────────────────┐
                            │      Kotlin Android SDK            │
                            │                                    │
                            │  Attribyte.init(context) { … }     │
                            │  Attribyte.track("event", props)   │
                            │                                    │
                            │  On init: reads Play Install       │
                            │  Referrer API → attribution source │
                            │  On event: POST /v1/events         │
                            └───────────────┬────────────────────┘
                                            │
                                            │  HTTPS
                                            ▼
                            ┌────────────────────────────────────┐
                            │      attribyte-api (Node.js/TS)    │
                            │                                    │
                            │  POST /v1/installs (attribution)   │
                            │  POST /v1/events (in-app events)   │
                            │  Multi-tenant (tenant_id per req)  │
                            │  Validates → publishes to Kafka    │
                            └───────────────┬────────────────────┘
                                            │
                                            │  produce
                                            ▼
                                  ┌─────────────────┐
                                  │  Kafka          │
                                  │  events.installs│
                                  │  events.actions │
                                  └────────┬────────┘
                                           │
                                           │  consume
                                           ▼
                            ┌────────────────────────────────────┐
                            │  Ingestion Worker (Node.js/TS)     │
                            │  Writes to PostgreSQL              │
                            │  Aggregates hourly + daily rollups │
                            └───────────────┬────────────────────┘
                                            │
                                            ▼
                                  ┌─────────────────┐
                                  │   PostgreSQL    │
                                  │   installs      │
                                  │   events        │
                                  │   rollups       │
                                  └────────┬────────┘
                                           │
                                           │  REST query
                                           ▼
                            ┌────────────────────────────────────┐
                            │   attribyte-web (Next.js React)    │
                            │                                    │
                            │  Marketing site (public)           │
                            │  Admin dashboard (authed):         │
                            │    - Install funnel by source      │
                            │    - Segmentation: OS / device /   │
                            │      app version / geography       │
                            │    - Real-time counter             │
                            └────────────────────────────────────┘
```

## The three repos

| Repo | Purpose | Tech |
|---|---|---|
| [**`attribyte-web`**](https://github.com/Dev-Harsh0218/attribyte-web) | Marketing site + React admin dashboard | Next.js 16 (App Router, Turbopack), React 19, TypeScript, Tailwind v4, Framer Motion, lucide-react |
| [**`attribyte-api`**](https://github.com/Dev-Harsh0218/attribyte-api) | Backend — REST ingestion + Kafka producer + Postgres consumer | Node.js 20+, Express 4, TypeScript (ES modules, NodeNext), Kafka (kafkajs), PostgreSQL (pg), Zod, Pino, Helmet, CORS |
| [**`attribyte-platform`**](https://github.com/Dev-Harsh0218/attribyte-platform) | Meta-repo (this) — architecture, cross-repo docs | — |

## The flow

1. Mobile app integrates the Kotlin Android SDK: `Attribyte.init(context) { apiKey = "..." }`
2. On first install, SDK reads **Google Play Install Referrer API** (~50ms after app open) → gets the referrer URL that led to the install
3. SDK parses UTM params from the referrer → classifies as **organic search / paid campaign / referral link**
4. SDK sends `POST /v1/installs` to `attribyte-api` with the attribution + device metadata
5. `attribyte-api` validates the payload (Zod), stamps `tenant_id`, publishes to Kafka `events.installs` topic
6. Ingestion worker consumes the topic, writes to Postgres `installs` table, updates hourly/daily rollups
7. React dashboard queries the API for aggregations, renders install funnel + segmentation charts
8. In-app events (`Attribyte.track(...)`) follow the same shape via `events.actions` topic

## Design decisions worth defending

| Decision | Why |
|---|---|
| **Kafka between API and DB writer** | Absorbs traffic spikes without dropping events. Consumer can scale independently of the HTTP surface. Replayable — if the DB writer has a bug, we replay from Kafka after fix instead of losing data. |
| **Play Install Referrer API (not fingerprinting)** | Official Google API, deterministic, first-party. Fingerprinting (IP + device model + screen size hash) is fragile and increasingly blocked by Play policies. |
| **Multi-tenant with `tenant_id` at every layer** | Every event carries `tenant_id`. Same DB, same Kafka topics, tenant-scoped queries. Isolates billing + data access without per-tenant infrastructure. |
| **PostgreSQL rollups (not real-time OLAP)** | Dashboard queries hit pre-aggregated hourly/daily rollups, not raw events. Sub-second dashboard performance without needing ClickHouse or Druid at this scale. Migrate later if volume warrants. |
| **Kotlin SDK (not Java)** | Modern Android tooling. Null-safety, coroutines for async I/O, smaller compiled size. Java SDKs are legacy; new Android greenfield is Kotlin. |
| **REST over gRPC for SDK ↔ API** | HTTP/S survives every corporate proxy, every CDN, every carrier. gRPC would be faster but adds ops complexity for zero benefit at this event volume. |
| **Zod at every ingress boundary** | Type-safe validation for env config + request bodies + Kafka messages. Parse errors become 400s automatically. Same tool for schemas across the codebase. |

## Live URLs

| Component | URL | Status |
|---|---|---|
| Marketing site + dashboard demo | _pending Vercel deploy_ | Building |
| API | _localhost only_ | Scaffold complete, needs Kafka + Postgres provisioned for real deploy |
| Kotlin Android SDK | _not published_ | Reference implementation in `attribyte-api/sdk/` (planned) |

## Build status

- ✅ **Platform meta-repo** (this repo) — architecture doc
- 🟡 **attribyte-web** — Next.js marketing site + dashboard preview (in progress)
- 🟡 **attribyte-api** — Node.js + Express + Kafka scaffold (in progress)
- ⏳ **Android SDK** — reference implementation planned (not required for portfolio demo)

## What I'd do differently at 10× scale

- Move raw event storage from Postgres to **ClickHouse** (or S3 + Athena) — Postgres rollups work up to ~1M events/day; beyond that, columnar storage wins
- Add **schema registry** on the Kafka side (Confluent or Redpanda) so producer/consumer schemas stay in sync as fields evolve
- Add **exactly-once semantics** with Kafka transactions if billing accuracy becomes a requirement
- Consider **CDN-hosted SDK config** so SDK behavior can be tuned without app-store re-releases

## Related

Built by [Harsh Bhardwaj](https://github.com/Dev-Harsh0218) — full-stack engineer working across Node.js, Django, React, and TypeScript.
