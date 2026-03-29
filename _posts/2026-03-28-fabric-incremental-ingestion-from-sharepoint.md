---
layout: default
title: "Incremental Ingestion in Microsoft Fabric: SharePoint to Lakehouse Without the Mess"
date: 2026-03-28
---

[Home](/) · [Blog](/blog/)

---

## Incremental Ingestion in Microsoft Fabric: SharePoint to Lakehouse Without the Mess

Enterprise teams accumulate data in SharePoint. Shared Excel files, monthly expense reports, tracking sheets that someone started in 2019 and never migrated. At some point, a data engineer is asked to "get this into our analytics platform." The question is not whether to ingest it — it is whether to do it in a way that survives the first schema change, the first duplicate row, and the first time someone asks "what did this look like last month?"

This post walks through a code-first incremental ingestion pattern in Microsoft Fabric: pipeline-driven Copy from SharePoint, immutable snapshots in OneLake, Spark notebook parsing, and a watermark-controlled schedule that only touches what changed.

### Why code-first

Fabric offers low-code options — Dataflow Gen2 can connect to SharePoint folders and do light transformations visually. For a one-off load of a clean CSV, that works. For recurring ingestion of Excel files from SharePoint, it falls apart fast:

- **SharePoint folder connector** is not supported in Fabric pipelines. It exists in Dataflow Gen2 only, which limits orchestration control.
- **Excel files are not databases.** They have merged cells, hidden sheets, inconsistent headers, and formulas that resolve to different values depending on who opened the file last.
- **Deduplication requires logic.** When the same file gets modified and re-ingested, you need row-level hashing and snapshot isolation. That is not a drag-and-drop operation.

The code-first approach uses a Copy activity (which does support SharePoint Online File with wildcards and last-modified filtering) chained to a Spark notebook that handles parsing, hashing, and append-only writes.

### The architecture

```text
SharePoint (Excel files)
    │
    ▼
Copy Activity ── filter by last_modified ── landing/run_id=<guid>/
    │
    ▼
Spark Notebook ── snapshot + parse ── snapshots/date=YYYY-MM-DD/run_id=<guid>/
    │                                  raw_manifest (Delta, append-only)
    │                                  raw_rows (Delta, append-only)
    ▼
Curated View ── latest-wins dedup ── Warehouse / Power BI
    │
    ▼
Watermark Update ── only after success
```

Four stages, each with a clear responsibility. The pipeline orchestrates sequencing and failure handling. The notebook handles the messy parts.

### The watermark pattern

The core mechanism is a control table that tracks the high-water mark — the last successful ingestion timestamp. Every pipeline run reads this watermark, uses it to filter SharePoint files by `last_modified`, and only advances the watermark after all downstream steps succeed.

```sql
CREATE TABLE ctl_watermark (
    source_name   VARCHAR(100),
    watermark_utc DATETIME2,
    updated_by    VARCHAR(200)
);

INSERT INTO ctl_watermark VALUES (
    'sharepoint_te_expenses',
    '1900-01-01T00:00:00Z',
    'initial_seed'
);
```

The pipeline's first activity is a Lookup that reads this value. The Copy activity then uses it as a dynamic parameter:

```text
Source: SharePoint Online File (Preview)
  File path type:        Wildcard
  Wildcard folder path:  Shared Documents/TE/
  Wildcard file name:    *.xlsx
  Filter by last modified:
    Start time:  @activity('LookupWatermark').output.firstRow.watermark_utc
    End time:    @addminutes(pipeline().TriggerTime, -5)
```

The five-minute safety lag on the end time avoids race conditions with files that are still being written when the pipeline fires. This is a standard defensive pattern — not Fabric-specific, but easy to forget.

### Immutable snapshots

The Copy activity lands files into a timestamped folder:

```text
Files/landing/run_id=<pipeline_run_id>/*.xlsx
```

The Spark notebook's first job is to promote these into an immutable snapshot path:

```text
Files/snapshots/date=2026-03-29/run_id=abc123/original_filename.xlsx
```

Why not write directly to snapshots? Because the landing zone is ephemeral and the snapshot zone is append-only. If the notebook fails mid-parse, the snapshot zone is not corrupted. The landing zone can be safely re-processed or cleaned up.

