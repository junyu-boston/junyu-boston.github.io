---
title: "Data Lake, Data Warehouse, and the Lakehouse: What Actually Changed"
date: 2026-04-21
categories: [Data Engineering]
tags: [aws, data-lake, data-warehouse, lakehouse, s3, athena, redshift, glue, apache-iceberg]
description: "Why data lakes, warehouses, and lakehouses solve different problems on AWS, and why the real boundary is workload shape rather than product labels."
---

*Athena over S3 proves data is queryable. It does not prove the workload has a serving contract. The real distinction between a lake, a warehouse, and a lakehouse is not the SQL surface. It is the operational behavior underneath it.*

The first time Athena returns results from a query over raw Parquet files in S3, the reaction is always the same: "Why would I ever copy this data into a warehouse?" The data is already there. The SQL works. There is no infrastructure to manage. The answer comes later — usually when a dashboard is slow, a finance report is wrong, or forty analysts hit the same table at 9 AM and the bill arrives.

S3 + Glue + Athena makes data queryable. A data warehouse makes data dependable. A lakehouse tries to do both. Understanding where each one breaks is the only way to choose correctly.

## The data lake: storage separated from compute

A data lake is not a product. It is an architecture pattern built from three layers: storage, metadata, and compute. On AWS, those layers map to S3, Glue Data Catalog, and Athena.

**S3** is the storage layer. It is cheap, durable (eleven nines), and schema-agnostic. It does not care whether you store CSV, JSON, Parquet, or raw binary. Files land in prefixes organized however you choose — by date, by source system, by team. S3 stores bytes. It has no opinion about what those bytes mean.

**Glue Data Catalog** is the metadata layer. It imposes structure on the unstructured. A crawler or manual registration creates database and table definitions that map S3 prefixes to schemas — column names, types, partition keys. The catalog is a schema registry for the lake. Without it, S3 is a file system. With it, S3 becomes something query engines can understand.

**Athena** is the compute layer. It reads table definitions from the Glue Data Catalog, plans the query, prunes partitions and files where possible, reads the relevant data from S3, and returns results. There is no cluster to provision, no capacity to reserve. You write SQL, Athena scans data, and you pay per terabyte scanned. Schema-on-read means the same underlying files can be queried through different table definitions without moving or transforming the data.

This architecture gives you several things quickly: low infrastructure overhead, ad-hoc exploration of almost any format, and a natural landing zone for raw data from dozens of source systems. It is extremely effective for the first six months of any data platform.

What it does not give you: performance guarantees, high concurrency, ACID transactions, or the ability to update a single row without rewriting an entire file. Those are not bugs. They are consequences of the separation between storage and compute. S3 is an object store, not a database.

## The data warehouse: managed performance over flexible storage

A data warehouse solves the problems a data lake cannot. Redshift — AWS's warehouse — takes a very different architectural stance: instead of querying raw files through schema-on-read, it behaves like a managed analytic database that organizes data ahead of time and optimizes for repeatable workloads.

When data enters Redshift, it can be sorted, compressed, and distributed according to the physical design of the table. The query planner maintains column-level statistics — min/max values, distinct counts, histograms. When a query arrives, the planner already has enough system knowledge to skip irrelevant blocks, choose a join strategy, and manage resources for the workload. None of this is accidental. It is the result of deliberate physical design.

This is what warehouses add:

- **More predictable performance.** Repeated business queries tend to behave consistently because the system is optimized around managed storage, statistics, and workload control rather than raw file scans.
- **Concurrency.** Workload management queues, resource isolation, and query prioritization let hundreds of users and dashboards hit the system simultaneously.
- **ACID transactions.** Updates, deletes, and upserts are native operations. Slowly changing dimensions, late-arriving facts, and corrections do not require rewriting entire datasets.
- **Materialized views.** Pre-computed aggregations, with automatic refresh available in some supported cases, eliminating redundant computation for repeated queries.

