# Promptfoo Persistence Layer Deep Dive

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Why SQLite?](#why-sqlite)
   - [The Local-First Decision](#the-local-first-decision)
3. [Architecture Overview](#architecture-overview)
   - [Why This Layering?](#why-this-layering)
4. [The Database Connection: Singleton Pattern](#the-database-connection-singleton-pattern)
   - [What is the Singleton Pattern?](#what-is-the-singleton-pattern)
   - [Write-Ahead Logging (WAL Mode)](#write-ahead-logging-wal-mode)
   - [Lazy Initialization](#lazy-initialization)
5. [Schema Design: Tables and Relationships](#schema-design-tables-and-relationships)
   - [The Core Question: How to Store Evaluation Results?](#the-core-question-how-to-store-evaluation-results)
   - [Entity-Relationship Diagram](#entity-relationship-diagram)
   - [Junction Tables: Why So Many?](#junction-tables-why-so-many)
   - [The eval_results Table: Denormalized by Design](#the-eval_results-table-denormalized-by-design)
   - [JSON Columns: Flexibility vs. Query Speed](#json-columns-flexibility-vs-query-speed)
6. [Domain Models: Eval and EvalResult](#domain-models-eval-and-evalresult)
   - [What is a Domain Model?](#what-is-a-domain-model)
   - [The Eval Class: Active Record Pattern](#the-eval-class-active-record-pattern)
   - [Lazy Loading: Why Results Aren't Loaded by Default](#lazy-loading-why-results-arent-loaded-by-default)
   - [Versioning: V2, V3, V4](#versioning-v2-v3-v4)
7. [The Migration System](#the-migration-system)
   - [What Are Database Migrations?](#what-are-database-migrations)
   - [Automatic Migration on Startup](#automatic-migration-on-startup)
   - [The drizzle Directory](#the-drizzle-directory)
8. [Real-Time Updates: The Signal File](#real-time-updates-the-signal-file)
   - [The Problem: Knowing When New Results Arrive](#the-problem-knowing-when-new-results-arrive)
   - [Rejected Approaches](#rejected-approaches)
   - [The Solution: File-Based Signaling](#the-solution-file-based-signaling)
9. [Query Optimization: Making Millions of Results Fast](#query-optimization-making-millions-of-results-fast)
   - [The Challenge](#the-challenge)
   - [Strategy 1: In-Memory Caching](#strategy-1-in-memory-caching)
   - [Strategy 2: Composite Indexes](#strategy-2-composite-indexes)
   - [Strategy 3: Two-Phase Pagination](#strategy-3-two-phase-pagination)
   - [Strategy 4: Single Source of Truth for Filters](#strategy-4-single-source-of-truth-for-filters)
10. [Blob Storage: Handling Large Binary Data](#blob-storage-handling-large-binary-data)
    - [The Problem: Audio and Images in Responses](#the-problem-audio-and-images-in-responses)
    - [The Externalization Strategy](#the-externalization-strategy)
    - [Size Thresholds](#size-thresholds)
    - [Blob Reference Tracking](#blob-reference-tracking)
11. [Tracing: OpenTelemetry Integration](#tracing-opentelemetry-integration)
    - [What Are Traces?](#what-are-traces)
    - [Why Store Traces?](#why-store-traces)
    - [The TraceStore Class](#the-tracestore-class)
    - [Relationship to Evaluations](#relationship-to-evaluations)
12. [Version Evolution: Learning from History](#version-evolution-learning-from-history)
    - [The V2/V3 Problem](#the-v2v3-problem)
    - [The V4 Solution](#the-v4-solution)
    - [Backward Compatibility Strategy](#backward-compatibility-strategy)
13. [Data Flow: Write Path](#data-flow-write-path)
14. [Data Flow: Read Path](#data-flow-read-path)
15. [Key Design Decisions Summary](#key-design-decisions-summary)
16. [Environment Variables Reference](#environment-variables-reference)
17. [File Reference](#file-reference)

## Executive Summary

Promptfoo uses **SQLite** as its embedded database with **Drizzle ORM** for type-safe schema management. This document explains not just *what* the persistence layer does, but *why* it was designed this way and the trade-offs involved.

**Core Design Philosophy**: Local-first, zero-configuration operation. Users should be able to run `promptfoo eval` without setting up a database server, yet the system must scale to millions of test results while supporting real-time UI updates.

## Why SQLite?

### The Local-First Decision

Promptfoo chose SQLite over PostgreSQL or MongoDB for several compelling reasons:

1. **Zero Configuration**: Users install promptfoo and immediately start evaluating. No database server to install, no connection strings to configure, no ports to manage. The database is simply a file that appears in `~/.promptfoo/promptfoo.db`.

2. **Single-File Portability**: An entire evaluation history can be backed up by copying a single file. This makes it trivial to share evaluation results with teammates or move between machines.

3. **Embedded Performance**: SQLite runs in-process, eliminating network round-trips. For the read-heavy workloads of browsing evaluation results, this provides sub-millisecond query latency.

4. **Reliability**: SQLite is one of the most tested pieces of software in existence, used in every smartphone and browser. It handles crashes gracefully and never corrupts data.

**Trade-off Acknowledged**: SQLite doesn't support multiple concurrent writers across processes. Promptfoo mitigates this through WAL mode (explained below) and by designing the write path to be single-threaded within each evaluation run.

[📂 Database Entry Point](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/database/index.ts)

## Architecture Overview

The persistence layer is organized into four distinct layers, each with a specific responsibility:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Application Layer                                 │
│                                                                             │
│   Domain models (Eval, EvalResult) that represent business concepts.        │
│   These classes know nothing about SQL—they work with objects.              │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Drizzle ORM Layer                                │
│                                                                             │
│   Translates between objects and SQL. Provides type safety so the           │
│   compiler catches errors like querying a non-existent column.              │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           better-sqlite3                                    │
│                                                                             │
│   Native SQLite bindings for Node.js. Runs SQLite directly in the           │
│   same process, no network calls needed.                                    │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ~/.promptfoo/promptfoo.db                           │
│                                                                             │
│   The actual database file on disk. Also creates .db-wal and .db-shm        │
│   files for Write-Ahead Logging mode.                                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why This Layering?

**Separation of Concerns**: The `Eval` class doesn't contain SQL strings. It calls methods like `findById()` that return domain objects. This means:
- Business logic stays clean and testable
- The database implementation could theoretically be swapped (though SQLite is deeply embedded)
- Developers working on the UI don't need to understand SQL

## The Database Connection: Singleton Pattern

[📂 src/database/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/database/index.ts)

### What is the Singleton Pattern?

A "singleton" means there's only ever one database connection for the entire application. The first time any code needs the database, a connection is created. Every subsequent request reuses that same connection.

**Why a Singleton?**

1. **Resource Efficiency**: Each database connection consumes memory and file handles. Creating a new connection for every query would be wasteful and slow.

2. **Consistency**: With one connection, all queries see a consistent view of the data. Multiple connections could theoretically see slightly different states during concurrent writes.

3. **Connection Pooling Not Needed**: Unlike PostgreSQL where you'd use a connection pool, SQLite with a single connection is actually the recommended pattern for embedded use.

### Write-Ahead Logging (WAL Mode)

When the database opens, promptfoo enables "WAL mode." This is a crucial optimization that deserves explanation.

**Traditional SQLite Behavior** (without WAL):
- When you write data, SQLite must lock the entire database file
- While writing, no other process can even read the database
- This means the UI would freeze during a long evaluation run

**With WAL Mode**:
- Writes go to a separate "write-ahead log" file (.db-wal)
- Readers can continue accessing the main database file simultaneously
- Periodically, the log is merged back into the main file ("checkpointing")

**Practical Impact**: During a 10,000-test evaluation that takes 30 minutes, the web UI remains responsive. Users can browse previous results, filter data, and export reports while new results stream in.

**Trade-off**: WAL mode creates additional files (.db-wal, .db-shm) alongside the main database. Some network filesystems (NFS, cloud drives) don't support WAL mode properly. Promptfoo detects this and falls back to traditional mode, logging a warning.

### Lazy Initialization

The database connection isn't created when promptfoo starts—it's created on first use. This is called "lazy initialization."

**Why Lazy?**

1. **Faster Startup**: If you run `promptfoo help`, there's no need to open a database. Lazy loading means the help command runs instantly.

2. **Testing Flexibility**: Test suites can configure a different database (like an in-memory database) before the connection is created.

3. **Error Handling**: If the database file is corrupted, the error occurs at a predictable point (first query) rather than during process startup.

## Schema Design: Tables and Relationships

[📂 src/database/tables.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/database/tables.ts)

### The Core Question: How to Store Evaluation Results?

An evaluation consists of:
- A **configuration** (which prompts, which LLM providers, which test cases)
- Many **results** (one for each combination of test case × prompt × provider)
- Associated **metadata** (tags, descriptions, timing information)

The schema must balance several competing concerns:
1. **Query Speed**: Finding all failures for a specific evaluation should be fast
2. **Storage Efficiency**: Don't duplicate data unnecessarily
3. **Flexibility**: Support arbitrary metadata for red-teaming, custom assertions, etc.
4. **Historical Compatibility**: Don't break old evaluation data when the schema evolves

### Entity-Relationship Diagram

```
                              ┌─────────────┐
                              │   prompts   │
                              │             │
                              │ - id        │
                              │ - prompt    │ (the actual prompt text)
                              └──────▲──────┘
                                     │
                                     │ many-to-many
                                     │ (an eval uses many prompts;
                                     │  a prompt can be reused across evals)
                                     │
┌─────────────┐     ┌────────────────┴─────────────────┐     ┌─────────────┐
│    tags     │◄────┤              evals               │────►│  datasets   │
│             │     │                                  │     │             │
│ - name      │     │ - id (e.g., eval-abc-2025-01-13) │     │ - id        │
│ - value     │     │ - config (full evaluation setup) │     │ - tests     │
└─────────────┘     │ - prompts (with metrics)         │     └─────────────┘
                    │ - createdAt, author, description │
                    └──────────────────┬───────────────┘
                                       │
                                       │ one-to-many
                                       │ (one eval has thousands of results)
                                       │
                              ┌────────▼────────┐
                              │  eval_results   │
                              │                 │
                              │ - testIdx       │ (which row in the table)
                              │ - promptIdx     │ (which column)
                              │ - response      │ (LLM output)
                              │ - success       │ (pass/fail)
                              │ - score         │ (0.0 to 1.0)
                              │ - gradingResult │ (detailed assertion results)
                              │ - metadata      │ (custom fields)
                              └─────────────────┘
```

### Junction Tables: Why So Many?

You'll notice tables like `evals_to_prompts`, `evals_to_tags`, and `evals_to_datasets`. These are "junction tables" that implement many-to-many relationships.

**Example**: The same prompt text might be used in hundreds of evaluations. Rather than duplicating the (potentially large) prompt text in each evaluation, we store it once in `prompts` and link to it via `evals_to_prompts`.

**Benefits**:
- Searching "all evals using this prompt" becomes a simple join
- Prompt deduplication saves significant storage
- Changing a prompt's metadata doesn't require updating every evaluation

### The eval_results Table: Denormalized by Design

The `eval_results` table is intentionally denormalized—it stores redundant data for query performance.

**What's Denormalized?**

Each result row contains a full copy of:
- The test case (including all variables)
- The prompt that was used
- The provider configuration

**Why Duplicate This Data?**

1. **Query Independence**: To display a result, we only need to read `eval_results`. No joins required. For a table with millions of rows, avoiding joins is crucial.

2. **Historical Accuracy**: If someone edits a prompt after an evaluation, the stored result still shows what was actually tested.

3. **Filtering Without Joins**: The UI allows filtering by test case variables. If variables were in a separate table, every filter would require a join.

**Trade-off**: Storage size is larger. A 100,000-result evaluation might store the same prompt text 100,000 times. In practice, SQLite's text compression and the relatively small size of prompts make this acceptable.

### JSON Columns: Flexibility vs. Query Speed

Several columns store JSON data: `config`, `gradingResult`, `namedScores`, `metadata`.

**Why JSON?**

1. **Schema Flexibility**: Different assertion types return different grading structures. A `llm-rubric` assertion stores a detailed rubric breakdown; a simple `contains` assertion just stores a boolean. JSON accommodates both without schema changes.

2. **Arbitrary Metadata**: Red-team plugins add fields like `pluginId`, `strategyId`, `severity`. Custom assertions might add entirely different fields. JSON allows extension without migration.

**How SQLite Handles JSON**:

SQLite has built-in JSON functions. The schema creates indexes on JSON paths:

```sql
-- Index on the pluginId field inside the metadata JSON
CREATE INDEX idx_metadata_plugin ON eval_results(
  json_extract(metadata, '$.pluginId')
);
```

This means queries like "find all results from the 'jailbreak' plugin" use the index rather than scanning every row.

**Trade-off**: JSON queries are slower than native column queries. For the most critical filters (success/failure, eval_id), promptfoo uses native columns. JSON is reserved for extensible, less-frequently-filtered data.

## Domain Models: Eval and EvalResult

[📂 src/models/eval.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/eval.ts) | [📂 src/models/evalResult.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/evalResult.ts)

### What is a Domain Model?

A domain model is a class that represents a business concept (like "an evaluation") rather than a database table. It provides methods that make sense in business terms: "add a result," "get statistics," "delete this evaluation."

**Why Use Domain Models?**

Consider the difference between:

```
Database thinking: "INSERT INTO eval_results (eval_id, test_idx, ...) VALUES (...)"
Domain thinking:   "evaluation.addResult(result)"
```

Domain models hide database complexity. The calling code doesn't know or care that:
- Results are stored in a separate table
- Binary data (audio, images) is extracted to blob storage
- A signal file is updated to trigger UI refresh

### The Eval Class: Active Record Pattern

The `Eval` class follows the "Active Record" pattern, where each instance knows how to save itself to the database.

**Key Responsibilities**:

1. **Factory Methods**: `Eval.create()`, `Eval.findById()`, `Eval.latest()`
   - These create or retrieve Eval instances
   - `create()` handles the complex transaction of inserting related records

2. **Instance State**: Each Eval holds its configuration, results, prompts
   - Results are loaded lazily (only when needed) to save memory

3. **Persistence Methods**: `save()`, `delete()`, `addResult()`
   - These modify the database to match the in-memory state

4. **Query Methods**: `getTablePage()`, `getResults()`, `getStats()`
   - These retrieve data for display in the UI

### Lazy Loading: Why Results Aren't Loaded by Default

When you call `Eval.findById()`, the results are NOT automatically loaded. Why?

**Memory Constraints**: An evaluation with 100,000 results, each with a full LLM response, could consume gigabytes of memory. Most use cases don't need all results in memory simultaneously.

**The Pattern**:
```
eval = Eval.findById("eval-abc-2025-01-13")  // Only loads metadata
// eval.results is empty at this point

await eval.loadResults()  // Now loads all 100,000 results
// eval.results is populated
```

**Pagination Alternative**: For the UI, `getTablePage()` fetches only the results needed for the current page—perhaps 50 at a time.

### Versioning: V2, V3, V4

The storage format has evolved over time. The `version()` method returns which format an evaluation uses:

**Version 2 (Legacy)**: All results stored as JSON in the `evals.results` column
- Problem: Slow for large evaluations (entire JSON must be parsed)
- Problem: Difficult to query individual results

**Version 3 (Transitional)**: Results stored with table structure in JSON
- Slight improvement but still had scaling issues

**Version 4 (Current)**: Results stored in `eval_results` table, normalized
- Individual results can be queried independently
- Pagination works at the database level
- Filters use indexes for fast retrieval

**Backward Compatibility**: Promptfoo can still read V2 and V3 evaluations. The `useOldResults()` method checks the version and routes queries appropriately. Old evaluations are gradually migrated when accessed.

## The Migration System

[📂 src/migrate.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/migrate.ts)

### What Are Database Migrations?

When the schema changes (new columns, new tables, new indexes), existing databases need to be updated. Migrations are SQL scripts that transform the database from one version to the next.

**Example Migration** (simplified):
```sql
-- 0015_add_is_redteam_column.sql
ALTER TABLE evals ADD COLUMN is_redteam BOOLEAN DEFAULT FALSE;
CREATE INDEX idx_evals_is_redteam ON evals(is_redteam);
```

### Automatic Migration on Startup

Every time promptfoo starts, it runs `runDbMigrations()`. This:
1. Checks which migrations have already been applied (stored in a `_journal` table)
2. Runs any new migrations in order
3. Records successful migrations so they won't run again

**Why Automatic?**

Users shouldn't need to run manual database commands. When they upgrade promptfoo, schema changes "just work."

**Trade-off**: This adds ~100ms to startup time. The alternative (manual migration commands) was rejected as too error-prone for a CLI tool.

### The drizzle Directory

```
drizzle/
├── 0000_lush_hellion.sql      # Initial schema
├── 0001_wide_calypso.sql      # Added indexes
├── ...
├── 0022_sleepy_ultimo.sql     # Latest changes
└── meta/
    ├── _journal.json          # Which migrations exist
    └── 0022_snapshot.json     # Current schema state
```

The whimsical names (lush_hellion, wide_calypso) are auto-generated by Drizzle Kit. They're memorable but arbitrary—the numbers provide the ordering.

## Real-Time Updates: The Signal File

[📂 src/database/signal.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/database/signal.ts)

### The Problem: Knowing When New Results Arrive

During an evaluation, results are written to the database one at a time. The web UI needs to show these results as they arrive. How does the UI know when to refresh?

### Rejected Approaches

**Polling**: The UI could query the database every second. 
- Problem: Wasteful, especially when no evaluation is running
- Problem: Adds latency (up to 1 second delay)

**WebSocket from Evaluator**: The CLI could push updates to the UI.
- Problem: Requires the UI to be running before the CLI starts
- Problem: Complex connection management

**Database Triggers**: SQLite could notify on INSERT.
- Problem: SQLite triggers run synchronously, slowing writes
- Problem: Cross-process notification is complex

### The Solution: File-Based Signaling

Promptfoo uses a simple text file at `~/.promptfoo/evalLastWritten`:

**Write Side** (in the evaluator):
```
When a new result is added:
1. Insert result into database
2. Write current timestamp + eval ID to signal file
```

**Read Side** (in the web UI):
```
On startup:
1. Set up a file watcher on the signal file
2. When the file changes:
   a. Read the eval ID from the file
   b. If it matches the currently-displayed eval, refresh the table
```

**Why This Works Well**:

1. **Decoupled**: The evaluator doesn't need to know if any UI is watching
2. **Efficient**: File watchers are built into operating systems, very low overhead
3. **Debounced**: Changes are batched (250ms) to avoid overwhelming the UI during fast writes
4. **Cross-Process**: Works whether the UI is in the same process (dev mode) or a separate browser tab

**The File Format**:
```
eval-abc-2025-01-13T10:30:45:2025-01-13T10:31:22.456Z
│                           │
└─ Eval ID                  └─ Timestamp
```

The UI parses this to determine which evaluation was updated and when.

## Query Optimization: Making Millions of Results Fast

[📂 src/models/evalPerformance.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/evalPerformance.ts)

### The Challenge

Large evaluations can have millions of results. Users expect:
- Instant page loads (under 100ms)
- Responsive filtering
- Accurate aggregate statistics

Naive queries would be far too slow.

### Strategy 1: In-Memory Caching

Certain queries are expensive but their results change slowly. Promptfoo caches these in memory:

**Count Caching**:
```
Query: "How many results does this evaluation have?"
Without cache: ~200ms (full table scan on first query)
With cache: <1ms (memory lookup)
```

The cache has a 5-minute TTL (time-to-live). After 5 minutes, the next query refreshes the cache. This works because result counts only change during active evaluation runs, which are relatively short.

**Trade-off**: During an evaluation, the cached count might be slightly stale. The UI handles this by showing the cached count while streaming in new results.

### Strategy 2: Composite Indexes

Indexes are database structures that speed up lookups. A "composite index" includes multiple columns.

**Example Query**:
```sql
SELECT * FROM eval_results 
WHERE eval_id = 'eval-abc' AND success = 0 
ORDER BY test_idx 
LIMIT 50
```

**Without a Composite Index**:
1. Find all rows for eval-abc (might be 1 million rows)
2. Filter to only failures (scan all 1 million)
3. Sort and return first 50

**With Index on (eval_id, success, test_idx)**:
1. Jump directly to the index position for eval-abc + failure
2. Read 50 rows sequentially
3. Done

The schema includes carefully designed composite indexes:
```
eval_result_eval_test_idx       → (eval_id, test_idx)
eval_result_eval_success_idx    → (eval_id, success)
eval_result_eval_test_success   → (eval_id, test_idx, success)
```

### Strategy 3: Two-Phase Pagination

A naive approach to pagination:
```sql
SELECT * FROM eval_results 
WHERE eval_id = 'eval-abc' 
ORDER BY test_idx 
LIMIT 50 OFFSET 10000
```

**Problem**: The database must still process 10,000 rows before skipping them.

**The Two-Phase Approach**:

**Phase 1**: Get just the test indices (lightweight)
```sql
SELECT DISTINCT test_idx FROM eval_results 
WHERE eval_id = 'eval-abc' 
ORDER BY test_idx 
LIMIT 50 OFFSET 200
```
This returns 50 integers—very fast.

**Phase 2**: Fetch full results for those specific indices
```sql
SELECT * FROM eval_results 
WHERE eval_id = 'eval-abc' AND test_idx IN (201, 202, 203, ...)
```
This fetches exactly the rows needed, using the index for fast lookup.

**Why Two Phases?**

Each test case might have multiple results (if testing multiple prompts/providers). A single test_idx might correspond to 5 rows. By first identifying which test indices to display, we avoid fetching extra data only to discard it.

### Strategy 4: Single Source of Truth for Filters

The UI supports complex filtering: "show failures from the jailbreak plugin with severity HIGH."

A subtle bug could occur if:
- The pagination query uses one WHERE clause
- The statistics query uses a slightly different WHERE clause

Suddenly the UI shows "50 failures" but the statistics say "47 failures."

**The Solution**: `buildFilterWhereSql()` is the single function that constructs all filter conditions. Both pagination and statistics call this same function, guaranteeing consistency.

## Blob Storage: Handling Large Binary Data

[📂 src/blobs/extractor.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/blobs/extractor.ts)

### The Problem: Audio and Images in Responses

Modern LLMs can generate audio (text-to-speech) and images. These responses can be megabytes in size. Storing them directly in `eval_results` would:
- Bloat the database file enormously
- Slow down queries (even simple metadata queries would load megabytes of binary data)
- Make database backups impractical

### The Externalization Strategy

When a result contains binary data:

1. **Detection**: Check if the response contains data URLs (like `data:audio/wav;base64,...`)

2. **Extraction**: Decode the base64, compute a SHA-256 hash

3. **Storage**: Save to a separate blob storage system
   - The blob is stored once, even if used in multiple results
   - Blobs are content-addressed (named by their hash)

4. **Reference**: Replace the inline data with a reference
   ```
   Before: "data:audio/wav;base64,UklGRnoGAAAW..."  (megabytes of data)
   After:  "promptfoo://blob/a1b2c3d4..."           (64-character hash)
   ```

5. **On Retrieval**: When the UI needs the audio, it fetches from blob storage using the hash

### Size Thresholds

Not everything gets externalized:

- **Minimum 1KB**: Tiny images (like emoji) aren't worth the overhead of separate storage
- **Maximum 100MB**: Extremely large files might indicate an error or attack

Data outside these bounds stays inline in the database.

### Blob Reference Tracking

The `blob_references` table tracks which evaluations use which blobs:

```
blob_hash: a1b2c3d4...
eval_id: eval-abc-2025-01-13
test_idx: 42
location: "response.audio.data"
kind: "audio"
```

This enables:
- **Garbage Collection**: When an evaluation is deleted, unused blobs can be cleaned up
- **Deduplication**: If two evaluations produce identical audio, only one copy is stored
- **Auditing**: Understanding which evaluations generated which media

## Tracing: OpenTelemetry Integration

[📂 src/tracing/store.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/tracing/store.ts)

### What Are Traces?

When an LLM call fails or behaves unexpectedly, understanding what happened is crucial. Traces provide a detailed timeline:

```
Trace: evaluation-of-test-42
├── Span: renderPrompt (2ms)
│   └── Variable substitution, template rendering
├── Span: callProvider (1,234ms)
│   ├── Span: httpRequest to OpenAI (1,200ms)
│   │   └── Attributes: model=gpt-4, tokens=1500
│   └── Span: parseResponse (34ms)
└── Span: runAssertions (45ms)
    ├── Span: assertion-contains (2ms)
    └── Span: assertion-llm-rubric (43ms)
        └── Attributes: score=0.7, reason="..."
```

### Why Store Traces?

1. **Debugging**: When a test fails, traces show exactly where and why
2. **Performance Analysis**: Identify slow providers or prompts
3. **Cost Tracking**: Token counts and timing are captured in span attributes

### The TraceStore Class

The `TraceStore` follows the same patterns as other domain models:

**Creating a Trace**:
- Linked to an evaluation and test case
- Assigned a unique trace ID for correlation

**Adding Spans**:
- Each span has a parent (except the root)
- Start/end times enable duration calculation
- Attributes store arbitrary key-value data

**Sensitive Data Handling**:
- Attributes containing "password," "token," "api_key," etc. are automatically redacted
- Long strings are truncated to 400 characters

### Relationship to Evaluations

Each evaluation can have thousands of traces (one per test case). The `traces.evaluationId` foreign key enables efficient queries like "show all traces for this evaluation."

When an evaluation is deleted, its traces are not automatically deleted (no CASCADE). This is intentional—trace data might be valuable for aggregate analysis even after the specific evaluation is forgotten.

## Version Evolution: Learning from History

### The V2/V3 Problem

Early versions of promptfoo stored all results as a single JSON blob in the `evals.results` column. This seemed simple but caused problems:

1. **Loading Time**: Opening an evaluation with 10,000 results required parsing a 50MB JSON string
2. **Memory Pressure**: The entire result set had to be in memory simultaneously
3. **No Partial Queries**: "Show me just the failures" still loaded all results
4. **No Pagination**: The database couldn't help with "give me results 100-150"

### The V4 Solution

Version 4 normalizes results into a separate table. Each result is an independent row.

**Impact on Queries**:
```
V2: Load 50MB JSON, parse, filter in application code
V4: SELECT * FROM eval_results WHERE success = 0 LIMIT 50

V2: 10 seconds for a page of failures
V4: 50 milliseconds for a page of failures
```

### Backward Compatibility Strategy

Promptfoo supports reading all versions. The `version()` method detects the format:

```
If evals.results contains a 'table' key → Version 3
If evals.results is empty or just metadata → Version 4
```

When displaying old evaluations, the appropriate code path is used. New evaluations always use V4.

**No Automatic Migration**: Old evaluations are NOT automatically converted to V4. Why?
- Conversion would take significant time for large evaluations
- Users might want to delete old data rather than migrate it
- The old format still works, just slower

## Data Flow: Write Path

Understanding how data flows during an evaluation run:

```
1. User runs: promptfoo eval

2. Evaluator loads configuration
   └── Resolves prompts, providers, test cases

3. Eval.create() is called
   └── Transaction begins
       ├── INSERT INTO evals (config, prompts, ...)
       ├── INSERT INTO prompts (for each unique prompt)
       ├── INSERT INTO evals_to_prompts (linking)
       ├── INSERT INTO datasets (the test cases)
       ├── INSERT INTO evals_to_datasets (linking)
       └── INSERT INTO tags (if configured)
   └── Transaction commits atomically

4. For each (test_case, prompt, provider) combination:
   a. Render the prompt with test variables
   b. Call the LLM provider
   c. Run assertions on the response
   d. EvalResult.createFromEvaluateResult()
      ├── Extract binary data to blob storage if needed
      └── INSERT INTO eval_results
   e. updateSignalFile() to notify UI

5. After all results:
   └── Eval.save() updates aggregate statistics
```

**Atomicity Guarantee**: Step 3 uses a database transaction. If anything fails (disk full, invalid data), the entire evaluation record is rolled back. There's never a "partial evaluation" in the database.

## Data Flow: Read Path

When the UI requests evaluation results:

```
1. Browser requests: GET /api/evals/eval-abc-2025-01-13/results?page=5

2. Server calls Eval.findById()
   └── SELECT * FROM evals WHERE id = ?
   └── Returns Eval instance with metadata (NOT results)

3. Server calls eval.getTablePage({offset: 200, limit: 50})
   a. getCachedResultsCount() → total count (from cache or query)
   b. queryTestIndices() → which test indices to display
      └── Uses buildFilterWhereSql() for consistent filtering
      └── SELECT DISTINCT test_idx ... LIMIT 50 OFFSET 200
   c. EvalResult.findManyByEvalIdAndTestIndices()
      └── SELECT * FROM eval_results WHERE eval_id = ? AND test_idx IN (...)
   d. Group results by test_idx
   e. Convert to table rows

4. Response sent to browser
   └── {head: {prompts, vars}, body: [...], totalCount, filteredCount}
```

**Key Insight**: The actual LLM responses are only fetched in step 3c, and only for the specific test indices needed. This is why viewing page 5 of 10,000 results is fast—we never load the other 9,950 results.

## Key Design Decisions Summary

| Decision | Rationale | Trade-off |
|----------|-----------|-----------|
| SQLite over PostgreSQL | Zero-config, portable, embedded | No concurrent writers across processes |
| WAL Mode | Concurrent read/write | Extra files, incompatible with some filesystems |
| Singleton connection | Resource efficiency, consistency | Single point of failure |
| Denormalized results | Query speed, historical accuracy | Larger storage |
| JSON columns | Schema flexibility | Slower queries on JSON fields |
| Lazy result loading | Memory efficiency | Extra query for results |
| File-based signaling | Simple, cross-process | Relies on filesystem watchers |
| V4 normalized storage | Scalability, pagination | Backward compatibility complexity |
| Blob externalization | Database size, query speed | Additional storage system |

## Environment Variables Reference

These environment variables customize persistence behavior:

| Variable | Default | Purpose |
|----------|---------|---------|
| `PROMPTFOO_DISABLE_WAL_MODE` | false | Use traditional journaling (for network filesystems) |
| `PROMPTFOO_ENABLE_DATABASE_LOGS` | false | Log all SQL queries for debugging |
| `PROMPTFOO_STRIP_PROMPT_TEXT` | false | Redact prompts in exports (privacy) |
| `PROMPTFOO_STRIP_RESPONSE_OUTPUT` | false | Redact LLM responses in exports |
| `IS_TESTING` | false | Use in-memory database for tests |

## File Reference

| File | Purpose |
|------|---------|
| [src/database/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/database/index.ts) | Connection management, WAL configuration |
| [src/database/tables.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/database/tables.ts) | Schema definitions (all tables) |
| [src/database/signal.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/database/signal.ts) | Real-time update signaling |
| [src/models/eval.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/eval.ts) | Eval domain model |
| [src/models/evalResult.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/evalResult.ts) | Result domain model |
| [src/models/evalPerformance.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/evalPerformance.ts) | Query optimization & caching |
| [src/migrate.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/migrate.ts) | Migration runner |
| [src/blobs/extractor.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/blobs/extractor.ts) | Binary data externalization |
| [src/tracing/store.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/tracing/store.ts) | Trace storage |
| [drizzle.config.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/drizzle.config.ts) | Drizzle ORM configuration |
| drizzle/*.sql | Individual migration scripts |
