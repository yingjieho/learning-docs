# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **documentation repository** — technical decision documents (comparisons, ADRs, evaluation matrices) authored in Markdown. There is no application code, build system, package manager, or test suite here. Work in this repo is writing and editing prose, not compiling or running software.

Documents are grouped into topic folders (`database/` today). A new topic gets a new folder rather than piling into an existing one.

## Working conventions

### Front matter

Every document opens with an `# H1` title, then a blockquote block:

```markdown
> **Status:** Draft for discussion
> **Date:** 2026-07-25
> **Author:** yingjie.ho@merquri.io
> **Related:** [`other-doc.md`](./other-doc.md) — what it covers and how it relates
> **Scope:** What the doc covers *and explicitly does not* cover.
```

- `Related` links companion docs by relative path, or `— (standalone …)` when there are none. Cross-references are reciprocal — if A points at B, B points back at A.
- `Scope` is a fence, not a summary: it names what is deliberately out of scope and where that material lives instead.
- Dates are absolute (`2026-07-25`), never relative.

### Structure

- **Numbered top-level sections** (`## 1. Context`, `## 2. Options considered`, …) ending with an unnumbered `## Sources` section listing every external reference as a Markdown link.
- **Decision documents** follow: Context → Options → Weighted evaluation criteria → Option-by-option analysis (Pros / Cons-risks) → Decision matrix with weighted scores → Recommendation → Open questions/next steps. See `database/clickhouse-local-dev-comparison.md` and `database/pipeline-local-dev-comparison.md`.
- **Head-to-head comparisons** use a shorter shape: Overview → one wide table where each row is a single dimension → Summary that tallies the verdict column and maps priorities to choices → Sources. See `database/03a-aurora-vs-rds-postgres-wal.md` and its companion `03b`.
- Comparison-table columns carry a fixed rhythm: option A | option B | Caveat | Impact | Mitigation | verdict. Caveat/Impact/Mitigation cells are `• `-bulleted with `<br>` separators and tag which option they apply to — `*(Aurora)*`, `*(RDS)*`, `*(Both)*`.
- **Hard requirements gate before scoring** — an option that fails a gating requirement is called out even if it tops the raw weighted score (see the ClickHouse Cloud row in `clickhouse-local-dev-comparison.md`).
- A recommendation may be a **portfolio rather than one winner** when the evidence supports it (see `pipeline-local-dev-comparison.md` §8), and trade-off rows are named as trade-offs instead of forced into a verdict.

### File naming

Lower-kebab-case. A numeric prefix orders docs within a folder; a letter suffix marks a split into companion docs on the same subject (`03a-…-wal.md`, `03b-…-performance.md`). Match the surrounding folder — don't add prefixes to files in a folder that doesn't use them.

## Content context

Current documents evaluate an **OLTP/OLAP split**: PostgreSQL (Aurora vs. RDS) as the OLTP source, CDC via PeerDB/Debezium, ClickHouse as the OLAP sink — with local-development ergonomics on a **Windows 11 / Node.js** stack as a recurring constraint. Several docs reference a companion ADR ("Adopt ClickHouse as OLAP data store") that is not yet in this repo.

When editing, keep claims sourced — every factual assertion (licensing thresholds, version/EOL dates, benchmark figures, tooling support) traces to a link in that document's Sources section. Vendor benchmark numbers are labelled as vendor claims and directional, not guarantees.

## Maintaining the README

`README.md` indexes every document with a one-line description. Adding or renaming a document means updating that index in the same change.