The cost model is inverted: you pay primarily for provisioned compute or metered warehouse compute, not per terabyte scanned from object storage. Repetition is usually cheaper operationally than in a pay-per-scan lake because the system is built for recurring workloads.

## Where they diverge

The distinction is not about data type or SQL dialect. It is about workload shape and operational guarantees.

| Dimension | Data lake (Athena) | Data warehouse (Redshift) |
| --- | --- | --- |
| **Performance** | Varies with file layout, partitioning, and data volume scanned | Predictable — pre-organized data, known statistics, optimized plans |
| **Concurrency** | Good for ad-hoc and light-to-moderate shared use; heavy BI concurrency gets expensive or inconsistent quickly | Designed for sustained dashboard and analyst concurrency with workload management |
| **Cost model** | Pay per TB scanned — repeated queries multiply cost linearly | Pay per compute — repeated queries are amortized or cached |
| **Data mutability** | Files are immutable; updates require full file rewrites | Native UPDATE, DELETE, MERGE, slowly changing dimensions |
| **Operational contract** | Capability: you *can* run the query | Contract: the query can be tuned, governed, and supported with stronger performance expectations |

The last row is the most important. Enterprises do not pay for SQL. They pay for predictability. When the CFO's dashboard refreshes every morning, the question is not "can this query run?" — it is "will this query return in under three seconds, every day, without breaking anything else?" A data lake provides capability. A warehouse provides a contract.

## The architecture that actually works

In practice, mature data platforms use both. The lake and the warehouse are not competitors — they are stages in a pipeline.

```text
Raw / semi-structured data
        │
        ▼
   S3 Data Lake
   (flexible, cheap, everything lands here)
        │
        ▼
   Athena / Spark
   (explore, validate, prototype)
        │
        ▼
   Curated, business-ready data
        │
        ▼
   Redshift / Data Warehouse
   (dashboards, KPIs, SLAs, BI tools)
```

Athena is where questions are born. Analysts explore new datasets, validate ingestion quality, and prototype transformations. Once a dataset matures — once it has consumers, SLAs, and a refresh schedule — it graduates into the warehouse. The lake is the workshop. The warehouse is the factory floor.

This two-tier architecture works, but it has a cost: the ETL pipeline between them. Data must be extracted from S3, transformed, and loaded into Redshift. That pipeline must be maintained, monitored, and debugged. The data now exists in two places. Schema changes in the lake must be propagated to the warehouse. Every copy is a liability.

## The lakehouse: dissolving the boundary

The lakehouse idea starts with a simple question: what if the storage layer could provide more database-like table behavior without copying data into a separate system?

**Open table formats** — Apache Iceberg, Delta Lake, and Apache Hudi — are the enabling technology. They sit between the query engine and the object store, adding a metadata layer that transforms a collection of files in S3 into something that behaves like a database table.

What open table formats add to S3:

- **ACID-style table semantics.** The format manages snapshots and metadata commits so compatible engines can provide transactional table behavior over object storage.
- **Row-level updates and deletes.** These changes are expressed through table metadata, delete files, and in some engines data-file rewrites. Compaction or optimization reconciles those changes over time.
- **Time travel.** Every transaction creates a new snapshot. You can query the table as it existed at any point in the past — essential for ML reproducibility and regulatory audits.
- **Schema evolution.** Many compatible engines can add or evolve columns without rewriting existing data, with the metadata layer handling much of the schema mapping.
- **Partition evolution.** Change the partitioning strategy without rewriting the table. Iceberg tracks which files belong to which partition spec version.

