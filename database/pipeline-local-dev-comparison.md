# Pipeline Local Development — Postgres → PeerDB → ClickHouse

> **Status:** Draft for discussion
> **Date:** 2026-07-24
> **Author:** yingjie.ho@merquri.io
> **Related:**
> - ADR — "Adopt ClickHouse as OLAP data store"
> - [`clickhouse-local-dev-comparison.md`](./clickhouse-local-dev-comparison.md) — local options for the ClickHouse tier alone (this doc builds on it)
> **Scope:** How developers run the *full proposed data pipeline* locally. The prior doc covered the ClickHouse sink in isolation; this one covers the end-to-end stack.

---

## 1. Context

The proposed production architecture is a three-tier pipeline:

```
Postgres (OLTP)          PeerDB (CDC)              ClickHouse (OLAP)
AWS RDS / Aurora   ──►   self-hosted or ClickPipes   ──►   self-hosted or CH Cloud
                         (staging via S3)
```

Running this locally is not "run one service" — it's a **multi-service stack**, and PeerDB itself is a multi-container system, not a single process. This document compares realistic local-development solutions, scores them, and recommends a portfolio (different solutions for different developer tasks).

### Key facts that shape the options

- **PeerDB is open-source (ELv2) and free to self-host**, including the Enterprise Helm charts. Locally it runs via its `run-peerdb.sh` / Docker Compose, which starts *its own* catalog Postgres + **Temporal** + PeerDB server + flow API/workers + **UI** (`localhost:3000`).
- **PeerDB stages data to S3; locally it uses MinIO** (S3-compatible) as a faithful stand-in.
- After ClickHouse's acquisition of PeerDB (2024), **ClickPipes is the managed implementation of the *same* PeerDB engine** in ClickHouse Cloud (Postgres CDC reached GA in May 2025). So **self-hosted PeerDB locally ≈ ClickPipes in prod** — strong parity either way.
- PeerDB maps source tables into **`ReplacingMergeTree`** (version column + soft-delete marker), so local runs exercise the exact merge/dedup semantics your analytical queries must handle.

---

## 2. What a local environment must include

| Tier | Production (proposed) | Local stand-in |
|---|---|---|
| OLTP source | RDS / Aurora Postgres | Postgres container with `wal_level=logical` |
| Pipeline | PeerDB (self-hosted) **or** ClickPipes (managed) | PeerDB Docker stack (catalog PG + Temporal + server + flow workers + UI) |
| Staging | S3 | **MinIO** container |
| OLAP sink | ClickHouse | ClickHouse container (see companion doc) |

---

## 3. Solutions considered

| # | Solution | Exercises PeerDB CDC? | Local? |
|---|----------|:--------------------:|:------:|
| A | **ClickHouse-only, seeded directly** (Tier 0) — skip PG & PeerDB; load sample/synthetic data straight into CH | ❌ | ✅ |
| B | **Postgres + ClickHouse, direct load** — move PG data into CH via dumps or CH's `postgresql()` table function; no PeerDB | ❌ | ✅ |
| C | **Full pipeline Docker Compose** (Tier 1) — source PG + PeerDB stack + MinIO + ClickHouse | ✅ | ✅ |
| D | **ClickPipes managed** (Tier 2) — ClickPipes against a reachable Postgres → ClickHouse Cloud | ✅ | ❌ |

---

## 4. Evaluation criteria

| Criterion | Weight | Why it matters here |
|-----------|:------:|---------------------|
| **Pipeline / CDC fidelity** | 5 | Does it actually exercise PeerDB CDC (snapshot + streaming, dedup, deletes)? The whole reason the pipeline exists. |
| **Production parity (overall)** | 4 | How closely the local stack behaves like the proposed prod topology. |
| **Team reproducibility** | 4 | Identical environment for every dev and for CI, from the repo. |
| **Setup / maintenance effort** | 3 | Friction to stand up and keep running. (Higher = less effort.) |
| **Resource footprint** | 3 | This stack can be *heavy*; laptop RAM/CPU is a real constraint. (Higher = lighter.) |
| **Iteration speed** | 3 | Fast inner loop for day-to-day work. |
| **Windows fit** | 2 | Our machines are Windows 11 (Docker needs a WSL2/Hyper-V backend). |
| **Cost / licensing** | 2 | No recurring cost or per-seat licensing traps. |
| **Offline capability** | 2 | Works without network or a cloud account. |

