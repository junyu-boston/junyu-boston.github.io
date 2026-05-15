---
layout: post
title: "What a Container Actually Runs: ENTRYPOINT, CMD, and the Process Lifecycle"
date: 2026-05-14
categories: [Software Engineering, Computational Biology]
tags: [docker, containers, entrypoint, cmd, kubernetes, bioinformatics, workflow-systems]
description: "A working mental model of Docker containers for computational biology: what ENTRYPOINT and CMD actually do, why the lifecycle of the main process is the lifecycle of the container, and how that maps cleanly onto Kubernetes Jobs and Deployments."
excerpt: "A container is not a tiny virtual machine. It is a packaged environment that starts one process — and the lifecycle of that process is the lifecycle of the container."
---

A container is not a tiny virtual machine. It is a packaged environment that starts one process, and the lifecycle of that process is the lifecycle of the container.

That single shift makes the rest of Docker tractable. `ENTRYPOINT` and `CMD` become the two knobs that define the process. Kubernetes `Deployment` and `Job` become the two natural shapes of containerized work — long-lived services and run-to-completion batch jobs.

This post is the working model: image vs container, the role of `ENTRYPOINT` and `CMD`, the two process lifecycles, and how all of that maps cleanly onto Kubernetes. For the orchestration model above the container, see [the Kubernetes Deployment, Job, and Service piece]({% post_url 2026-05-13-kubernetes-deployment-job-service-mental-model %}).

> **Key takeaways**
>
> - A Docker image is a packaged environment plus startup defaults. A container is a running instance.
> - `ENTRYPOINT` and `CMD` define what process starts inside the container.
> - The container's lifetime equals that process's lifetime: services keep running, batch jobs exit.
> - This maps directly to Kubernetes `Deployment` (long-lived) and `Job` (run-to-completion).

---

## The Short Version

```text
Docker image = packaged software environment
Container    = running instance of that image
ENTRYPOINT   = main executable
CMD          = default arguments
```

When I run:

```bash
docker run my-image
```

Docker is asking one question: *what command should start inside this container?* The answer comes from the image's `ENTRYPOINT` and `CMD`, possibly overridden at runtime.

---

## The Mental Model in One Diagram

<figure style="max-width: 760px; margin: 1.5rem auto;">
  <div style="border: 1px solid #cbd5e1; border-radius: 18px; padding: 1rem 1.25rem; background: linear-gradient(135deg, #ffffff, #f8fafc); box-shadow: 0 10px 24px rgba(15, 23, 42, 0.08);">
    <div style="display: inline-block; padding: 0.35rem 0.7rem; border-radius: 999px; background: #dbeafe; color: #1d4ed8; font-weight: 700; font-size: 0.95rem;">1. Docker image</div>
    <div style="margin-top: 0.8rem; font-size: 1.3rem; font-weight: 700; color: #0f172a;">Packaged environment plus startup defaults</div>
    <div style="margin-top: 0.55rem; color: #475569;">A reproducible bundle for a QC script, aligner wrapper, Nextflow process, or small analysis service.</div>
    <div style="margin-top: 0.75rem; font-family: SFMono-Regular, Menlo, Monaco, Consolas, monospace; color: #0f172a;">ENTRYPOINT - executable</div>
    <div style="font-family: SFMono-Regular, Menlo, Monaco, Consolas, monospace; color: #0f172a;">CMD - default args</div>
  </div>

  <div style="text-align: center; font-size: 1.5rem; color: #64748b; margin: 0.45rem 0;">↓</div>

  <div style="border: 1px solid #cbd5e1; border-radius: 18px; padding: 1rem 1.25rem; background: linear-gradient(135deg, #ffffff, #f8fafc); box-shadow: 0 10px 24px rgba(15, 23, 42, 0.08);">
    <div style="display: inline-block; padding: 0.35rem 0.7rem; border-radius: 999px; background: #dcfce7; color: #166534; font-weight: 700; font-size: 0.95rem;">2. Container start</div>
    <div style="margin-top: 0.8rem; font-size: 1.3rem; font-weight: 700; color: #0f172a;">One main process launches</div>
    <div style="margin-top: 0.7rem; font-family: SFMono-Regular, Menlo, Monaco, Consolas, monospace; color: #0f172a;">docker run qc-image --input sample.fastq</div>
    <div style="margin-top: 0.55rem; color: #475569;">Docker combines image defaults with runtime arguments and starts the program inside the container.</div>
  </div>

  <div style="text-align: center; font-size: 1rem; font-weight: 700; color: #475569; margin: 0.7rem 0 0.8rem;">Two common lifecycles</div>

  <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(260px, 1fr)); gap: 0.85rem;">
    <div style="border: 1px solid #93c5fd; border-radius: 18px; padding: 1rem 1.25rem; background: linear-gradient(135deg, #e0f2fe, #dbeafe); box-shadow: 0 10px 24px rgba(15, 23, 42, 0.08);">
      <div style="display: inline-block; padding: 0.35rem 0.7rem; border-radius: 999px; background: #ffffff; color: #1d4ed8; font-weight: 700; font-size: 0.95rem;">A. Service</div>
      <div style="margin-top: 0.8rem; font-size: 1.2rem; font-weight: 700; color: #0f172a;">Process keeps listening</div>
      <div style="margin-top: 0.55rem; color: #334155;">API, Jupyter, dashboard, model endpoint</div>
      <div style="margin-top: 0.7rem; font-weight: 700; color: #1e3a8a;">Container stays alive → Kubernetes Deployment</div>
    </div>

    <div style="border: 1px solid #fdba74; border-radius: 18px; padding: 1rem 1.25rem; background: linear-gradient(135deg, #fef3c7, #ffedd5); box-shadow: 0 10px 24px rgba(15, 23, 42, 0.08);">
      <div style="display: inline-block; padding: 0.35rem 0.7rem; border-radius: 999px; background: #ffffff; color: #b45309; font-weight: 700; font-size: 0.95rem;">B. Batch</div>
      <div style="margin-top: 0.8rem; font-size: 1.2rem; font-weight: 700; color: #7c2d12;">Process finishes work and exits</div>
      <div style="margin-top: 0.55rem; color: #9a3412;">QC, alignment, variant calling, Nextflow step</div>
      <div style="margin-top: 0.7rem; font-weight: 700; color: #b45309;">Container exits → Kubernetes Job</div>
    </div>
  </div>

  <figcaption style="margin-top: 0.8rem; text-align: center; color: #475569; font-style: italic;">
    Figure: the useful question is not "what machine is this?" but "what executable starts here, and does it stay alive or finish the job?"
  </figcaption>
