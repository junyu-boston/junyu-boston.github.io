---
layout: post
title: "A 5 + 2 Architecture for Biomni-Like Biomedical Agents"
date: 2026-04-29
categories: [AI Systems]
tags: [biomni, biomedical-ai, computational-biology, provenance, evaluation, workflow-engineering]
---

One of the easiest ways to confuse a biomedical AI agent project is to mix together two very different kinds of difficulty.

The first kind is standard engineering difficulty: containers, versioning, logging, manifests, tools, execution environments, and reproducibility.

The second kind is scientific difficulty: deciding whether an answer is biologically sensible, experimentally useful, or genuinely defensible.

Those are not the same problem.

Once I separated them clearly, a Biomni-like architecture became much easier to think about.

> **Key takeaways**
>
> - Provenance belongs inside the local engineering design as a first-class layer.
> - Docker, Git, manifests, and workflow tooling solve much of the reproducibility layer.
> - The local engineering design has five parts: operator, agent, tools, execution, and provenance.
> - Biological validation and evaluation belong in a separate scientific assurance layer.
> - The hard part is not recording what the agent did. The hard part is deciding whether the result is scientifically sound.

---

## The Short Version

If you want to build a Biomni-like app locally, you can solve a large fraction of the systems problem with normal engineering tools.

You can use:

- VS Code as the operator surface,
- Docker for reproducible execution,
- Git for code and configuration versioning,
- MCP for tool access,
- workflow metadata and manifests for provenance,
- database and dataset versioning tools for structured resources.

None of that is exotic.

What remains hard is not the execution trace. What remains hard is scientific judgment.

---

## The Local Design

The cleanest local design is to separate the platform into five engineering parts, then place scientific assurance in a separate layer above them.

### 1. Operator Layer

- VS Code for interaction, prompt iteration, notebooks, and artifact inspection.
- A local workspace with fixed directories for inputs, outputs, reports, and run metadata.

### 2. Agent Layer

- A coding-capable agent that plans tasks, chooses tools, writes Python or R, runs commands, and revises its plan from observed outputs.
- A task state object that stores user goal, current plan, active datasets, and prior intermediate results.

### 3. Tool Layer

- MCP servers that expose biological databases, file access, notebook execution, and computational tools through stable schemas.
- A small number of deterministic wrappers around common operations like annotation lookup, enrichment, variant filtering, or structure retrieval.

### 4. Execution Layer

- Docker containers for Python, R, and any fragile bioinformatics dependencies.
- Mounted volumes for datasets, outputs, and run manifests.
- A job runner that executes generated code with timeouts, resource limits, and captured stdout and stderr.

### 5. Provenance Layer

- A run manifest for every execution.
- Captured prompts, tool calls, parameters, code, artifacts, database releases, and container identifiers.
- Enough metadata to reconstruct what happened without guessing.

