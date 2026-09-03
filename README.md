# IngestXcel

**A metadata-driven data ingestion framework for Microsoft Fabric.**

IngestXcel ingests data from multiple heterogeneous source systems into a governed Bronze/Silver medallion architecture — without writing new pipeline code for every source. Onboarding a new source is a metadata registration exercise (rows in a SQL database), not a development task. A single, generic set of pipelines and notebooks reads that metadata at runtime to determine how to connect, extract, transform, and load each entity.

> **A note on process, for transparency.** This project was designed, developed, and tested through an extensive collaborative process using Claude (Anthropic) as an AI development partner — for architecture discussion, code generation, and debugging guidance throughout. Every design decision, every test, and every verification of actual results (row counts, pipeline outputs, query results) was carried out and confirmed directly by the project owner against the real, live Fabric environment — Claude did not have direct access to the Fabric workspace itself. This included finding and fixing real bugs together (some introduced by Claude's own suggestions, caught through the project's own "verify before trusting" testing discipline), and several genuine architectural corrections made along the way as real Fabric behavior was discovered to differ from initial assumptions.

---

## Table of Contents

- [Why This Exists](#why-this-exists)
- [Architecture](#architecture)
- [Supported Sources](#supported-sources)
- [How It Works](#how-it-works)
- [Repository Structure](#repository-structure)
- [Onboarding a New Source](#onboarding-a-new-source)
- [What's Been Proven](#whats-been-proven)
- [Known Limitations & Roadmap](#known-limitations--roadmap)
- [Documentation](#documentation)

---

## Why This Exists

Conventional data ingestion typically means one hand-coded pipeline per source. Incremental-load logic, watermark tracking, and Slowly Changing Dimension (SCD) handling get re-implemented — often inconsistently — for every new table and every new system. IngestXcel inverts this: source connections, execution methods, and transformation rules are defined entirely as **metadata**, and one generic engine per layer (Bronze, Silver) reads that metadata to decide what to do — with zero source-specific code.

## Architecture

```
Multiple Source Systems (SQL Server, Azure SQL, Excel, CSV, ...)
                    |
                    v
          PL_Master_Orchestrator
   (metadata-driven, sequences layers per source system)
                    |
                    v
            PL_Bronze_Ingestion  ---->  Bronze Layer (Delta)
        (raw ingestion, all sources)   [ Bronze_LH ]
                    |
                    v
            PL_Silver_Ingestion  ---->  Silver Layer (Delta)
        (SCD1 / SCD2 merge, incremental)  [ Silver_LH ]
                    |
                    v
              Gold Layer (planned — not yet implemented)
```

A **metadata database** (Fabric SQL Database) sits behind every layer, holding connection details, entity registrations, and per-entity configuration. See [DB_SCHEMA.md](DB_SCHEMA.md) for the full schema.

## Supported Sources

| Source Type | Bronze Execution Method(s) | Status |
|---|---|---|
| **Fabric SQL Database** | PIPELINE (Copy activity), NOTEBOOK (staged Parquet) | ✅ Proven — both methods, backfill + incremental |
| **Azure SQL Database** | PIPELINE, NOTEBOOK | ⚠️ Partial — PIPELINE entities need re-verification after a metadata reset; NOTEBOOK entity (`DimDate`) needs rebuilding |
| **Lakehouse Files (CSV, Excel)** | NOTEBOOK (one shared, format-agnostic notebook) | ✅ Proven — both formats, including multi-file union and a real quarantine scenario |
| SQL Server, Oracle | — | 📋 Planned — paused pending a live test instance |
| REST API | — | 📋 Planned — split design (simple built-in connector vs. custom notebook for complex auth) |

Regardless of source, **every entity flows through the exact same Silver engine** (`NB_Silver_SCD_Load`) — this is the framework's core, empirically-validated claim, not just a design intention.

## How It Works

1. **A source is onboarded** by registering it in the metadata database: a connection endpoint, a source entity, an orchestration row (per layer), and configuration (execution method, watermark column, SCD type, etc.).
2. **`PL_Master_Orchestrator`** is triggered (per source system, per frequency) with a `SystemIdentifier` and `Frequency`, derives the correct trigger names, and invokes Bronze then Silver in strict sequence.
3. **`PL_Bronze_Ingestion`** reads the metadata batch for its trigger, and for each entity, routes — via a `Switch` keyed on `EXECUTION_METHOD` + source tech type — to one of several paths: a plain Copy activity, a staged-Parquet-then-notebook path (for types a Copy activity can't handle directly), or a file-ingestion notebook.
4. **`PL_Silver_Ingestion`** reads Bronze's own output (never the original source) and applies SCD1 (simple upsert) or SCD2 (historized, with close-out/new-version logic) per entity's metadata — including a no-watermark, hash-comparison fallback for entities with no reliable timestamp column.
5. **Every run is logged** (`META_INGESTION_LOG`) with row counts and watermark state, and a failed run can be retried scoped to just the failed entities (`TargetEntityIds`), without reprocessing everything else.

## Repository Structure

| Path | Contents |
|---|---|
| `CLAUDE.md` | Project overview, non-negotiable principles, naming conventions |
| `docs/decisions/000N-*.md` | Architecture Decision Records — one per major design decision, never edited in place (superseded by a new ADR instead) |
| `LESSONS_LEARNED.md` | Every hard-won bug and its fix, searchable |
| `PLATFORM_CONSTRAINTS.md` | Confirmed Fabric platform facts and limits |
| `DB_SCHEMA.md` | Metadata database DDL |
| `FEATURE_MATRIX.md` | Capability-by-capability status across 16 categories |

**Key pipelines**: `PL_Master_Orchestrator`, `PL_Bronze_Ingestion`, `PL_Silver_Ingestion`
**Key notebooks**: `NB_Bronze_Staged_Ingestion` (SQL sources, NOTEBOOK method), `NB_Bronze_File_Ingestion` (CSV/Excel), `NB_Silver_SCD_Load` (all Silver entities, every source type)
**Key stored procedures**: `spGet_Bronze_Batch`, `spGet_Silver_Batch`, `spStart_Ingestion_Log`, `spComplete_Ingestion_Log`, `spValidate_Bronze_Metadata`, `spValidate_Silver_Metadata`, `spGet_Failed_Entities`

## Onboarding a New Source

Per the "DDL-first" governance decision ([ADR-0002](docs/decisions/0002-onboarding-governance-table-provisioning.md)), the framework never auto-creates target tables. Onboarding a new entity means:

1. Create the Bronze table via explicit DDL (nullable columns — Bronze lands data raw, no validation)
2. Create the Silver table via explicit DDL (add SCD2 tracking columns if applicable)
3. Create the quarantine table
4. Register the entity across `META_SOURCE_ENTITY`, `META_ORCHESTRATION`, and `META_CONFIGURATION_CORE` (Bronze row, Silver row — always separate)
5. Run `PL_Bronze_Ingestion` and `PL_Silver_Ingestion` scoped to the new entity's `TargetEntityIds`, verify independently

No pipeline or notebook code changes are required for a new entity of an already-supported source type.

## What's Been Proven

- **Full clean-rebuild test**: metadata wiped to zero, re-registered, re-run from scratch across multiple entities — backfill and incremental correctness both confirmed, independently verified via direct row counts (not just trusting pipeline-reported numbers)
- **Every distinct mechanism the framework supports**, demonstrated correct simultaneously: PIPELINE Upsert-merge, NOTEBOOK append-with-Silver-dedup, SCD1 overwrite, SCD2 close-out/new-version, no-watermark hash-comparison
- **Cross-source-type consistency**: the same Silver notebook, unmodified, proven correct against Fabric SQL Database, Azure SQL Database, CSV, and Excel sources
- **Real incremental correctness**: a genuine new row inserted at a live source, correctly detected via watermark (not a full rescan) and correctly propagated through both layers
- **Orchestration correctness**: Bronze-before-Silver sequencing enforced with zero manual intervention, across both backfill and incremental runs

## Known Limitations & Roadmap

- **Gold layer**: not implemented — reserved placeholder scope only ([ADR-0006](docs/decisions/0006-gold-placeholder-scope.md))
- **Azure SQL**: several entities need re-verification following a metadata reset that had no entity-scoping filter
- **Monitoring/alerting**: run failures are logged but nothing currently pushes a notification anywhere
- **Real Fabric Schedule Trigger**: not yet proven end-to-end — every run to date has been manually triggered
- **Silver reconciliation check**: a known non-blocking bug produces a false mismatch on SCD2 runs with real changed rows (the underlying merge itself is correct, independently verified)
- **SQL Server, Oracle, REST API sources**: planned, not yet built

Every one of these is deliberately documented here rather than silently omitted — see [LESSONS_LEARNED.md](LESSONS_LEARNED.md) and the ADR index for full detail on each.

## Documentation

Start with `CLAUDE.md` for the project overview and index. For *why* a specific design decision was made, check the relevant ADR in `docs/decisions/` — each one follows the same format (Context, Decision, Consequences) and is never rewritten after the fact; a changed decision gets a new, superseding ADR instead.