The storage is still S3. The files are often Parquet, though open table formats also support other columnar layouts. What changes is not the bytes but the identity of the table. In a plain lake, the "table" is mostly whatever files happen to live under a prefix — meaning the shared beginning of an S3 object key, which looks like a folder path but is really just a naming convention. In Iceberg, the table becomes a versioned metadata object: current snapshot, manifests, schema versions, partition specs, delete files, and file-level statistics. That metadata layer is what enables atomic commits, consistent reads, time travel, and smarter pruning. It makes S3 data behave like a managed table, even though it does not by itself provide the serving guarantees of a full warehouse. Athena v3 queries Iceberg tables natively. So do Spark, Trino, Flink, and Redshift Spectrum.

The architecture compresses from two tiers to one:

```text
Raw / semi-structured data
        │
        ▼
   S3 + Iceberg (lakehouse)
   (ACID, time travel, schema evolution, open format)
        │
        ├── Athena / Trino  (exploration, ad-hoc)
        ├── Spark           (heavy transforms, ML)
        └── Redshift Spectrum / BI tools  (dashboards, reporting)
```

One copy of the core table data. Multiple engines querying it. In many cases, no mandatory copy pipeline between lake and warehouse, even though serving copies and caches may still exist downstream. The format is open — which weakens vendor lock-in. Any engine with meaningful Iceberg support can participate, though feature completeness still varies by engine.

## What changes and what stays the same

The lakehouse can eliminate the most painful part of the two-tier architecture: the mandatory warehouse-load copy. Data lands once in S3, gets written as Iceberg tables with transactional table semantics, and is immediately queryable by compatible engines. Schema changes propagate through the metadata layer, not through a separate pipeline. In the best case, the core analytical table becomes the shared source of truth even if downstream serving copies still exist.

Vendor lock-in weakens. Iceberg is an open specification governed by the Apache Software Foundation. A table written by Spark can often be read by Athena, Trino, Flink, Snowflake, and other engines that implement Iceberg support, though the real-world experience still depends on catalog choices, spec support, and feature completeness. The query engine becomes a more pluggable component rather than a permanent platform commitment. This is a fundamental shift from the warehouse era, where data formats were often proprietary and migration meant re-ingesting everything.

But some things do not change.

**High-concurrency BI still needs a serving layer.** Query engines reading Iceberg tables on object storage are usually a poor fit for very high-concurrency, sub-second BI serving. For that workload, you still need a dedicated engine — Redshift Serverless, Snowflake, or a BI-optimized cache. The lakehouse reduces what must flow through that engine, but does not eliminate the engine itself.

**Governance is still required.** Lake Formation, IAM roles, and column-level access controls apply regardless of whether the table is a Hive-style partition layout or an Iceberg catalog entry. The lakehouse does not simplify identity management or data classification.

**The workload distinction persists.** Exploration and operational analytics are different disciplines with different requirements. An analyst prototyping a new metric needs flexibility and tolerance for messy data. A finance team running monthly close needs determinism and auditability. The lakehouse unifies storage. It does not unify these two modes of working.

Redshift Spectrum and Redshift Serverless are AWS's explicit acknowledgment of this convergence. Spectrum lets Redshift query external data in S3 and push down some operations, but external lake storage still has a different performance and optimization profile than native Redshift tables. Serverless removes capacity management. The boundary between lake and warehouse blurs, but the workload boundary remains.

## The uncomfortable convergence

The lake-warehouse split was never really about technology. It was about workload shape. Exploratory queries need flexibility, cheap storage, and tolerance for heterogeneous formats. Operational queries need determinism, concurrency guarantees, and contracts that the rest of the business can depend on.

The lakehouse dissolves the storage boundary. Open table formats give S3 the transactional semantics it lacked. One copy of the data serves both workloads. The ETL copy — the most expensive and fragile piece of the old architecture — disappears for many use cases.

But the workload boundary stays. An analyst exploring a new dataset and a CFO dashboard refreshing at 8 AM are fundamentally different consumption patterns. No amount of metadata engineering changes that. The lakehouse gives you a single storage layer with stronger table semantics and less data duplication. It does not give you a single engine that optimally serves every query pattern.

Choose based on what your queries actually need — not on where the vendor drew the product boundary.
