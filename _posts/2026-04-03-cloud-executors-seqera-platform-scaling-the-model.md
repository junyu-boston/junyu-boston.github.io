---
title: "Cloud Executors + Seqera Platform: Scaling the Model"
date: 2026-04-03
categories: [Computer Science]
tags: [nextflow, bioinformatics, cloud-computing, data-engineering, aws]
---

*Part 4 of 4 — From Scripts to Systems: The CS Theory Behind Nextflow*

The first three parts established the computational model and the engineering practices — [Part 1](/posts/why-workflow-managers-exist-and-why-dataflow-won/) covered DAGs and dataflow, [Part 2](/posts/nextflow-channel-algebra-pipelines-as-dataflow-programs/) unpacked the channel algebra, [Part 3](/posts/dsl2-nfcore-software-engineering-finally-came-to-pipelines/) showed how DSL2 and nf-core brought software engineering to pipelines. Part 4 is about what happens when that model meets enterprise infrastructure.

My first AWS Batch job through Nextflow failed with an error about an invalid compute environment. I spent two days in IAM policies, VPC subnet configurations, and job queue settings before the first task actually ran. Then it ran. And so did 200 others — simultaneously, across a fleet of spot instances, without a single change to the pipeline code I had been testing on my laptop.

That moment was clarifying. The two days of debugging were real. But the fact that zero pipeline code changed between local and cloud — that was the abstraction doing exactly what it was designed to do.

### The executor abstraction

Nextflow separates two concerns that most pipeline tools conflate: what runs and where it runs. The pipeline defines the DAG — processes, channels, data flow. The executor defines the runtime environment — local machine, HPC cluster, cloud batch service.

```groovy
// nextflow.config
process.executor = 'awsbatch'
process.queue    = 'nextflow-batch-queue'
aws.region       = 'us-east-1'
aws.batch.cliPath = '/home/ec2-user/miniconda/bin/aws'
```

Change `awsbatch` to `local` and the pipeline runs on your laptop. Change it to `slurm` and it submits jobs to an HPC scheduler. Change it to `google-batch` and it runs on Google Cloud. The pipeline code — the processes, the channels, the operators — is identical in every case.

The executor is a plugin that translates Nextflow's internal task model into whatever the underlying scheduler understands. A task in Nextflow's model is a containerized script with declared inputs, declared outputs, and resource requirements (CPUs, memory, disk). The local executor runs this as a child process. The SLURM executor wraps it in an `sbatch` submission. The AWS Batch executor creates a job definition and submits it to a job queue.

This is the same pattern as a database driver. The application issues queries in SQL. The driver translates them into the wire protocol that PostgreSQL, MySQL, or SQLite understands. The application does not know — and should not need to know — which database engine is on the other end.

### The work directory as a content-addressable cache

In Part 2, I described processes as pure functions: same inputs, same container, same script, same outputs. Part 4 is where that purity pays off.

When Nextflow executes a task, it computes a hash over three things: the content of the input files (not their names or paths — their actual bytes), the container image digest, and the script text. This hash becomes the task's unique identifier.

```text
task_hash = hash(input_file_contents + container_digest + script_text)

work/ab/3f7c2d1e8a.../
    .command.sh          ← the script that ran
    .command.run         ← the wrapper script
    .exitcode            ← 0 = success
    .command.log         ← stdout/stderr
    output_file.bam      ← the actual output
```

On `-resume`, Nextflow walks the DAG and checks each task: does a work directory with this hash exist, and does it contain `.exitcode = 0`? If yes, skip. If no, execute.

The correctness of this caching depends entirely on the purity assumption. Same inputs plus same container plus same script must equal same outputs. If that holds — and for well-written containerized processes it does — then skipping a cached task is always safe. This is not a heuristic. It is content-addressable storage applied to computation, the same principle behind Git objects, Docker layer caching, and Nix store paths.

Understanding what invalidates the cache matters as much as understanding how it works:

**Breaks `-resume`:** Changing the container tag, even if the underlying image is byte-identical (the digest is computed from the tag, not the image content in older Nextflow versions). Modifying any character in the process `script` block. Changing input file content.

**Does not break `-resume`:** Adding a new downstream process. Changing parameters not consumed by the cached task. Reordering unrelated processes in the workflow file. Changing executor configuration.

This last point is significant. You can develop locally with `process.executor = 'local'`, accumulate cached results in the work directory, switch to `process.executor = 'awsbatch'`, and Nextflow will reuse every cached task whose hash matches. The executor is not part of the hash. Only the computation is.

### AWS Batch from Nextflow's perspective

AWS Batch has three layers that pipeline authors need to understand — but only one of them appears in pipeline code.

The **compute environment** defines the fleet: which EC2 instance types, minimum and maximum vCPUs, spot versus on-demand, VPC and subnet configuration. This is infrastructure — set up once by a platform engineer, referenced by name thereafter.

