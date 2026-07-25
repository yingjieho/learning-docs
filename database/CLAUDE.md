# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **documentation repository** — technical decision documents (comparisons, ADRs, evaluation matrices) authored in Markdown. There is no application code, build system, package manager, or test suite here. Work in this repo is writing and editing prose, not compiling or running software.

## Working conventions

Documents follow a consistent structure worth matching when creating new ones:

- **Front matter block** at the top using blockquote lines: `Status`, `Date`, `Author`, `Related` (links to companion docs), and `Scope` (explicitly stating what the doc does *and does not* cover).
- **Numbered top-level sections** (`## 1. Context`, `## 2. Options considered`, …) ending with a `## Sources` section listing every external reference as a Markdown link.
- **Decision documents** follow a repeatable pattern: Context → Options → Weighted evaluation criteria → Option-by-option analysis (Pros / Cons-risks) → Decision matrix with weighted scores → Recommendation → Open questions/next steps.
- **Hard requirements gate before scoring** — an option that fails a gating requirement is called out even if it tops the raw weighted score (see the ClickHouse Cloud row in `clickhouse-local-dev-comparison.md` for the pattern).

## Content context

The current document evaluates local-development options for **ClickHouse** (an OLAP data store under evaluation) on a **Windows 11 / Node.js** stack. It references a companion ADR ("Adopt ClickHouse as OLAP data store") that is not yet in this repo. When editing, keep claims sourced — every factual assertion (licensing thresholds, version/EOL dates, tooling support) should trace to a link in the Sources section, and dates should be absolute.
