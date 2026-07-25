# learning-docs

Technical decision documents — option comparisons, evaluation matrices, and ADR-style write-ups produced while designing and evaluating data infrastructure. Everything here is Markdown; there is no code to build or run.

Current focus is an **OLTP/OLAP split**: PostgreSQL as the transactional source, change data capture into ClickHouse as the analytical sink, and how developers run that pipeline locally on a Windows 11 / Node.js stack.

## Documents

### `database/`

| Document | What it covers |
|---|---|
| [`03a-aurora-vs-rds-postgres-wal.md`](./database/03a-aurora-vs-rds-postgres-wal.md) | Aurora PostgreSQL vs. RDS-for-PostgreSQL as a CDC source — WAL storage, logical replication slots, failover slot survival, data-integrity caveats. Verdict: default to RDS for CDC reliability. |
| [`03b-aurora-vs-rds-postgres-performance.md`](./database/03b-aurora-vs-rds-postgres-performance.md) | The performance counterpart to `03a` — pure OLTP runtime behaviour and price-performance, assuming analytics live on a separate OLAP engine. |
| [`clickhouse-local-dev-comparison.md`](./database/clickhouse-local-dev-comparison.md) | How developers run ClickHouse locally — options compared against weighted criteria, with a decision matrix and recommendation. |
| [`pipeline-local-dev-comparison.md`](./database/pipeline-local-dev-comparison.md) | Running the full Postgres → PeerDB → ClickHouse pipeline locally, including production-parity gaps to carry into a runbook. Recommends a portfolio of setups rather than one. |

Referenced but not yet in this repo: the ADR *"Adopt ClickHouse as OLAP data store"*.

## Reading a document

Each document opens with a blockquote header giving its **Status**, **Date**, **Author**, **Related** documents, and **Scope** — read the Scope first, since it states what the document deliberately leaves out and where that material belongs instead. Every factual claim traces to a link in the closing `## Sources` section. Vendor benchmark figures are directional, not guarantees.

Documents are marked *Draft for discussion* unless stated otherwise; recommendations are proposals, not settled decisions.

## Contributing

Conventions for structure, front matter, and file naming live in [`CLAUDE.md`](./CLAUDE.md). Adding or renaming a document means updating the index above in the same change.

## License

[Apache License 2.0](./LICENSE).
