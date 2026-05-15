---
layout: post
title: "What Kubernetes Actually Runs: Deployment, Job, and Service"
date: 2026-05-13
categories: [Software Engineering, Computational Biology]
tags: [kubernetes, k8s, deployment, job, service, orchestration, bioinformatics, workflow-systems, cloud]
description: "A working mental model of Kubernetes for computational biology: how Deployments, Jobs, and Services map to long-lived services and batch pipeline steps, plus the supporting objects — Pods, ConfigMaps, Secrets, PVCs, Ingress — that make the architecture coherent."
excerpt: "Kubernetes is a declarative system. Each object describes one specific thing the control plane should keep true. Once that idea is stable, the API stops looking like a vocabulary problem and starts looking like a small, internally consistent design."
---

Kubernetes introductions tend to hand you the full object catalog at once — Pods, Deployments, Services, ReplicaSets, ConfigMaps, Secrets, PVCs, StatefulSets, DaemonSets, Ingress, Namespaces, CronJobs — and then leave you to work out why those specific objects exist together. The list is technically correct and operationally useless until you find the frame.

The frame is small:

> Kubernetes is a declarative system for running workloads. Each object describes one specific thing the control plane should keep true.

Almost every object is either a description of a workload, a description of how that workload is reached on the network, or a description of what that workload depends on. Once that idea is stable, the API stops looking like a vocabulary problem and starts looking like a small, internally consistent design.

I came to Kubernetes from a workflow-engine background — Nextflow, Snakemake, Argo Workflows — and the mapping is cleaner than the introductory docs make it look: a workflow step is essentially a Kubernetes `Job`. That observation collapses a lot of fog. From there the rest fell into place. `Deployments` are for the things that should keep running. `Services` give those Pods stable addresses. The supporting cast exists so the whole thing stays portable across dev, staging, and production.

This post is the synthesis: the three workload-shaped objects most teams actually use, plus the supporting cast that makes them portable. For the process model below the kubelet — what a container actually runs when it starts — see [the container internals piece]({% post_url 2026-05-14-what-a-container-actually-runs-entrypoint-cmd-and-why-it-clicked %}).

> **Key takeaways**
>
> - **Deployment** keeps an app running. Use it for APIs, dashboards, model servers.
> - **Job** runs a task to completion. Use it for QC, alignment, training, batch steps.
> - **Service** gives a stable network address to a set of Pods.
> - **Pods** are the unit of execution. You almost never create them by hand.
> - The supporting cast — `ConfigMap`, `Secret`, `PVC`, `Ingress`, `Namespace` — exists so workloads stay portable.

---

## The Frame: Declared State, Not Imperative Commands

In Kubernetes you do not tell the cluster what to do. You tell it what should be true.

```text
"Run 3 copies of my API."
"Expose them at a stable address."
"Restart them if they crash."
"Give them 500 GiB of persistent storage."
```

A controller in the control plane reads that description and tries to make the cluster match. If a Pod dies, a controller notices the mismatch and creates a replacement. If you change `replicas: 3` to `replicas: 5`, a controller starts two more.

That principle explains why the API has so many object types. Each kind of object describes a different thing the controller should keep true.

```text
Cluster
├── Control plane: decides what should happen
└── Nodes: machines that run workloads
    └── Pods: actual running units
        └── Containers: actual programs
```

Every other object in Kubernetes is, in some sense, instructions for how the control plane should manage Pods.

---

## Pod: the Smallest Running Thing

A Pod is the smallest thing Kubernetes runs. It contains one or more containers and represents one running workload — an analysis task, a notebook server, an API instance, or a single pipeline step.

```text
Pod
└── Container
    └── Runs a program (FastAPI service, QC script, model inference, etc.)
```

In production, you do **not** create Pods by hand. You create higher-level objects that create Pods for you. A bare Pod describes "this exact Pod should exist," which is rarely what you actually want. You usually want one of:

- "Keep N copies of this app running" → Deployment
- "Run this task to completion once" → Job
- "Run this task on a schedule" → CronJob
- "Run one of these on every node" → DaemonSet
- "Run a stateful set of Pods with stable identity" → StatefulSet

Pods are the unit. The higher-level objects are the policies.

---

## Deployment: For Things That Should Keep Running

A Deployment is the right object whenever the application should stay alive — REST APIs, web apps, dashboards, Shiny and Streamlit apps, JupyterHub frontends, model-serving endpoints, internal annotation services. If the program reads requests, returns responses, and is supposed to be available again tomorrow, it belongs in a Deployment.

A Deployment says, in effect:

> Keep N copies of this app running. If one dies, replace it. If I change the image, roll the new version out gradually.

Internally:

```text
Deployment
└── ReplicaSet
    ├── Pod 1
    ├── Pod 2
    └── Pod 3
```

A minimal manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: annotation-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: annotation-api
  template:
    metadata:
      labels:
        app: annotation-api
    spec:
      containers:
        - name: annotation-api
          image: registry.example.com/bio/annotation-api:v1
          ports:
            - containerPort: 8080
```

Three Pods of `annotation-api`, each listening on 8080.

The most useful behavior in practice is the rolling update. When you push a new image:

```bash
kubectl set image deployment/annotation-api \
  annotation-api=registry.example.com/bio/annotation-api:v2
```

Kubernetes starts new v2 Pods, waits until they pass health checks, removes old v1 Pods, and repeats until the whole Deployment has been replaced. If something breaks:

```bash
kubectl rollout undo deployment/annotation-api
```

You did not write any of that update logic. The Deployment object describes the desired state; a controller does the rest.

---

## Job: For Things That Should Finish

A Job is the right object whenever the work is supposed to end.

This is where most computational biology workloads actually live: FASTQ QC, alignment, variant calling, batch annotation, model training, one-time database migrations, scheduled report generation. None of these are services. They read declared inputs, compute deterministic outputs, and exit.

A Deployment is for "keep running."
A Job is for "run, finish, and stop."

```text
Job
└── Pod
    └── Container
        └── Script or command (exits when done)
```

A minimal Job:

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
          image: registry.example.com/bio/fastq-qc:v1
          args:
            - "--input"
            - "/data/sample.fastq.gz"
            - "--output"
            - "/results/qc.html"
      restartPolicy: Never
```

A Job ends in one of three states:

```text
Active     = still running
Succeeded  = completed successfully
Failed     = failed too many times
```

If you are coming from a workflow background — Nextflow, Snakemake, Argo Workflows — a Kubernetes Job will feel familiar. It is essentially the unit those workflow engines target when they run on Kubernetes. One step of a pipeline is one Job. Once that mapping clicks, the line between "workflow engine" and "Kubernetes" stops looking sharp at all.

For recurring batch work, the matching object is `CronJob`, which creates a Job on a schedule:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-qc-report
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: report
              image: registry.example.com/bio/qc-report:v1
          restartPolicy: Never
```

Cron syntax in a Kubernetes object — exactly what it sounds like.

---

## Service: A Stable Address for Disposable Pods

Pods are temporary. They get rescheduled, restarted, and replaced. Their IPs change. That is by design — but it means you cannot point a client at a Pod IP and expect it to keep working.

A Service is the answer.

> A Service is a stable network name that forwards traffic to whichever Pods currently match a label.

That is the entire idea. The Service does not own the Pods, replicate them, or restart them. It just routes traffic.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: annotation-api-service
spec:
  selector:
    app: annotation-api
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

Stable internal address `annotation-api-service` on port 80, forwarding to port 8080 on every Pod with the label `app=annotation-api`.

If the Deployment scales from 3 Pods to 10, the Service automatically picks up the new Pods. If a Pod is rescheduled to a different node, the Service silently re-routes. The clients never need to know.

The four common Service types:

| Type             | Meaning                                    | Use case                                |
|------------------|--------------------------------------------|-----------------------------------------|
| **ClusterIP**    | Internal-only address inside the cluster   | App-to-app traffic                      |
| **NodePort**     | Exposes the Service on each node's IP/port | Quick external testing                  |
| **LoadBalancer** | Provisions an external cloud load balancer | Production external API                 |
| **ExternalName** | DNS alias to an external host              | Pointing in-cluster names at outside services |

Most internal services are `ClusterIP`. External traffic usually arrives through an Ingress, covered below.

The cleanest way to read Deployment and Service together is that they answer two different questions:

```text
Deployment → "How many of these should be running, and which version?"
Service    → "What address do I send traffic to?"
```

They are independent objects on purpose. You can change one without changing the other.

---

## How These Three Fit Together

For a typical internal API:

```text
Namespace: bio-prod

Deployment: annotation-api
└── ReplicaSet
    ├── Pod 1 (label: app=annotation-api)
    ├── Pod 2 (label: app=annotation-api)
    └── Pod 3 (label: app=annotation-api)

Service: annotation-api-service
└── selector: app=annotation-api
    → forwards traffic to all matching Pods
```

For a batch analysis task:

```text
Namespace: rnaseq-dev

Job: fastq-qc
└── Pod
    └── Container runs QC script and exits

PVC: analysis-workspace
└── mounted into the Pod at /data
    └── backed by a PV (the actual storage)