![Diagram of a Biomni-inspired local design with five engineering parts: operator, agent, tools, execution, and provenance, plus two scientific assurance parts: biological validation and evaluation](data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIxNDAwIiBoZWlnaHQ9Ijc2MCIgdmlld0JveD0iMCAwIDE0MDAgNzYwIiByb2xlPSJpbWciIGFyaWEtbGFiZWxsZWRieT0idGl0bGUgZGVzYyI+CiAgPHRpdGxlIGlkPSJ0aXRsZSI+QmlvbW5pLWluc3BpcmVkIGxvY2FsIGRlc2lnbiB3aXRoIGFzc3VyYW5jZSBsYXllcnM8L3RpdGxlPgogIDxkZXNjIGlkPSJkZXNjIj5BIGRpYWdyYW0gc2hvd2luZyBmaXZlIGVuZ2luZWVyaW5nIGxheWVyczogb3BlcmF0b3IsIGFnZW50LCB0b29scywgZXhlY3V0aW9uLCBhbmQgcHJvdmVuYW5jZSwgcGx1cyB0d28gYXNzdXJhbmNlIGxheWVyczogYmlvbG9naWNhbCB2YWxpZGF0aW9uIGFuZCBldmFsdWF0aW9uLjwvZGVzYz4KICA8cmVjdCB3aWR0aD0iMTQwMCIgaGVpZ2h0PSI3NjAiIGZpbGw9IiNmOGZhZmMiLz4KICA8dGV4dCB4PSI2MCIgeT0iNzIiIGZvbnQtZmFtaWx5PSJBcmlhbCwgSGVsdmV0aWNhLCBzYW5zLXNlcmlmIiBmb250LXNpemU9IjM2IiBmb250LXdlaWdodD0iNzAwIiBmaWxsPSIjMGYxNzJhIj5BIEJpb21uaS1JbnNwaXJlZCBMb2NhbCBEZXNpZ248L3RleHQ+CiAgPHRleHQgeD0iNjAiIHk9IjEwOCIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMjEiIGZpbGw9IiM0NzU1NjkiPlRyZWF0IHByb3ZlbmFuY2UgYXMgcGFydCBvZiB0aGUgZW5naW5lZXJpbmcgc3RhY2ssIHRoZW4gcGxhY2UgYmlvbG9naWNhbCB2YWxpZGF0aW9uIGFuZCBldmFsdWF0aW9uIGluIGEgc2VwYXJhdGUgYXNzdXJhbmNlIGxheWVyLjwvdGV4dD4KCiAgPHJlY3QgeD0iNjAiIHk9IjE1MCIgd2lkdGg9IjEyODAiIGhlaWdodD0iMzAwIiByeD0iMjgiIGZpbGw9IiNmZmZmZmYiIHN0cm9rZT0iI2NiZDVlMSIgc3Ryb2tlLXdpZHRoPSIyIi8+CiAgPHRleHQgeD0iOTUiIHk9IjE5NSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMjgiIGZvbnQtd2VpZ2h0PSI3MDAiIGZpbGw9IiMwZjE3MmEiPkxvY2FsIEVuZ2luZWVyaW5nIERlc2lnbjwvdGV4dD4KCiAgPHJlY3QgeD0iOTUiIHk9IjIzNSIgd2lkdGg9IjIwNSIgaGVpZ2h0PSIxMjAiIHJ4PSIxOCIgZmlsbD0iI2ZmZmZmZiIgc3Ryb2tlPSIjY2JkNWUxIiBzdHJva2Utd2lkdGg9IjIiLz4KICA8dGV4dCB4PSIxOTciIHk9IjI4MCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMjgiIGZvbnQtd2VpZ2h0PSI3MDAiIGZpbGw9IiMwZjE3MmEiPk9wZXJhdG9yPC90ZXh0PgogIDx0ZXh0IHg9IjE5NyIgeT0iMzEzIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBmb250LWZhbWlseT0iQXJpYWwsIEhlbHZldGljYSwgc2Fucy1zZXJpZiIgZm9udC1zaXplPSIxOCIgZmlsbD0iIzMzNDE1NSI+VlMgQ29kZSBhbmQgd29ya3NwYWNlPC90ZXh0PgoKICA8cmVjdCB4PSIzNDUiIHk9IjIzNSIgd2lkdGg9IjIwNSIgaGVpZ2h0PSIxMjAiIHJ4PSIxOCIgZmlsbD0iI2VjZmVmZiIgc3Ryb2tlPSIjNjdlOGY5IiBzdHJva2Utd2lkdGg9IjIiLz4KICA8dGV4dCB4PSI0NDciIHk9IjI4MCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMjgiIGZvbnQtd2VpZ2h0PSI3MDAiIGZpbGw9IiMwZjE3MmEiPkFnZW50PC90ZXh0PgogIDx0ZXh0IHg9IjQ0NyIgeT0iMzEzIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBmb250LWZhbWlseT0iQXJpYWwsIEhlbHZldGljYSwgc2Fucy1zZXJpZiIgZm9udC1zaXplPSIxOCIgZmlsbD0iIzMzNDE1NSI+cGxhbm5pbmcgYW5kIGNvZGU8L3RleHQ+CgogIDxyZWN0IHg9IjU5NSIgeT0iMjM1IiB3aWR0aD0iMjA1IiBoZWlnaHQ9IjEyMCIgcng9IjE4IiBmaWxsPSIjZWVmMmZmIiBzdHJva2U9IiM5M2M1ZmQiIHN0cm9rZS13aWR0aD0iMiIvPgogIDx0ZXh0IHg9IjY5NyIgeT0iMjgwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBmb250LWZhbWlseT0iQXJpYWwsIEhlbHZldGljYSwgc2Fucy1zZXJpZiIgZm9udC1zaXplPSIyOCIgZm9udC13ZWlnaHQ9IjcwMCIgZmlsbD0iIzBmMTcyYSI+VG9vbHM8L3RleHQ+CiAgPHRleHQgeD0iNjk3IiB5PSIzMTMiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGZvbnQtZmFtaWx5PSJBcmlhbCwgSGVsdmV0aWNhLCBzYW5zLXNlcmlmIiBmb250LXNpemU9IjE4IiBmaWxsPSIjMzM0MTU1Ij5NQ1AgYW5kIGRhdGFiYXNlczwvdGV4dD4KCiAgPHJlY3QgeD0iODQ1IiB5PSIyMzUiIHdpZHRoPSIyMDUiIGhlaWdodD0iMTIwIiByeD0iMTgiIGZpbGw9IiNmZmY3ZWQiIHN0cm9rZT0iI2ZkYmE3NCIgc3Ryb2tlLXdpZHRoPSIyIi8+CiAgPHRleHQgeD0iOTQ3IiB5PSIyODAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGZvbnQtZmFtaWx5PSJBcmlhbCwgSGVsdmV0aWNhLCBzYW5zLXNlcmlmIiBmb250LXNpemU9IjI4IiBmb250LXdlaWdodD0iNzAwIiBmaWxsPSIjMGYxNzJhIj5FeGVjdXRpb248L3RleHQ+CiAgPHRleHQgeD0iOTQ3IiB5PSIzMTMiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGZvbnQtZmFtaWx5PSJBcmlhbCwgSGVsdmV0aWNhLCBzYW5zLXNlcmlmIiBmb250LXNpemU9IjE4IiBmaWxsPSIjMzM0MTU1Ij5Eb2NrZXIgYW5kIGpvYnM8L3RleHQ+CgogIDxyZWN0IHg9IjEwOTUiIHk9IjIzNSIgd2lkdGg9IjIwNSIgaGVpZ2h0PSIxMjAiIHJ4PSIxOCIgZmlsbD0iI2YwZmRmNCIgc3Ryb2tlPSIjODZlZmFjIiBzdHJva2Utd2lkdGg9IjIiLz4KICA8dGV4dCB4PSIxMTk3IiB5PSIyODAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGZvbnQtZmFtaWx5PSJBcmlhbCwgSGVsdmV0aWNhLCBzYW5zLXNlcmlmIiBmb250LXNpemU9IjI4IiBmb250LXdlaWdodD0iNzAwIiBmaWxsPSIjMGYxNzJhIj5Qcm92ZW5hbmNlPC90ZXh0PgogIDx0ZXh0IHg9IjExOTciIHk9IjMxMyIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMTgiIGZpbGw9IiMzMzQxNTUiPm1hbmlmZXN0cyBhbmQgbGluZWFnZTwvdGV4dD4KCiAgPGxpbmUgeDE9IjMwMCIgeTE9IjI5NSIgeDI9IjM0NSIgeTI9IjI5NSIgc3Ryb2tlPSIjMzM0MTU1IiBzdHJva2Utd2lkdGg9IjUiLz4KICA8cG9seWdvbiBwb2ludHM9IjM0NSwyOTUgMzMxLDI4NyAzMzEsMzAzIiBmaWxsPSIjMzM0MTU1Ii8+CiAgPGxpbmUgeDE9IjU1MCIgeTE9IjI5NSIgeDI9IjU5NSIgeTI9IjI5NSIgc3Ryb2tlPSIjMzM0MTU1IiBzdHJva2Utd2lkdGg9IjUiLz4KICA8cG9seWdvbiBwb2ludHM9IjU5NSwyOTUgNTgxLDI4NyA1ODEsMzAzIiBmaWxsPSIjMzM0MTU1Ii8+CiAgPGxpbmUgeDE9IjgwMCIgeTE9IjI5NSIgeDI9Ijg0NSIgeTI9IjI5NSIgc3Ryb2tlPSIjMzM0MTU1IiBzdHJva2Utd2lkdGg9IjUiLz4KICA8cG9seWdvbiBwb2ludHM9Ijg0NSwyOTUgODMxLDI4NyA4MzEsMzAzIiBmaWxsPSIjMzM0MTU1Ii8+CiAgPGxpbmUgeDE9IjEwNTAiIHkxPSIyOTUiIHgyPSIxMDk1IiB5Mj0iMjk1IiBzdHJva2U9IiMzMzQxNTUiIHN0cm9rZS13aWR0aD0iNSIvPgogIDxwb2x5Z29uIHBvaW50cz0iMTA5NSwyOTUgMTA4MSwyODcgMTA4MSwzMDMiIGZpbGw9IiMzMzQxNTUiLz4KCiAgPHRleHQgeD0iOTUiIHk9IjQwMiIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMTkiIGZpbGw9IiM0NzU1NjkiPkVuZ2luZWVyaW5nIHF1ZXN0aW9uOiB3aGF0IHJhbiwgd2l0aCB3aGljaCB0b29scywgcGFyYW1ldGVycywgY29udGFpbmVycywgYW5kIGlucHV0cz88L3RleHQ+CgogIDxyZWN0IHg9IjYwIiB5PSI1MTAiIHdpZHRoPSIxMjgwIiBoZWlnaHQ9IjE5MCIgcng9IjI4IiBmaWxsPSIjZmZmZGY3IiBzdHJva2U9IiNmY2QzNGQiIHN0cm9rZS13aWR0aD0iMiIvPgogIDx0ZXh0IHg9Ijk1IiB5PSI1NTUiIGZvbnQtZmFtaWx5PSJBcmlhbCwgSGVsdmV0aWNhLCBzYW5zLXNlcmlmIiBmb250LXNpemU9IjI4IiBmb250LXdlaWdodD0iNzAwIiBmaWxsPSIjMGYxNzJhIj5TY2llbnRpZmljIEFzc3VyYW5jZTwvdGV4dD4KCiAgPHJlY3QgeD0iMTkwIiB5PSI1ODUiIHdpZHRoPSI0MjAiIGhlaWdodD0iODIiIHJ4PSIxOCIgZmlsbD0iI2ZmZmZmZiIgc3Ryb2tlPSIjY2JkNWUxIiBzdHJva2Utd2lkdGg9IjIiLz4KICA8dGV4dCB4PSI0MDAiIHk9IjYxOCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMjgiIGZvbnQtd2VpZ2h0PSI3MDAiIGZpbGw9IiMwZjE3MmEiPkJpb2xvZ2ljYWwgVmFsaWRhdGlvbjwvdGV4dD4KICA8dGV4dCB4PSI0MDAiIHk9IjY0OCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMTgiIGZpbGw9IiMzMzQxNTUiPkRvZXMgdGhlIHJlc3VsdCBtYWtlIGJpb2xvZ2ljYWwgc2Vuc2U/PC90ZXh0PgoKICA8cmVjdCB4PSI3OTAiIHk9IjU4NSIgd2lkdGg9IjQyMCIgaGVpZ2h0PSI4MiIgcng9IjE4IiBmaWxsPSIjZmZmZmZmIiBzdHJva2U9IiNjYmQ1ZTEiIHN0cm9rZS13aWR0aD0iMiIvPgogIDx0ZXh0IHg9IjEwMDAiIHk9IjYxOCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMjgiIGZvbnQtd2VpZ2h0PSI3MDAiIGZpbGw9IiMwZjE3MmEiPkV2YWx1YXRpb248L3RleHQ+CiAgPHRleHQgeD0iMTAwMCIgeT0iNjQ4IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBmb250LWZhbWlseT0iQXJpYWwsIEhlbHZldGljYSwgc2Fucy1zZXJpZiIgZm9udC1zaXplPSIxOCIgZmlsbD0iIzMzNDE1NSI+SG93IHJlbGlhYmxlIGlzIHRoZSBzeXN0ZW0gYWNyb3NzIHRhc2tzPzwvdGV4dD4KCiAgPGxpbmUgeDE9IjExOTciIHkxPSIzNTUiIHgyPSIxMTk3IiB5Mj0iNTEwIiBzdHJva2U9IiMzMzQxNTUiIHN0cm9rZS13aWR0aD0iNSIvPgogIDxwb2x5Z29uIHBvaW50cz0iMTE5Nyw1MTAgMTE4OSw0OTYgMTIwNSw0OTYiIGZpbGw9IiMzMzQxNTUiLz4KICA8bGluZSB4MT0iMTE5NyIgeTE9IjUxMCIgeDI9IjQwMCIgeTI9IjUxMCIgc3Ryb2tlPSIjMzM0MTU1IiBzdHJva2Utd2lkdGg9IjUiLz4KICA8cG9seWdvbiBwb2ludHM9IjQwMCw1MTAgNDE0LDUwMiA0MTQsNTE4IiBmaWxsPSIjMzM0MTU1Ii8+CiAgPGxpbmUgeDE9IjExOTciIHkxPSI1MTAiIHgyPSIxMDAwIiB5Mj0iNTEwIiBzdHJva2U9IiMzMzQxNTUiIHN0cm9rZS13aWR0aD0iNSIvPgogIDxwb2x5Z29uIHBvaW50cz0iMTAwMCw1MTAgMTAxNCw1MDIgMTAxNCw1MTgiIGZpbGw9IiMzMzQxNTUiLz4KCiAgPHRleHQgeD0iOTUiIHk9IjY4OCIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMTkiIGZpbGw9IiM0NzU1NjkiPkFzc3VyYW5jZSBxdWVzdGlvbnM6IHNob3VsZCBhIHNjaWVudGlzdCB0cnVzdCB0aGUgb3V0cHV0LCBhbmQgaG93IG9mdGVuIGRvZXMgdGhlIHN5c3RlbSBiZWhhdmUgd2VsbD88L3RleHQ+Cjwvc3ZnPgo=)