> Weights are a starting point; re-weight with the team. **Hard requirements gate before scoring** (§6.1).

---

## 5. Solution-by-solution analysis

### A. ClickHouse-only, seeded directly *(Tier 0 — the daily driver)*
**Pros:** lightest possible; fast inner loop; no pipeline moving parts; covers the ~80% case (schema modeling, query tuning, dashboards). Reuses the ClickHouse setup from the companion doc (Docker or chDB).
**Cons:** exercises **no** CDC — you hand-shape data instead of receiving it from PeerDB, so you can miss real type-mapping and `ReplacingMergeTree` dedup behavior unless you deliberately mimic it.
**Use when:** working on the ClickHouse side only.

### B. Postgres + ClickHouse, direct load (no PeerDB)
**Pros:** realistic *source* schema/types in Postgres; moderate footprint; CH can pull directly via the `postgresql()` table function or `PostgreSQL` engine for convenient seeding.
**Cons:** **this is not the production pipeline** — it's batch/pull, not PeerDB CDC. No streaming, no soft-delete semantics, no staging layer. Easy to build habits that don't survive contact with PeerDB.
**Use when:** you want real source data shapes but don't need CDC behavior. *(Note: avoid depending on `MaterializedPostgreSQL` — it's experimental and not the proposed pipeline.)*

### C. Full pipeline Docker Compose *(Tier 1 — the reference environment)*
**Pros:** **highest fidelity** — real PeerDB CDC (initial snapshot + streaming), real staging via MinIO, real `ReplacingMergeTree` mapping, the PeerDB UI, mirror configuration, schema evolution. Mirrors the **self-hosted PeerDB** prod path almost exactly, and closely approximates ClickPipes. One checked-in Compose file = reproducible for the team and CI.
**Cons:** **heavy** — ~8–10 containers (source PG, catalog PG, Temporal + deps, PeerDB server, flow API, flow worker, PeerDB UI, MinIO, ClickHouse). Real RAM/CPU on a laptop; slower to bring up; more to maintain. Docker Desktop licensing applies at company scale (mitigate with Podman/Rancher Desktop — see companion doc).
**Use when:** working *on the pipeline itself*, or running end-to-end integration tests.

### D. ClickPipes managed *(Tier 2 — managed-path validation, not local)*
**Pros:** same PeerDB engine, managed; nothing to run locally; matches prod exactly **if** prod is ClickHouse Cloud + ClickPipes.
**Cons:** **not local, not offline**; needs a reachable Postgres (RDS, or a tunnelled local PG) and a ClickHouse Cloud account; trial is 30 days / $300 then paid. Shared instance state isn't isolated per dev.
**Use when:** validating managed-only behavior, *only if* the production sink is ClickHouse Cloud.

---

## 6. Decision matrix

Scores **1–5 (5 = best)**. Weighted total = Σ(score × weight). Max = 140.

| Criterion (weight) | A. CH-only | B. PG+CH direct | C. Full pipeline | D. ClickPipes* |
|---|:--:|:--:|:--:|:--:|
| Pipeline / CDC fidelity (5) | 1 | 2 | 5 | 5 |
| Production parity (4) | 2 | 2 | 5 | 4 |
| Team reproducibility (4) | 5 | 4 | 5 | 3 |
| Setup / maintenance (3) | 5 | 4 | 2 | 4 |
| Resource footprint (3) | 5 | 4 | 1 | 5 |
| Iteration speed (3) | 5 | 4 | 2 | 3 |
| Windows fit (2) | 3 | 3 | 3 | 5 |
| Cost / licensing (2) | 5 | 5 | 4 | 2 |
| Offline capability (2) | 5 | 5 | 5 | 1 |
| **Weighted total** | **104** | **96** | **104** | **105\*** |

\* The three leaders are within a rounding error **because they serve different jobs** — this is not a single-winner decision. The gating filter below is what actually assigns each solution to its role.

### 6.1 Hard requirements (gating filter)

| Requirement | A | B | C | D |
|---|:--:|:--:|:--:|:--:|
| Runs **locally / offline** | ✅ | ✅ | ✅ | ❌ |
| Exercises **real PeerDB CDC** | ❌ | ❌ | ✅ | ✅ |

