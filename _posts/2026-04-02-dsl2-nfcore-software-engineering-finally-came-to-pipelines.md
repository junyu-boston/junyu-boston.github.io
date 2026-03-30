---
title: "DSL2 + nf-core: Software Engineering Finally Came to Pipelines"
date: 2026-04-02
categories: [Computer Science]
tags: [nextflow, bioinformatics, cloud-computing, data-engineering, software-engineering]
---

*Part 3 of 4 — From Scripts to Systems: The CS Theory Behind Nextflow*

Parts [1](/posts/why-workflow-managers-exist-and-why-dataflow-won/) and [2](/posts/nextflow-channel-algebra-pipelines-as-dataflow-programs/) established the computational model — processes as pure functions, channels as typed async streams, operators as composable transforms. Part 3 is about what happens when you apply software engineering discipline to that model.

DSL1 Nextflow looked like a script. One file. Every process defined inline. Reusing a process in a second pipeline meant copying it, renaming the variables, hoping nothing broke. There was no `import`, no module boundary, no way to version a single process independently of the pipeline that contained it.

DSL2, introduced in 2020, changed the fundamental unit of organization: the process became an importable module with a defined interface. And nf-core took that module system and applied twenty years of software engineering practice to it — typed interfaces, a dependency registry, schema validation, CI/CD, and unit testing. Here is what each of those means in concrete pipeline terms.

### DSL1 vs DSL2: the module system

DSL1 had one namespace. Every process, every channel, every variable lived in a single file scope. A 2,000-line pipeline had 2,000 lines in one file. Extracting a process into a separate file was not supported by the language.

DSL2 introduced three things that changed the structure of Nextflow code.

First, **process imports**:

```groovy
include { SAMTOOLS_SORT } from './modules/local/samtools_sort'
include { STAR_ALIGN }    from './modules/nf-core/star/align'
```

A process is now a file. It has an input contract, an output contract, and a script body. It can be imported by any pipeline that matches its interface. This is the same transition that JavaScript made from inline `<script>` blocks to ES modules — the unit of reuse moved from "copy this block" to "import this file."

Second, **subworkflows**: a named composition of processes that takes channels as input and returns channels as output.

```groovy
workflow ALIGN_AND_SORT {
    take: ch_reads
    main:
        STAR_ALIGN(ch_reads)
        SAMTOOLS_SORT(STAR_ALIGN.out.bam)
    emit:
        bam = SAMTOOLS_SORT.out.bam
}
```

A subworkflow is a higher-order function over processes. It encapsulates a sequence of steps behind a clean interface. The calling workflow does not need to know the internal wiring — only what goes in and what comes out.

Third, **process aliasing**: the same process can be included multiple times under different names. `include { FASTQC as FASTQC_RAW } from './modules/nf-core/fastqc'` and `include { FASTQC as FASTQC_TRIMMED }` in the same pipeline, running the identical tool at two different stages with two different inputs. In DSL1, you would have duplicated the entire process block.

### Process as a typed function

In DSL2, a process has an explicit input block and output block that serve as its type signature:

```groovy
process SAMTOOLS_SORT {
    input:
    tuple val(meta), path(bam)

    output:
    tuple val(meta), path("*.sorted.bam"), emit: bam
    tuple val(meta), path("*.sorted.bam.bai"), emit: bai

    script:
    """
    samtools sort -o ${meta.id}.sorted.bam ${bam}
    samtools index ${meta.id}.sorted.bam
    """
}
```

The `tuple val(meta), path(bam)` input declaration is the function signature. Any upstream channel emitting a tuple of that shape — a metadata map followed by a path to a BAM file — can connect to this process. The `emit: bam` label on the output lets downstream consumers reference `SAMTOOLS_SORT.out.bam` by name rather than by position.

This is structural typing enforced by convention, not by a compiler. Nextflow will not reject a mismatched tuple at compile time — you will get a runtime error when the process tries to resolve a path that does not exist. But nf-core's linter catches many of these mismatches statically, which brings us to the registry.

### nf-core modules as a typed registry

nf-core/modules is a public repository of over 1,300 process definitions. Each module provides:

- A standardized input/output interface following the META map convention. The first element of every input and output tuple is `val(meta)`. This is the contract that makes modules composable.
- A pinned container image — both Docker and Singularity — versioned with the tool version it wraps. `samtools/sort:1.18--h50ea8bc_1` means samtools 1.18, built from a specific Bioconda recipe, reproducible across time and machines.
- A test definition (nf-test) that verifies the module runs correctly on known input data and produces expected output checksums.
- A `meta.yml` schema describing every input channel, output channel, and parameter with types and descriptions.

Installing a module into your pipeline:

```bash
nf-core modules install samtools/sort
```