*Figure: a Biomni-inspired local architecture with five engineering parts and a separate scientific assurance layer. Provenance lives inside the engineering stack; biological validation and evaluation sit above it.*

This is already enough to build a serious local platform.

Above the local design sits a separate scientific assurance layer with two parts:

1. Biological validation: does the result make biological sense?
2. Evaluation: how reliable is the system across tasks?

---

## What Provenance Looks Like in Practice

When people describe a system like Biomni, it is easy to make the infrastructure sound magical. It is not.

A serious local stack can record, for every run:

- tool version,
- database version,
- container image or digest,
- parameter set,
- timestamp,
- source dataset identifiers or checksums,
- generated code,
- commands executed,
- output artifact locations.

This is standard workflow practice.

If Nextflow can do it, a local biomedical agent platform can do it too. The same is true for other workflow systems, experiment trackers, or even a simple run registry backed by JSON, SQLite, or Postgres.

This layer is important, but it is not the intellectual bottleneck.

---

## A Concrete Run Manifest

For example, I would want every agent run to emit a manifest that looks roughly like this:

```yaml
run_id: 2026-04-30T18-42-11Z_variant-prioritization_001
task_type: variant_prioritization
user_question: "Prioritize likely causal genes for this phenotype and VCF"
timestamp_utc: 2026-04-30T18:42:11Z

agent:
  model: claude-sonnet-4
  prompt_version: prompts/variant_triage/v3.md
  planner_commit: a1b2c3d

environment:
  container_image: biomni-local-python@sha256:abc123
  python_version: 3.11.11
  r_version: 4.4.1
  os: linux-amd64

datasets:
  - path: data/cases/case_017/input.vcf.gz
    sha256: 52df...
  - path: data/references/hpo_terms.tsv
    sha256: d901...

references:
  clinvar_release: 2026-04-15
  gnomad_release: 4.1
  ensembl_release: 114

tools_used:
  - name: clinvar_lookup
    version: 1.3.2
  - name: ensembl_variant_annotator
    version: 2.4.0
  - name: custom_variant_ranker
    version: 0.8.1

parameters:
  max_af_threshold: 0.001
  inheritance_model: recessive
  phenotype_terms:
    - HP:0001250
    - HP:0001290

artifacts:
  generated_code: runs/2026-04-30_001/code/analysis.py
  ranked_table: runs/2026-04-30_001/results/ranked_genes.tsv
  report: runs/2026-04-30_001/report.md
  execution_log: runs/2026-04-30_001/logs/execution.log
```