</figure>

---

## ENTRYPOINT vs CMD

```text
ENTRYPOINT = the executable
CMD        = the default arguments
```

Given:

```dockerfile
ENTRYPOINT ["python", "run_analysis.py"]
CMD ["--help"]
```

Default invocation:

```bash
python run_analysis.py --help
```

Override the args at runtime, keep the entrypoint:

```bash
docker run my-image --input sample.fastq --output results/
# becomes: python run_analysis.py --input sample.fastq --output results/
```

The image declares *what program this container is for*. Runtime arguments declare *what specific work this run should do*. The split is the whole point.

---

## Two Lifecycles, One Model

**Service container.** Process keeps listening — Flask, FastAPI, Jupyter, Postgres, model endpoint. Container stays alive because the main process stays alive. Maps to Kubernetes `Deployment`.

**Batch container.** Process does work and exits — alignment, QC, variant calling, training, a Nextflow step. Container exits because the main process exits, and the exit code is the job's outcome. Maps to Kubernetes `Job`, `CronJob`, or a workflow engine like Nextflow, Snakemake, or Argo.

A short-lived container is not a failure case. It is what you want for declared inputs → deterministic work → declared outputs → done.

---

## Kubernetes Just Reuses the Model

Kubernetes does not invent a new execution model. With no overrides:

```yaml
containers:
  - name: qc
    image: rnaseq-qc:v1
```

Kubernetes starts the container with the image's default `ENTRYPOINT` and `CMD`. Override them explicitly:

```yaml
containers:
  - name: qc
    image: rnaseq-qc:v1
    command: ["python"]                                  # overrides ENTRYPOINT
    args: ["run_qc.py", "--input", "/data/sample.fastq"] # overrides CMD
```

```text
Kubernetes command  →  overrides Docker ENTRYPOINT
Kubernetes args     →  overrides Docker CMD
```

A bioinformatics step lines up with this naturally: one executable, one declared input, one declared output, one exit code.

---

## A Bioinformatics Example, End to End

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY qc.py .
ENTRYPOINT ["python", "qc.py"]
```

Local Docker:

```bash
docker run qc-image --input sample.fastq --output qc_report.html
```

Same image, Kubernetes `Job`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: fastq-qc
spec:
  template:
    spec:
      containers:
        - name: qc
          image: qc-image:v1
          args: ["--input", "/data/sample.fastq", "--output", "/results/qc.html"]
      restartPolicy: Never
```

Read inputs, do the work, write outputs, exit. Same image, same contract, two execution surfaces.

---

## The Whole Idea in One Sentence

A container starts by running a program, and its lifetime is that program's lifetime. The rest of Docker — and most of Kubernetes — is a consequence of that one fact.

For a bioinformatics workflow, that maps cleanly to how the work already wants to be expressed: one toolchain, one executable, one set of declared inputs, one set of outputs, one exit code. The image is the reproducible surface. The runtime arguments are the specific run.

Once the model is stable, Docker and Kubernetes stop being two large topics and start being one small one with two execution surfaces. For the orchestration side — Deployments, Jobs, Services — see [the Kubernetes piece]({% post_url 2026-05-13-kubernetes-deployment-job-service-mental-model %}).