→ **Local CDC/pipeline work → only C qualifies.** **Daily local dev without CDC → A.** **Managed-path validation → D (non-local).**

---

## 7. Production parity gaps (write these into the runbook)

The "works locally, breaks in prod" traps specific to this stack:

1. **RDS/Aurora enablement can't be reproduced locally.** Local Postgres sets `wal_level=logical` in one line. RDS/Aurora needs a **custom parameter group** (`rds.logical_replication=1`), the **`rds_replication`** role, `max_replication_slots` / `max_wal_senders` tuning, and an **instance reboot**. Maintain this as a documented runbook — local tests won't cover it.
2. **Aurora ≠ RDS ≠ local Postgres.** Aurora has a known logical-replication **data-loss caveat** (`rds.logical_wal_cache=0` recommended) and storage-layer differences. Decide **RDS vs. Aurora deliberately** — Aurora's CDC story carries more caveats.
3. **Self-hosted PeerDB (local) vs. ClickPipes (managed)** share an engine but not the same configuration surface — close, not identical.
4. **MinIO ↔ S3** is a faithful local substitution for the staging layer — a genuine strength.
5. **Version skew:** pin local source Postgres to the **RDS/Aurora major version**, and ClickHouse to the **LTS line** (companion doc).

---

## 8. Recommendation — a portfolio, not one winner

- **Default developer loop = A (ClickHouse-only, seeded).** Fast, cheap, covers most work. Reuse the Docker/chDB setup from the companion doc.
- **One checked-in C (full pipeline) Compose stack** as the shared reference environment for pipeline/CDC work and CI integration tests. Accept that it's heavy; don't make it the everyday loop.
- **D (ClickPipes) only if** the production sink is ClickHouse Cloud — for validating managed-only behavior.
- **Skip B** as a standard workflow; it builds habits the real pipeline won't honor. Use `postgresql()` only for occasional seeding.
- **Match versions to prod** and keep the **RDS/Aurora enablement runbook** (§7) separate from local setup.

---

## 9. Open questions / next steps

- [ ] **Production topology:** self-hosted PeerDB + self-hosted ClickHouse, or ClickPipes + ClickHouse Cloud? This decides whether C or D is the parity reference.
- [ ] **RDS vs. Aurora** for the OLTP source — resolve early (§7.2).
- [ ] **Author the Tier-1 `docker-compose.yml`** (source PG with `wal_level=logical` + PeerDB stack + MinIO + ClickHouse) + a seed script + a smoke test, and check it into the repo.
- [ ] **Define the CI integration test:** stand up C, run a mirror, assert rows land in ClickHouse with correct dedup/delete behavior.
- [ ] **Laptop resourcing:** confirm the full stack runs acceptably on team hardware; document minimum RAM.
- [ ] **Re-weight the matrix (§4)** with the team.

---

## Sources

- [ClickHouse acquires PeerDB](https://clickhouse.com/blog/clickhouse-acquires-peerdb-to-boost-real-time-analytics-with-postgres-cdc-integration)
- [Postgres CDC in ClickHouse — a year in review](https://clickhouse.com/blog/postgres-cdc-year-in-review-2025)
- [PeerDB → ClickHouse CDC setup (PeerDB Docs)](https://docs.peerdb.io/mirror/cdc-pg-clickhouse)
- [PeerDB — GitHub](https://github.com/PeerDB-io/peerdb)
- [PostgreSQL + ClickHouse as the OSS unified data stack](https://clickhouse.com/blog/postgres-clickhouse-oss)
- [Simple Postgres→ClickHouse replication featuring MinIO](https://blog.peerdb.io/simple-postgres-to-clickhouse-replication-featuring-minio)
- [Migrate PostgreSQL data using PeerDB (ClickHouse Docs)](https://clickhouse.com/docs/cloud/managed-postgres/migrations/peerdb)
- [Setting up logical replication for Aurora PostgreSQL (AWS)](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Replication.Logical.Configure.html)
- [Enable CDC on RDS/Aurora PostgreSQL (RisingWave docs)](https://docs.risingwave.com/ingestion/sources/postgresql/aws-rds)