That is not difficult in principle. It is disciplined engineering.

It gives you a precise answer to the provenance question:

> What happened during execution?

It does not tell you whether the biological conclusion was strong.

---

## A Concrete Example: Single-Cell Analysis

To make the architecture concrete, imagine a scientist asking:

> Compare treated and control samples in this scRNA-seq dataset and identify the main cell-state shifts.

In the 5 + 2 design, that request moves through the system in a structured way.

### The Five Engineering Parts

#### Operator

- The scientist works in VS Code and can inspect input files, prior runs, notebooks, QC plots, and generated reports.
- The workspace already has fixed locations for raw matrices, metadata, references, intermediate artifacts, and final outputs.

#### Agent

- The agent turns the request into a plan: inspect metadata, run QC, normalize, cluster, annotate, compare treated vs control, and summarize the result.
- It decides when to write code, when to call a tool, and when to pause for review.

#### Tools

- MCP-exposed tools provide reference atlases, marker databases, ontology lookup, notebook execution, and file access.
- Deterministic wrappers handle recurring steps such as cell filtering, differential expression, marker lookup, and enrichment.

#### Execution

- The workflow runs inside a pinned container with the expected Python or R single-cell stack.
- The job runner captures code, logs, warnings, plots, and intermediate outputs.

#### Provenance