This pulls the process file, its container reference, and its test into your project's `modules/nf-core/` directory. It is `npm install` for bioinformatics processes — a versioned dependency with a defined interface.

The META map convention is what makes this registry work at scale. Because every module accepts `tuple val(meta), path(...)` as primary input and emits the same structure, any module can connect to any other module that operates on the same data type without writing adapter code. BAM modules connect to BAM modules. VCF modules connect to VCF modules. The meta map carries the sample identity through every connection. It is an interface contract that one thousand independent contributors follow because the alternative — custom tuple shapes per module — would make composition impossible.

### Parameter validation as a type system

Pipeline parameters in Nextflow are untyped by default. `params.input` accepts any string. `params.max_cpus` accepts any value. There is no validation, no constraint checking, no schema. A typo in a parameter name silently creates a new, unused parameter.

nf-core pipelines add `nextflow_schema.json`: a JSON Schema document declaring every parameter's type, constraints, default value, and description.

```json
{
  "input": {
    "type": "string",
    "format": "file-path",
    "description": "Path to comma-separated samplesheet",
    "pattern": "^\\S+\\.csv$",
    "exists": true
  },
  "max_cpus": {
    "type": "integer",
    "description": "Maximum number of CPUs per process",
    "default": 16,
    "minimum": 1
  }
}
```

The nf-validation plugin reads this schema at pipeline startup, before any process executes. Running `nextflow run pipeline --input bad_path.txt` fails immediately with a human-readable error: "Input file does not exist" or "Parameter does not match expected pattern." The error appears at launch, not buried in a process log at step 23 after an hour of compute.

This is type checking at the pipeline boundary. The pipeline's parameters are its public API. The schema is the API contract.

### Unit testing for processes

nf-test enables snapshot-based testing of individual Nextflow processes. A test file specifies input data, runs the process in isolation inside a minimal workflow, and asserts that the output matches a stored snapshot — typically MD5 checksums of output files or structural assertions on channel contents.

```groovy
nextflow_process {
    name "Test SAMTOOLS_SORT"
    process "SAMTOOLS_SORT"

    test("Should sort BAM file") {
        when {
            process {
                """
                input[0] = [[ id: 'test' ], file('test.bam')]
                """
            }
        }
        then {
            assert process.success
            assert snapshot(process.out.bam).match()
        }
    }
}
```

Every module in the nf-core registry has a passing test suite before it is merged. When a contributor updates a container version — upgrading samtools from 1.17 to 1.18 — nf-test immediately reveals whether the output changed. If checksums differ, either the change is intentional (update the snapshot) or it is a regression (investigate).

This closes a gap that existed for years in bioinformatics pipelines. Individual tools had their own test suites, but the *integration* of a tool into a pipeline — the input wiring, the output naming, the container compatibility — was tested only by running the full pipeline end to end. nf-test makes the process the unit of testing, not the pipeline.

### Linting and continuous integration

`nf-core lint` enforces approximately 150 structural checks on a pipeline: correct directory layout, valid `nextflow_schema.json`, required metadata files, parameter naming conventions, module versioning, container declarations, and more. It runs on every pull request to any nf-core pipeline via GitHub Actions.

The checks are opinionated. Parameter names must be `snake_case`. Modules must declare `conda`, `container`, and `singularity` directives. Output channels must use `emit` labels. These are not suggestions — a failing lint check blocks the PR.

`nf-core sync` propagates template updates from the nf-core pipeline template to every downstream pipeline. When the template adds a new GitHub Actions workflow, updates a linting rule, or patches a security issue, `nf-core sync` creates a PR in every pipeline that inherits from the template. This is the same mechanism as inheriting from a shared base class and pulling upstream changes — except applied to pipeline infrastructure rather than application code.

### What this adds up to

Take a step back and list what nf-core provides on top of raw Nextflow:

- **Module system** with importable, versioned, aliasable processes
- **Typed interfaces** via the META map convention — structural typing by community contract
- **Dependency registry** with 1,300+ modules, each with pinned containers and test suites
- **Schema validation** for pipeline parameters at launch time
- **Unit testing** for individual processes via nf-test
- **Linting** enforcing 150+ structural and naming conventions
- **Template inheritance** propagating infrastructure updates to all pipelines

These are not bioinformatics innovations. They are software engineering practices — the same ones that web developers, backend engineers, and platform teams have used for decades. Typed interfaces. Package registries. Schema validation. Unit tests. CI/CD. Linting. Template-based project scaffolding.

The contribution of DSL2 and nf-core is applying them systematically to a domain that historically treated pipelines as disposable scripts. A bioinformatics pipeline can be held to the same engineering standards as any production service. DSL2 made it structurally possible. nf-core made it the default.