The **job queue** is a priority lane that routes submitted jobs to a compute environment. Multiple queues can point to the same compute environment with different priorities, or to different compute environments entirely. A high-memory queue might route to `r5` instances; a GPU queue might route to `p3` instances.

The **job definition** specifies what runs: container image, vCPU and memory requirements, mount points, environment variables. Nextflow generates job definitions automatically from process directives — the pipeline author never writes one.

The only AWS Batch concept that surfaces in pipeline code is the queue:

```groovy
process {
    withLabel: 'process_low'    { queue = 'standard-queue' }
    withLabel: 'process_high'   { queue = 'high-memory-queue' }
    withLabel: 'process_gpu'    { queue = 'gpu-queue' }
}
```

A process author labels their process `process_high`. The pipeline configuration maps that label to a queue. The queue routes to a compute environment. The compute environment provisions instances. Four layers of abstraction, and the pipeline code touches only one of them.

The remaining complexity — IAM roles, S3 bucket policies, VPC networking, launch templates, ECS agent configuration — is real and substantial. It is also entirely in the infrastructure layer. My two days of debugging were spent there, not in the pipeline. That is the correct place for infrastructure complexity to live.

### What Seqera Platform adds

Raw Nextflow handles computation: define a DAG, execute it on an infrastructure target, cache results, resume on failure. Seqera Platform (formerly Nextflow Tower) handles a different concern: orchestration.

The distinction matters. Computation is "run this pipeline on this data." Orchestration is "who can run which pipeline, on which infrastructure, with what parameters, and how do we track it?"

What Seqera Platform provides:

**Compute environment management.** Define an AWS Batch environment once — instance types, queues, IAM roles, S3 work directory — and reuse it across pipelines, teams, and projects. No one copies `nextflow.config` blocks between repositories. The environment is a named, versioned resource.

**Launch templates.** A parameterized pipeline launch — input samplesheet, reference genome, output directory, resource limits — exposed as a form in a web UI or as a JSON payload to a REST API. A bench scientist can launch a pipeline without SSH access, without editing config files, without knowing what AWS Batch is.

**Run monitoring.** Real-time visualization of the DAG execution: which tasks are running, which are cached, which failed. Per-task resource utilization — actual CPU and memory versus requested. Task logs accessible from the UI without navigating S3 bucket paths.

**Cost attribution.** Per-run and per-process cost breakdowns computed from instance hours and storage usage. When multiple projects share a compute environment, this is how you answer "how much did the exome sequencing project cost this month?" without manual accounting.

**Team access control.** Role-based permissions: who can launch pipelines, who can manage compute environments, who can view runs. In a multi-team environment, this is the difference between a shared platform and a shared AWS account.

### When you need what

The boundary is straightforward.

Raw Nextflow is sufficient when a single analyst or a small team runs pipelines from the command line, the compute target is stable (one cluster, one cloud account), runs do not require audit trails, and cost attribution is not a concern. This covers a large fraction of academic and early-stage work.

Seqera Platform earns its overhead when multiple teams share compute infrastructure, when non-engineers need to launch pipelines, when production workloads require audit trails and reproducibility records, or when cost attribution across projects is a business requirement. This is the enterprise environment: shared infrastructure, diverse users, accountability.

The platform does not change the computational model. The DAG is the same. The channels are the same. The processes are the same. The caching is the same. What changes is the operational surface area — who can trigger the computation, how it is monitored, and how costs are attributed.

### The abstraction boundary

Looking back at the four parts of this series, one theme runs through all of them: correct abstraction boundaries.

Part 1: the pipeline is a DAG, not a script. The workflow manager separates dependency logic from execution order. Part 2: channels and operators are a typed algebra for data flow, not ad hoc wiring. Part 3: DSL2 and nf-core separate module interfaces from module implementations, enabling a registry of composable parts. Part 4: the executor separates computation from infrastructure, and the platform separates orchestration from computation.

Each layer hides the complexity that belongs to it and exposes only what the layer above needs. The pipeline author does not think about IAM policies. The platform administrator does not think about channel operators. The module developer does not think about job queues.

My first pipeline was 2,000 lines of Bash with no abstraction boundaries at all. The execution logic, the dependency tracking, the file management, the error handling, and the parallelism strategy were all mixed together in one file. When it failed at step 47, every layer was implicated. I could not reason about any single concern in isolation.

The complexity did not go away. The same pipeline, expressed in Nextflow and deployed through Seqera Platform, involves channels, operators, processes, containers, executors, compute environments, job queues, and access control policies. But each piece of complexity lives in its own layer, governed by its own rules, testable in isolation. That is not simplicity. It is well-placed complexity — which, for production systems, is the more useful property.