- The run manifest records dataset identifiers, container digests, package versions, atlas versions, thresholds, generated code, and output paths.
- If the scientist wants to rerun the analysis later, the trace is already there.

### The Two Scientific Assurance Parts

#### Biological validation

- Check whether the claimed cell types are supported by marker specificity rather than one noisy gene.
- Check whether the apparent treatment effect could instead be a batch effect, low-quality cells, or doublets.
- Check whether the selected atlas is appropriate for the tissue and species.

#### Evaluation in this example

- Measure whether the agent consistently chooses the right workflow for similar single-cell tasks.
- Score whether it recovers expected markers or known perturbation signatures on benchmark datasets.
- Review whether its final writeup stays calibrated instead of overstating weak signals.

That example shows why the split matters. The engineering stack can tell you exactly what ran. The assurance layer tells you whether the analysis should be trusted.

---

## The Scientific Assurance Layer

The engineering design answers one class of question:

> What ran, with which tools, parameters, containers, and inputs?

The scientific assurance layer answers a different class:

> Should a scientist trust the output, and how often does the system behave well?

That is why I would keep biological validation and evaluation out of the engineering stack even though they interact with it closely.

---

## Biological Validation

Biological validation asks a different question:

> Even if the pipeline ran correctly, did it make biological sense?

