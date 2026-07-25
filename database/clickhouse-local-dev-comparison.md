# ClickHouse Local Development — Options Comparison & Decision Matrix

> **Status:** Draft for discussion
> **Date:** 2026-07-24
> **Author:** yingjie.ho@merquri.io
> **Related:** ADR — "Adopt ClickHouse as OLAP data store" (companion document)
> **Scope:** How developers run ClickHouse *locally* while building/evaluating on it. Does **not** cover production deployment topology (see ADR).

---

## 1. Context

We are evaluating ClickHouse as our OLAP data store. Before committing, developers need a way to run ClickHouse on their machines to model schemas, load sample data, test queries, and integrate it with our application (Node.js stack).

Two constraints shape the options:

1. **ClickHouse has no native Windows server build.** The server is a Linux binary. On Windows it runs via a Linux VM (Docker or WSL2). Embedded variants (chDB, `clickhouse-local`) are the exceptions.
2. **Local dev should track production.** The closer local behaves to the eventual production deployment, the fewer "works on my machine" surprises. This favors running the real server over embedded engines for integration work.

This document compares the realistic options, scores them against weighted criteria, and recommends a primary + complementary setup.

---

## 2. Options considered

| # | Option | What it is |
|---|--------|-----------|
| A | **Docker Compose** (`clickhouse/clickhouse-server`) | The official server container, orchestrated with a compose file checked into the repo. |
| B | **chDB** (embedded, Node.js bindings) | An in-process ClickHouse engine — like SQLite-for-OLAP. Imported as a library; no server, no ports. |
| C | **`clickhouse-local`** (standalone CLI) | The ClickHouse engine as a single-binary CLI. Runs full SQL over files/tables without a server. |
| D | **WSL2 + native server binary** | The real Linux `clickhouse-server` running inside WSL2 on Windows. |
| E | **ClickHouse Cloud dev instance** | *Not local* — a managed trial instance. Included as a baseline for comparison. |

---

## 3. Evaluation criteria

| Criterion | Weight | Why it matters here |
|-----------|:------:|---------------------|
| **Production parity** | 5 | We're evaluating for a production decision; local behavior must reflect the real engine/server. |
| **Team reproducibility** | 4 | Every dev (and CI) should get an identical environment from the repo. |
| **Setup / maintenance effort** | 3 | Low friction to onboard and keep running. (Higher score = less effort.) |
| **Windows fit / prerequisites** | 3 | Our machines are Windows 11; fewer/lighter prerequisites is better. |
| **Cluster / replication testing** | 3 | Ability to test sharding, replication, ReplicatedMergeTree, Keeper locally. |
| **Iteration speed (ad-hoc)** | 3 | Fast loop for exploring SQL, schemas, and sample data. |
| **Resource footprint** | 2 | RAM/CPU/disk cost on a dev laptop. (Higher score = lighter.) |
| **Cost / licensing freedom** | 3 | No per-seat licensing traps or recurring cost. |
| **Offline capability** | 2 | Works on a plane / without network or an account. |

> Weights are a starting point — adjust them in §5 to match how the team actually prioritizes. **Hard requirements gate before scoring** (see §5.1).

---

## 4. Option-by-option analysis

### A. Docker Compose — `clickhouse-server`
**The mainstream choice for prod-like local dev.**

**Pros**
- Runs the *exact same server binary* as production → highest parity.
- One `docker-compose.yml` in the repo = identical env for every dev and for CI.
- Can grow into a multi-node + ClickHouse Keeper topology to test replication/sharding.
- Persistent named volumes; trivial reset (`docker compose down -v`).
- Init SQL/scripts mount cleanly for seeding schemas and sample data.
- Fully offline once the image is pulled.

**Cons / risks**
- Requires **Docker Desktop**, which needs a Linux backend (WSL2 or Hyper-V) on Windows.
- **Licensing:** Docker Desktop is paid for orgs with **≥250 employees *or* ≥$10M annual revenue** (~$9–24/user/mo). *Mitigation:* use license-free alternatives — **Podman Desktop** or **Rancher Desktop** — which run the same containers without the Docker Desktop license.
- Container overhead (moderate RAM/CPU) vs. embedded options.