```

The three core objects — Deployment, Job, Service — cover most everyday workloads. The remaining pieces exist so those workloads can stay configurable, secret-aware, and persistent.

---

## The Supporting Cast

Each remaining object solves one specific need.

**Namespace** is a logical grouping — a project folder inside a cluster. `rnaseq-dev`, `singlecell-prod`, `ml-platform`. Most resources live inside a namespace; nodes and persistent volumes are cluster-wide.

**ConfigMap** stores non-secret configuration: log level, reference build name, database hostname, feature flags. The point is that the same container image can run across dev, staging, and production by reading a different ConfigMap each time.

**Secret** is the same idea for sensitive values: database passwords, API tokens, S3 keys. The interface is similar to ConfigMap — Pods read Secrets as environment variables or mounted files. Bluntly: do not bake credentials into container images.

**PersistentVolumeClaim (PVC)** is how a Pod requests durable storage. A claim describes what you need; a `PersistentVolume` (PV) is the actual storage; a `StorageClass` defines what kinds of PVs the cluster can provision on demand.

```text
StorageClass  defines how to provision storage
     ↓
PV            actual storage (cloud disk, NFS share, SSD volume)
     ↑
PVC           request for storage that gets bound to a PV
     ↑
Pod           mounts the PVC at a path (e.g., /data)
```

For computational biology, this matters whenever data must outlive the Pod that produced it: FASTQ/BAM/VCF files, single-cell matrices, reference genomes, pipeline outputs, model checkpoints, notebook home directories.

**Ingress** is how external HTTP/HTTPS traffic reaches a Service. A Service stops at the cluster boundary; an Ingress crosses it. It handles hostnames, paths, and TLS:

```text
https://annotation.example.com
        ↓
Ingress  (host + path routing, TLS termination)
        ↓
Service  (stable cluster-internal address)
        ↓
Pods     (managed by a Deployment)
```

That diagram is the whole shape of a typical internal web app.

**StatefulSet** is for workloads that need stable identity and stable storage — databases, distributed systems, queues. A Deployment treats Pods as interchangeable; a StatefulSet gives them stable names like `db-0`, `db-1`, `db-2`, each with its own persistent volume. Most stateless services do not need this.

**DaemonSet** runs one Pod per node, or per selected node. Used for cluster-level agents — log shippers, monitoring exporters, GPU device plugins. Application teams rarely write DaemonSets; platform teams use them heavily.

---

## Decision Guide

The choice between objects collapses into a small set of questions:

- **Deployment** — app should keep running, serve requests, restart automatically, possibly multiple replicas.
- **Job** — task should run once and finish; "success" means the command exited cleanly.
- **CronJob** — that Job should run on a schedule.
- **Service** — other Pods or users need a stable way to reach your Pods.
- **PVC** — data has to survive Pod restarts.
- **ConfigMap** — non-secret configuration. **Secret** — credentials.
- **Ingress** — external HTTP/HTTPS traffic needs to reach a Service.
- **StatefulSet** — replicas need stable identity. **DaemonSet** — something must run on every node.

A typical team runs a handful of Deployments and Services for their APIs and dashboards, a stream of Jobs for their pipelines, a couple of CronJobs for scheduled reports, and lets the platform team worry about the rest.

---

## Command Cheat Sheet

```bash
# List
kubectl get pods         -n <namespace>
kubectl get deployments  -n <namespace>
kubectl get jobs         -n <namespace>
kubectl get services     -n <namespace>
kubectl get pvc          -n <namespace>

# Describe
kubectl describe deployment <name> -n <namespace>
kubectl describe job        <name> -n <namespace>
kubectl describe service    <name> -n <namespace>

# Logs
kubectl logs <pod-name>      -n <namespace>
kubectl logs job/<job-name>  -n <namespace>

# Apply / delete
kubectl apply  -f manifest.yaml
kubectl delete -f manifest.yaml
```

The naming is uniform on purpose. Once the object types make sense, the commands stop feeling like memorization.

---

## Closing

A **Deployment** keeps an app running. A **Job** runs a task and stops. A **Service** gives Pods a stable address. **Pods** are where containers actually execute. **PVCs** make data outlive Pods. **ConfigMaps** and **Secrets** keep configuration out of container images. **Ingress** lets external traffic in. **Namespaces** keep the whole thing organized.

That mental model gets you reading manifests as descriptions of desired state instead of as a wall of vocabulary.

For computational biology specifically: use **Jobs** for pipeline steps and batch analyses, **Deployments** for long-running APIs and dashboards, **Services** to reach those Pods reliably, **PVCs** when results must survive Pod restarts. The orchestration model above the kubelet pairs with [the process model below it]({% post_url 2026-05-14-what-a-container-actually-runs-entrypoint-cmd-and-why-it-clicked %}).