This is where the difficulty changes category.

An agent can execute perfectly and still be wrong in ways that are scientifically important.

For example, a system can produce a polished single-cell analysis while:

- using an inappropriate reference atlas,
- confusing batch effect with biology,
- over-interpreting weak marker genes,
- failing to remove doublets,
- making cell-state claims stronger than the data supports.

Nothing in a container digest or execution manifest will detect that automatically.

That is not a workflow-engineering failure. That is a domain-validation failure.

---

## A Concrete Validation Design

For a local biomedical agent, I would not let the model move directly from execution to final answer. I would insert a biological validation layer with explicit checks.

### Preflight Checks

Before the main workflow runs:

- validate file format and schema,
- verify database and reference versions are present,
- confirm the task matches the modality,
- reject incompatible combinations such as human references on mouse data,
- ensure required metadata exists.

### Workflow-Specific Guardrails

During execution, add checks that depend on the analysis type.

For computational biology examples:

- scRNA-seq: cell count thresholds, mitochondrial fraction warnings, doublet flags, marker specificity checks, batch-confound warnings,
- variant prioritization: population frequency sanity checks, ClinVar conflict flags, inheritance-model consistency checks, transcript-selection warnings,
- enrichment analysis: background-set validation, multiple-testing checks, low-count warnings, database release checks.

### Post-Execution Review Gates

After the workflow runs:

- force the agent to summarize assumptions,
- require citation of the databases and tool outputs actually used,
- compare major claims against predefined red-flag rules,
- trigger human review if the confidence is low or if guardrails fired.