This is the same principle behind write-ahead logs in databases: separate the "I received this" step from the "I processed this" step.

### Parsing Excel in Spark

This is where things get ugly. Excel files are not columnar data. They are visual documents that happen to have rows. The Spark notebook must handle:

- **Sheet selection.** The data might be on Sheet1 or on a named sheet like "Q1 Expenses."
- **Header detection.** The first row is not always the header. Sometimes row 1 is a title, row 2 is blank, and row 3 is the header.
- **Type coercion.** Excel stores dates as floating-point numbers. Currency as strings with dollar signs. Booleans as 0/1.
- **Row hashing.** Since Excel rows rarely have stable primary keys, you need a deterministic hash of the row content to detect duplicates across snapshots.

If the Spark runtime has the `com.crealytics.spark.excel` reader, you can read `.xlsx` directly. If not, the fallback is converting to CSV or Parquet during the Copy activity and reading that in Spark. The dedup and lineage logic stays the same either way.

The notebook appends every parsed row to a Delta table with lineage columns:

```text
raw_rows (Delta, append-only):
  | row_hash | col_A | col_B | ... | snapshot_path | run_id | ingest_ts |
```

Every row carries the snapshot it came from, the pipeline run that ingested it, and its content hash. Nothing is overwritten. This is the audit trail.

### The manifest

Alongside raw rows, the notebook writes a manifest table tracking every file that was processed:

```text
raw_manifest (Delta, append-only):
  | file_name | snapshot_path | file_size | row_count | run_id | ingest_ts |
```

This answers the question "what files did we ingest and when?" without scanning the snapshot directory. When Finance asks about a specific expense report from three months ago, you query the manifest, not the file system.

### Curated layer: latest-wins dedup

The raw layer has multiple versions of the same logical row (because the source file was modified and re-ingested). The curated layer resolves this with a latest-wins query:

```sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY row_hash
               ORDER BY ingest_ts DESC
           ) AS rn
    FROM raw_rows
)
SELECT * FROM ranked WHERE rn = 1;
```

This gives you the most recent version of each unique row. If your domain has a stable business key (like an ExpenseID), partition on that instead of `row_hash` for more precise dedup.

This view — or a materialized table built from it — feeds Power BI or the Fabric Warehouse for governed BI.

### Watermark update: the critical ordering constraint

The watermark update is the last activity in the pipeline, and it only runs if all preceding activities succeed. This is not optional — it is the correctness guarantee of the entire pattern.

```sql
UPDATE ctl_watermark
SET watermark_utc = '@{pipeline().TriggerTime}',
    updated_by    = '@{pipeline().Pipeline}/@{pipeline().RunId}'
WHERE source_name = 'sharepoint_te_expenses';
```

If the pipeline fails mid-run (Spark notebook crashes, Delta write fails), the watermark does not advance. The next run picks up the same files and reprocesses them. Because the raw layer is append-only and dedup happens at the curated layer, reprocessing is idempotent.

This is the standard incremental pattern across every data platform — Fabric, Airflow, Dagster, Step Functions. The implementation details change; the principle does not: never advance the checkpoint until the work is committed.

### What this gives you

- **Incremental by default.** Only new/changed files are copied each run.
- **Full audit trail.** Every file, every row, every pipeline run is tracked.
- **Immutable history.** Snapshots are append-only. You can reconstruct the state of any file at any point in time.
- **Idempotent reprocessing.** Failed runs can be retried without data corruption.
- **Clean separation of concerns.** Pipeline handles orchestration. Notebook handles parsing. SQL handles dedup. Each layer can be debugged independently.

The pattern is not unique to Fabric. The same architecture works in any lakehouse: S3 + Airflow + Spark, ADLS + ADF + Databricks, GCS + Composer + Dataproc. Fabric's advantage is that all the components — Copy, Spark, Delta, SQL, scheduling — live in one workspace with shared authentication and a single OneLake namespace. You do not have to wire five services together with IAM roles and network policies.

The hardest part, as always, is not the platform. It is the Excel files.
