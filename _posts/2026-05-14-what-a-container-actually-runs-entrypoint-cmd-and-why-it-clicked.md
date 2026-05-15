---
layout: post
title: "What a Container Actually Runs: ENTRYPOINT, CMD, and the Pitfalls That Bite in Production"
date: 2026-05-14
categories: [Software Engineering, Computational Biology]
tags: [docker, containers, entrypoint, cmd, kubernetes, bioinformatics, workflow-systems]
description: "A working mental model of Docker containers for computational biology: what ENTRYPOINT and CMD actually do, how the model maps to Kubernetes Jobs and Deployments, and the production pitfalls (PID 1, shell vs exec form, non-root) that the mini-VM analogy hides."
excerpt: "A container is not a tiny virtual machine. It is a packaged environment that starts one process — and the lifecycle of that process is the lifecycle of the container."
---

A container is not a tiny virtual machine. It is a packaged environment that starts one process, and the lifecycle of that process is the lifecycle of the container.

That single shift makes the rest of Docker tractable. `ENTRYPOINT` and `CMD` become the two knobs that define the process. Kubernetes `Deployment` and `Job` become the two natural shapes of containerized work. And the production failure modes — zombies, orphaned signals, dropped writes on shutdown, root-owned files in mounted volumes — stop looking like Docker quirks and start looking like predictable consequences of a simple model.

This post is the model I actually use, plus the three places it bites in production.

> **Key takeaways**
>
> - A Docker image is a packaged environment plus startup defaults. A container is a running instance.
> - `ENTRYPOINT` and `CMD` define what process starts inside the container.
> - The container's lifetime equals that process's lifetime: services keep running, batch jobs exit.
> - This maps directly to Kubernetes `Deployment` (long-lived) and `Job` (run-to-completion).
> - The production pitfalls — PID 1 signal handling, shell vs exec form, non-root users — all stem from the same model.

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
    <div style="margin-top: 0.8rem; font-size: 1.3rem; font-weight: 700; color: #0f172a;">One main process launches as PID 1</div>
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

## Where the Model Bites in Production

The mini-VM analogy hides three failure modes. Each comes from the same fact: **the container's main process is PID 1**, and PID 1 is special on Linux.

### 1. PID 1, signal handling, and zombie processes

PID 1 has two responsibilities most programs don't expect: it must reap orphaned child processes, and it does **not** receive default signal handlers. If your `ENTRYPOINT` is a Python script that spawns subprocesses, those subprocesses' exits become zombies because Python doesn't reap them. And when Kubernetes wants to stop the pod, it sends `SIGTERM` to PID 1 — if your process didn't install a handler, the signal is **ignored**, and the kubelet eventually escalates to `SIGKILL` after `terminationGracePeriodSeconds` (default 30s). That's how you get dropped writes, half-uploaded files, and corrupted partial outputs.

Two fixes, in order of preference:

```dockerfile
# Option A: tell Docker/Kubernetes to use a minimal init.
# Docker:     docker run --init my-image
# Kubernetes: not built-in; use Option B or a sidecar init.

# Option B: bake an init into the image.
RUN apt-get update && apt-get install -y --no-install-recommends tini
ENTRYPOINT ["/usr/bin/tini", "--", "python", "run_analysis.py"]
```

`tini` runs as PID 1, reaps zombies, and forwards `SIGTERM` to your actual program. For long-running services and any batch job that spawns subprocesses, this is the boring default.

### 2. Shell form vs exec form

These two lines look almost identical and behave very differently:

```dockerfile
ENTRYPOINT python run_analysis.py             # shell form
ENTRYPOINT ["python", "run_analysis.py"]      # exec form
```

Shell form runs your command via `/bin/sh -c`, so `sh` becomes PID 1 and your Python process becomes its child. `SIGTERM` from Kubernetes goes to `sh`, which does not forward it. Your program never gets the chance to flush, close handles, or finish writing the current row. You'll see this in production as "graceful shutdown didn't work" — it didn't work because the signal never arrived.

Default to exec form. Use shell form only when you genuinely need shell features (variable expansion, pipes), and pair it with an init.

### 3. Root user, multi-stage, and the volume that bites back

By default, containers run as `root`. Two consequences:

- Files written to a mounted host volume are owned by root, which is hostile to shared cluster filesystems and makes downstream "read this output" steps fail with permission errors.
- Any compromise of the process is a compromise of root inside the container, and depending on your runtime configuration, occasionally outside it.

The fix is a non-root user, and multi-stage builds make it cheap because you can keep the final stage minimal:

```dockerfile
# Stage 1: build dependencies into a fat image.
FROM python:3.11 AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

# Stage 2: ship only what runs.
FROM python:3.11-slim
RUN groupadd --system app && useradd --system --gid app --home /app app
COPY --from=builder /install /usr/local
WORKDIR /app
COPY --chown=app:app qc.py .
USER app
ENTRYPOINT ["python", "qc.py"]
```

Three things changed:

- The runtime image has no compiler toolchain, no build dependencies, no pip cache. Smaller surface area, smaller image, faster pulls.
- A dedicated `app` user owns `/app` and runs the process. Output files land with sane ownership.
- `COPY --chown` avoids a follow-up `chown` layer.

In Kubernetes, reinforce it at the pod level so the manifest doesn't trust the image alone:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
```

`runAsNonRoot: true` will refuse to start a container that's still running as root. That refusal is the feature.

---

## The Whole Idea in One Sentence

A container starts by running a program. Its lifetime is that program's lifetime, that program is PID 1, and the rest of Docker and Kubernetes is consequences of those facts.

For a bioinformatics workflow, that maps cleanly to how the work already wants to be expressed: one toolchain, one executable, one set of declared inputs, one set of outputs, one exit code. The image is the reproducible surface. The runtime arguments are the specific run. The pitfalls are the places where the Linux process model leaks through.

Once the model is stable, Docker and Kubernetes stop being two large topics and start being one small one with two execution surfaces.