That design does not make the system biologically correct by default, but it does make failure modes more visible and more manageable.

---

## Evaluation

Evaluation asks yet another question:

> How often does the system produce results that are scientifically defensible across realistic tasks?

That sounds simple until you try to build the benchmark.

In software engineering, many tasks have crisp oracles. A test passes or it fails. In biomedical research, the answer is often conditional, partial, context-dependent, or still uncertain in the literature.

That means evaluation is hard because the oracle is hard.

A biomedical agent may need to be judged on things like:

- whether it chose an appropriate workflow,
- whether it used the right databases,
- whether it handled confounders sensibly,
- whether its interpretation was calibrated,
- whether an expert would consider the output actionable.

Those are much more difficult to score than simple execution success.

---

## A Concrete Evaluation Design

If I were building this locally, I would create a benchmark harness with three kinds of scoring.

### 1. Execution Scoring

- Did the workflow complete?
- Did the required tools run?
- Were the expected artifacts generated?
- Is the run reproducible on rerun?

### 2. Process Scoring

- Did the agent choose an appropriate workflow?
- Did it use the right references and databases?
- Did it avoid invalid shortcuts?
- Did it explain uncertainty rather than overstate confidence?

### 3. Scientific Outcome Scoring

- Did the result match accepted answers on benchmark cases?
- Did expert reviewers judge the output as usable?
- Did the agent recover known markers, genes, pathways, or diagnoses when it should have?
- Did it avoid biologically misleading conclusions even when the output looked polished?

For many tasks, the best evaluation object is not a single exact answer. It is a rubric.

That is why biomedical evaluation is hard: the oracle is often conditional, partial, and domain-dependent.

---

## A Useful Distinction

I find it helpful to separate the whole problem into three questions.

| Layer | Question | Difficulty Type |
| --- | --- | --- |
| Provenance | What happened? | Workflow engineering |
| Biological validation | Was the result scientifically sensible? | Scientific judgment |
| Evaluation | How reliably does the system produce defensible results? | Scientific assurance with engineering and domain components |

The first is largely tractable with mature engineering patterns.

The second and third require domain-specific assumptions, expert calibration, and carefully designed task suites.

---

## What Still Needs Explicit Design

Where these systems become difficult is not the local-vs-cloud choice. It is the validation design.

For a credible biomedical agent, I would want at least three explicit layers.

### 1. Provenance Layer

This should record:

- software and tool versions,
- database and reference versions,
- container identifiers,
- prompts and tool calls,
- generated code,
- parameters,
- input and output artifact identities.

This is the most straightforward part of the stack.

### 2. Biological Guardrail Layer

This should encode domain-specific checks.

Examples:

- scRNA-seq QC threshold checks,
- marker specificity warnings,
- multiple-testing checks for enrichment,
- inheritance-model consistency checks for rare disease analysis.

This is not generic AI safety. It is domain logic.

### 3. Evaluation Layer

This should test the agent on realistic tasks with defensible scoring.

Examples:

- benchmark prompts with known expected behaviors,
- tool-choice scoring,
- expert rubric review,
- rerun reproducibility checks,
- factual support checks against trusted references.

This is where many impressive demos start to become uncomfortable, because the benchmark itself requires real scientific thought.

---

## How I Think About Biomni-Like Systems

Publicly described systems like Biomni appear to take the environment problem seriously: they package tools, databases, a large data layer, code execution, and model-driven planning into a coherent biomedical stack.

That is already much better than a generic chat interface.

But even a very strong biomedical agent still faces the same underlying truth:

> execution success is not scientific correctness.

The system can reduce error by using tools, curated resources, and structured workflows. It cannot make the scientific validation problem disappear.

---

## Final Takeaway

If you are building a Biomni-like app locally, do not be intimidated by the provenance problem. Engineers already know how to solve that.

The real challenge is deciding whether the system's answer is biologically meaningful and whether it stays reliable across tasks.

That is why the deepest question is not:

> Can the agent run the workflow?

It is:

> Can the agent produce a result that a serious scientist would trust enough to use?

That is the boundary between an impressive execution engine and a trustworthy research assistant.