**Windows note:** Works well via WSL2 backend. Enable virtualization; expect a one-time WSL2 setup.

---

### B. chDB (embedded, Node.js bindings)
**Best for fast, throwaway SQL exploration and unit tests inside our Node app.**

**Pros**
- `npm i chdb` — zero server, zero ports, in-process.
- Instant iteration; tiny footprint; Apache-2.0, no licensing concerns.
- Node bindings are actively maintained (single N-API binary across Node 18/20/22 + Bun + Deno; typed errors; an experimental `@clickhouse/client` connection hook).
- Great for embedding OLAP queries directly in application code and for generating/validating synthetic data.

**Cons / risks**
- **Low production parity:** same query core, but no server, no server-side config, no MergeTree replication/merge semantics as a running service, no cluster behavior.
- **Windows-native support is the weak point** — prebuilt Node binaries historically target Linux/macOS; on Windows you may need WSL anyway. *Verify before relying on it.*
- Single process only — cannot test anything distributed.

---

### C. `clickhouse-local` (standalone CLI)
**A middle ground: the real engine, full SQL, no server.**

**Pros**
- Single binary; full ClickHouse SQL and functions; can create MergeTree tables persisted to disk (via a named database + `--path`).
- Excellent for ad-hoc analysis: query CSV/Parquet/JSON files directly, prototype schemas, run migration SQL.
- Free, light, offline.

**Cons / risks**
- **No native Windows binary → needs WSL2** to run.
- Single-process engine — no server lifecycle, no ports/clients-over-network, no replication/sharding.
- The *default* database is in-memory; you must create a named DB to persist — a common footgun.

---

### D. WSL2 + native server binary
**The real server, without Docker.**

**Pros**
- Runs the genuine Linux `clickhouse-server` → parity as high as Docker.
- No Docker Desktop licensing question.
- Full server: ports, clients, configs, and (with effort) replication.

**Cons / risks**
- **Manual, per-developer setup** → environment drift; weakest reproducibility. Scriptable but fragile.
- Manual service lifecycle management (start/stop, config, upgrades).
- Requires WSL2; carries a Linux VM's resource overhead.

---

### E. ClickHouse Cloud dev instance *(not local — baseline)*
**Included for comparison; fails the "local" requirement.**

**Pros**
- Zero local setup; browser/client only; matches *managed-cloud* production if that's our target.
- Real, fully-featured, managed cluster.

**Cons / risks**
- **Not local and not offline** — needs network + an account.
- **Cost:** 30-day / $300-credit trial, then pay-as-you-go. No permanent free tier.
- Shared instance state isn't isolated per developer without extra structure.

---

## 5. Decision matrix

Scores are **1–5 (5 = best)**. Weighted total = Σ(score × weight). Max possible = 140.

| Criterion (weight) | A. Docker Compose | B. chDB | C. clickhouse-local | D. WSL2 native | E. CH Cloud* |
|---|:--:|:--:|:--:|:--:|:--:|
| Production parity (5) | 5 | 2 | 3 | 5 | 5 |
| Team reproducibility (4) | 5 | 5 | 4 | 2 | 4 |
| Setup / maintenance (3) | 3 | 5 | 4 | 2 | 5 |
| Windows fit (3) | 3 | 3 | 2 | 3 | 5 |
| Cluster / replication (3) | 4 | 1 | 1 | 3 | 5 |
| Iteration speed (3) | 3 | 5 | 5 | 3 | 4 |
| Resource footprint (2) | 3 | 5 | 5 | 3 | 5 |
| Cost / licensing (3) | 3 | 5 | 5 | 5 | 2 |
| Offline capability (2) | 5 | 5 | 5 | 5 | 1 |
| **Weighted total** | **109** | **107** | **102** | **97** | **116\*** |

