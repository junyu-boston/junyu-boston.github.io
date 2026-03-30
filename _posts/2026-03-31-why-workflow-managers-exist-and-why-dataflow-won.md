---
title: "Why Workflow Managers Exist — And Why Dataflow Won"
date: 2026-03-31
categories: [Computer Science]
tags: [nextflow, bioinformatics, cloud-computing, data-engineering, workflow-orchestration]
---

*Part 1 of 4 — From Scripts to Systems: The CS Theory Behind Nextflow*

My first production genomics pipeline was about 2,000 lines of Bash. It called STAR for alignment, GATK for variant calling, and a half-dozen custom Python scripts for filtering and annotation, all wired together with pipes and temporary files. It worked. It ran for months without incident — until the server ran out of disk space at step 47 of a 60-step workflow.

The question was immediate: which steps actually completed? Which intermediate files are trustworthy? Which need to be regenerated? I had no answers. The script had no concept of checkpoints. I added `set -e` and ran the entire thing from the beginning, burning eight hours of compute on steps that had already finished cleanly.

This is not a workflow management problem. It is a dependency graph problem with a correctness requirement.

### What a workflow manager actually is

Strip away the domain-specific terminology and a pipeline is a directed acyclic graph. G = (V, E), where V is a set of processes and E is the set of data dependencies between them. Process B depends on process A if B requires an output that A produces. The graph must be acyclic — no circular dependencies — otherwise there is no valid execution order.

A workflow manager is a runtime that executes this graph with three guarantees. First, dependency tracking: no process runs before all of its inputs exist and are valid. Second, parallelism: independent processes — those with no edge between them — run concurrently. Third, fault recovery: when a run fails partway through, the manager can resume from the last successful process instead of starting over.

These are the three things my Bash script could not do. It ran steps sequentially regardless of independence. It had no concept of which outputs were valid. And it could not resume without human intervention.

Any topological sort of the DAG gives a valid sequential execution order. But the real value is that *any* topological ordering that respects edges is valid — which means the manager can exploit all available parallelism at each layer of the graph. The width of the DAG at a given topological layer is the upper bound on concurrent tasks at that stage.

### The Make/Snakemake model

The oldest solution to DAG execution is Make, written in 1976. Make defines rules: each rule specifies an output file, input files, and a recipe (shell command) to produce the output from the inputs. Make derives the dependency graph from file timestamps — if an input is newer than the output, the output is stale and the recipe re-runs.

Snakemake, introduced in 2012, applies the same idea to bioinformatics. Rules match output file name patterns, and dependencies are declared as input file patterns. The scheduler derives the DAG by pattern matching. You declare what you want (output files), and Snakemake figures out what to run to get there.

```python
rule align:
    input:  "data/{sample}.fastq.gz"
    output: "results/{sample}.bam"
    shell:  "bwa mem ref.fa {input} | samtools view -b > {output}"
```

This works well when the set of input files is known upfront and each rule produces a predictable output file pattern. For many workflows — especially file-transformation workflows where one input produces one output — it is clean and effective.

The friction shows up in data-parallel genomics. Consider 100 samples, each passing through alignment, variant calling, and annotation. Snakemake requires the sample list to be enumerated before execution to expand the DAG statically. You define a `SAMPLES` list in the Snakefile, and wildcards expand against it.

When sample metadata changes mid-run — a new sample arrives, or a sample fails QC and needs to be excluded — the file-matching model requires redefining the DAG from scratch. When a process emits a variable number of outputs — splitting a BAM into per-chromosome shards where the count is not known ahead of time — the pattern-matching model becomes awkward. You need dynamic re-evaluation or checkpoints, features that were bolted on later because the original model assumed static file sets.

None of this makes Snakemake a bad tool. For workflows with known inputs and one-to-one file transformations, it is simple and correct. But the underlying computational model has a boundary, and data-parallel genomics sits right on it.

### The dataflow alternative

Nextflow approaches the same problem from a different computational model. Instead of rules that match file patterns, Nextflow defines processes that subscribe to input channels and emit to output channels.

A channel is an asynchronous queue. A process fires whenever a complete set of inputs arrives on its input channels, without any process needing to know the total count of items. The runtime schedules as many concurrent instances of a process as the executor allows — one per input tuple.

```groovy
process ALIGN {
    input:
    tuple val(meta), path(reads)

    output:
    tuple val(meta), path("*.bam")

    script:
    """
    bwa mem ref.fa ${reads} | samtools view -b > ${meta.id}.bam
    """
}
```

This process does not know how many samples exist. It does not enumerate them. It receives one sample at a time from an upstream channel, processes it, and emits the result to a downstream channel. If 100 samples flow through the channel, the runtime creates 100 independent task executions. If 500 samples flow through, it creates 500. The pipeline code is identical.

In graph terms, the DAG still exists — Nextflow enforces acyclicity at compile time. But the execution model is reactive rather than declarative. Processes do not declare which files they need by name pattern. They declare the *shape* of data they accept and produce. The runtime matches shapes and routes data accordingly.

### Why this matters for data-parallel workloads

The unit of parallelism in genomics is a sample, not a step. One hundred samples through an alignment step means 100 independent executions of the same process. The question is whether the computational model makes this natural or requires workarounds.

In the file-matching model, parallelism across samples requires enumerating the sample list and expanding the DAG with one node per sample per step. The graph gets wide, but the width is determined at DAG construction time.

In the dataflow model, parallelism across samples is the default behavior. Each item on a channel is an independent unit of work. The runtime parallelizes automatically up to the executor's capacity. No sample list. No upfront enumeration. No DAG reconstruction when a sample is added or removed.

This extends to the harder cases. Scatter-gather operations — where one process emits multiple outputs per input (splitting a BAM into shards) and a downstream process needs to collect all shards for one sample — map naturally to channel operators like `groupTuple`. The data determines the structure, not a statically declared file pattern.

### The cost of correctness

The complexity of Nextflow's syntax — channels, operators, process input/output blocks, the `emit` keyword, the distinction between `val` and `path` — is not arbitrary verbosity. It is the surface area of a computational model designed for reactive, data-parallel execution with deterministic caching.

Every process in Nextflow is intended to be a pure function: given the same inputs and the same container, it produces the same outputs. The runtime hashes the inputs, the container digest, and the script text into a unique task identifier. If that identifier already exists in the work directory with a successful exit code, the task is skipped on resume. This is why `-resume` works correctly by construction, not by heuristic — a topic I will return to in Part 4.

When I first encountered Nextflow, I found the syntax noisy compared to Snakemake's compact rule definitions. I wanted the simplicity of pattern matching. What I did not understand was that the verbosity was buying me something: an explicit type signature for my data pipeline. The channel declarations are the interface contract. The operators are typed transformations. The process blocks are pure functions with declared inputs and outputs.

Once I understood the computational model underneath, the syntax stopped feeling like an obstacle and started feeling like a type system — one that made my pipelines resumable, parallelizable, and portable across execution environments without changing a single line of pipeline code.