\* **ClickHouse Cloud tops the raw score but violates the hard "local + offline" requirement** — see §5.1. Among genuinely-local options, **Docker Compose leads.**

### 5.1 Hard requirements (gating filter)

Before scores matter, an option must pass these:

| Requirement | A | B | C | D | E |
|---|:--:|:--:|:--:|:--:|:--:|
| Runs **locally / offline** | ✅ | ✅ | ✅ | ✅ | ❌ |
| Runs a **real server** (network clients, config, ports) | ✅ | ❌ | ❌ | ✅ | ✅ |
| Can test **replication / cluster** | ✅ | ❌ | ❌ | ⚠️ | ✅ |

→ For **integration & prod-parity work**, only **A** and **D** clear the bar; **A** wins on reproducibility. For **fast ad-hoc exploration**, **B/C** are excellent complements.

---

## 6. Recommendation

**Primary: Docker Compose with `clickhouse/clickhouse-server`** — highest production parity among local options, one-file reproducibility for the whole team and CI, and a growth path to multi-node/replication testing.

- On Windows, use the **WSL2 backend**.
- To sidestep Docker Desktop licensing at company scale, standardize on **Podman Desktop** or **Rancher Desktop** (same compose files, no per-seat license).
- **Pin the image to the current LTS line** (LTS is 25.8, EOS ~Aug 29 2026; next LTS ~Aug 2026) for prod-parity; bump deliberately when a new LTS lands.

**Complement: chDB and/or `clickhouse-local`** for instant SQL exploration, prototyping schemas over sample files, and lightweight unit tests inside the Node app — where spinning up a server is overkill. (Confirm chDB's Windows-native support first; otherwise use it under WSL.)

**Optional: a short ClickHouse Cloud trial** *if* managed ClickHouse Cloud is the likely production target — to validate managed-specific behavior the local server won't reproduce.

---

## 7. Open questions / next steps

- [ ] **What is the production deployment target?** Self-hosted (Docker/K8s on Linux) vs. ClickHouse Cloud — this decides which local option maximizes parity.
- [ ] **Confirm Docker Desktop licensing applicability** for Merquri's size; if it applies, adopt Podman/Rancher Desktop.
- [ ] **Verify chDB Windows-native support** on our machines, or standardize embedded use under WSL.
- [ ] **Decide the reproducible artifact:** author the `docker-compose.yml` + init SQL + a Node client smoke test, and check it into the repo.
- [ ] **Re-weight the matrix (§3)** with the team and confirm the ranking holds.

---

## Sources

- [ClickHouse Docs — Deployment modes](https://clickhouse.com/docs/deployment-modes)
- [ClickHouse — Docker image (`clickhouse/clickhouse-server`)](https://hub.docker.com/r/clickhouse/clickhouse-server/)
- [ClickHouse Docs — chDB for Node.js](https://clickhouse.com/docs/chdb/install/nodejs)
- [chdb-io/chdb-node — GitHub](https://github.com/chdb-io/chdb-node)
- [ClickHouse Docs — chDB & clickhouse-local guide](https://github.com/ClickHouse/clickhouse-docs/blob/main/docs/chdb/guides/clickhouse-local.md)
- [Tinybird — ClickHouse vs chDB: embedded ClickHouse](https://www.tinybird.co/blog/clickhouse-vs-chdb-embedded-clickhouse)
- [How to Set Up a Local ClickHouse Development Environment (OneUptime)](https://oneuptime.com/blog/post/2026-01-21-clickhouse-local-dev-environment/view)
- [Docker Plans FAQ](https://www.docker.com/pricing/faq/) · [Docker licensing quick guide (USU)](https://www.usu.com/en/blog/quick-guide-to-docker-licensing)
- [ClickHouse — End of Life dates (endoflife.date)](https://endoflife.date/clickhouse)
- [ClickHouse Cloud](https://clickhouse.com/cloud) · [Pricing docs](https://clickhouse.com/docs/cloud/manage/billing/overview)
