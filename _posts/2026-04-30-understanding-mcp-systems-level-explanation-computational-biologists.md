---
layout: post
title: "Understanding MCP: A Systems-Level Explanation for Computational Biologists"
date: 2026-04-30
categories: [AI Systems]
tags: [mcp, model-context-protocol, llm, client-server, computational-biology, systems-design]
---

I spent longer than I should have trying to understand Model Context Protocol (MCP) because many explanations started from AI hype instead of system structure.

What finally made it click for me was ignoring the hype and looking at it the same way I would look at any other engineering boundary: client, server, contract, execution.

If you come from computational biology, data engineering, workflow systems, or scientific computing, MCP is not introducing an unfamiliar architecture. It is introducing a familiar one with a new kind of client.

That client happens to be an LLM.

Once I saw it that way, the whole thing became much easier to reason about.

> **Key takeaways**
>
> - MCP is client-server architecture, not AI magic.
> - The LLM does not discover servers. It only sees the tools the runtime gives it.
> - The safest default is still deterministic server-side code.
> - MCP is most useful when it cleanly separates deciding from doing.

---

## The Short Version

MCP is a client-server protocol that defines how an LLM can call tools exposed by a server.

The architecture is not new. The main change is that the client is no longer a browser, CLI, backend service, or workflow engine. The client is a language model running inside an application or agent runtime.

That is the core idea.

---

## Why This Makes Sense to Computational Biologists

Computational biology already depends on clean boundaries between interpretation and execution.

We build systems where:

- workflows orchestrate many deterministic steps,
- shared services expose stable functionality,
- results need to be auditable and reproducible,
- sensitive data access has to be controlled,
- probabilistic interpretation sits on top of deterministic infrastructure.

MCP fits naturally into that worldview.

It gives you a structured contract for letting a probabilistic component decide which deterministic operation to invoke, without collapsing the whole system into unstructured prompting.

---

## The Core Insight

MCP defines how a client talks to a server.

In practice, that means you move deterministic functions out of the client and expose them on the server side through an explicit tool interface.

The LLM does not become the system. It becomes a decision-making layer that sits in front of the system.

That distinction matters.

---

## MCP in One Diagram

![MCP architecture diagram showing a user request flowing through an application runtime to an LLM client, then through MCP to a server that executes deterministic systems](data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIxMjAwIiBoZWlnaHQ9IjcyMCIgdmlld0JveD0iMCAwIDEyMDAgNzIwIiByb2xlPSJpbWciIGFyaWEtbGFiZWxsZWRieT0idGl0bGUgZGVzYyI+CiAgPHRpdGxlIGlkPSJ0aXRsZSI+TUNQIGNsaWVudCBzZXJ2ZXIgb3ZlcnZpZXc8L3RpdGxlPgogIDxkZXNjIGlkPSJkZXNjIj5BIHN5c3RlbXMgZGlhZ3JhbSBzaG93aW5nIFVzZXIsIEFwcGxpY2F0aW9uIG9yIEFnZW50IFJ1bnRpbWUsIExMTSBjbGllbnQsIE1DUCBTZXJ2ZXIsIGFuZCBEZXRlcm1pbmlzdGljIFN5c3RlbXMgY29ubmVjdGVkIGluIHNlcXVlbmNlLCB3aXRoIHRoZSBNQ1AgcHJvdG9jb2wgbGFiZWwgYmV0d2VlbiB0aGUgTExNIGNsaWVudCBhbmQgTUNQIFNlcnZlci48L2Rlc2M+CiAgPGRlZnM+CiAgICA8bGluZWFyR3JhZGllbnQgaWQ9ImJnIiB4MT0iMCIgeTE9IjAiIHgyPSIxIiB5Mj0iMSI+CiAgICAgIDxzdG9wIG9mZnNldD0iMCUiIHN0b3AtY29sb3I9IiNmOGZhZmMiLz4KICAgICAgPHN0b3Agb2Zmc2V0PSIxMDAlIiBzdG9wLWNvbG9yPSIjZWVmMmZmIi8+CiAgICA8L2xpbmVhckdyYWRpZW50PgogICAgPGxpbmVhckdyYWRpZW50IGlkPSJjYXJkIiB4MT0iMCIgeTE9IjAiIHgyPSIxIiB5Mj0iMSI+CiAgICAgIDxzdG9wIG9mZnNldD0iMCUiIHN0b3AtY29sb3I9IiNmZmZmZmYiLz4KICAgICAgPHN0b3Agb2Zmc2V0PSIxMDAlIiBzdG9wLWNvbG9yPSIjZjhmYWZjIi8+CiAgICA8L2xpbmVhckdyYWRpZW50PgogICAgPG1hcmtlciBpZD0iYXJyb3ciIHZpZXdCb3g9IjAgMCAxMCAxMCIgcmVmWD0iOCIgcmVmWT0iNSIgbWFya2VyV2lkdGg9IjEwIiBtYXJrZXJIZWlnaHQ9IjEwIiBvcmllbnQ9ImF1dG8tc3RhcnQtcmV2ZXJzZSI+CiAgICAgIDxwYXRoIGQ9Ik0gMCAwIEwgMTAgNSBMIDAgMTAgeiIgZmlsbD0iIzMzNDE1NSIvPgogICAgPC9tYXJrZXI+CiAgICA8ZmlsdGVyIGlkPSJzaGFkb3ciIHg9Ii0xMCUiIHk9Ii0xMCUiIHdpZHRoPSIxMjAlIiBoZWlnaHQ9IjEyMCUiPgogICAgICA8ZmVEcm9wU2hhZG93IGR4PSIwIiBkeT0iOCIgc3RkRGV2aWF0aW9uPSIxMCIgZmxvb2QtY29sb3I9IiMwZjE3MmEiIGZsb29kLW9wYWNpdHk9IjAuMTIiLz4KICAgIDwvZmlsdGVyPgogIDwvZGVmcz4KCiAgPHJlY3Qgd2lkdGg9IjEyMDAiIGhlaWdodD0iNzIwIiBmaWxsPSJ1cmwoI2JnKSIvPgogIDx0ZXh0IHg9IjgwIiB5PSI5MCIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMzYiIGZvbnQtd2VpZ2h0PSI3MDAiIGZpbGw9IiMwZjE3MmEiPlVuZGVyc3RhbmRpbmcgTUNQIGFzIENsaWVudC1TZXJ2ZXIgQXJjaGl0ZWN0dXJlPC90ZXh0PgogIDx0ZXh0IHg9IjgwIiB5PSIxMzAiIGZvbnQtZmFtaWx5PSJBcmlhbCwgSGVsdmV0aWNhLCBzYW5zLXNlcmlmIiBmb250LXNpemU9IjIwIiBmaWxsPSIjMzM0MTU1Ij5UaGUgbW9kZWwgaXMgYSBjbGllbnQgaW4gZnJvbnQgb2YgZGV0ZXJtaW5pc3RpYyBzeXN0ZW1zLiBNQ1AgaXMgdGhlIGNvbnRyYWN0IGJldHdlZW4gdGhlbS48L3RleHQ+CgogIDxyZWN0IHg9IjgwIiB5PSIyMjAiIHJ4PSIyNCIgcnk9IjI0IiB3aWR0aD0iMTgwIiBoZWlnaHQ9IjExMCIgZmlsbD0idXJsKCNjYXJkKSIgc3Ryb2tlPSIjY2JkNWUxIiBzdHJva2Utd2lkdGg9IjIiIGZpbHRlcj0idXJsKCNzaGFkb3cpIi8+CiAgPHRleHQgeD0iMTcwIiB5PSIyNzUiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGZvbnQtZmFtaWx5PSJBcmlhbCwgSGVsdmV0aWNhLCBzYW5zLXNlcmlmIiBmb250LXNpemU9IjI4IiBmb250LXdlaWdodD0iNzAwIiBmaWxsPSIjMGYxNzJhIj5Vc2VyPC90ZXh0PgoKICA8cmVjdCB4PSIzMzAiIHk9IjIyMCIgcng9IjI0IiByeT0iMjQiIHdpZHRoPSIyNzAiIGhlaWdodD0iMTEwIiBmaWxsPSJ1cmwoI2NhcmQpIiBzdHJva2U9IiNjYmQ1ZTEiIHN0cm9rZS13aWR0aD0iMiIgZmlsdGVyPSJ1cmwoI3NoYWRvdykiLz4KICA8dGV4dCB4PSI0NjUiIHk9IjI2MiIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMjYiIGZvbnQtd2VpZ2h0PSI3MDAiIGZpbGw9IiMwZjE3MmEiPkFwcGxpY2F0aW9uIC88L3RleHQ+CiAgPHRleHQgeD0iNDY1IiB5PSIyOTYiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGZvbnQtZmFtaWx5PSJBcmlhbCwgSGVsdmV0aWNhLCBzYW5zLXNlcmlmIiBmb250LXNpemU9IjI2IiBmb250LXdlaWdodD0iNzAwIiBmaWxsPSIjMGYxNzJhIj5BZ2VudCBSdW50aW1lPC90ZXh0PgoKICA8cmVjdCB4PSI2NzAiIHk9IjIyMCIgcng9IjI0IiByeT0iMjQiIHdpZHRoPSIxOTAiIGhlaWdodD0iMTEwIiBmaWxsPSIjZWNmZWZmIiBzdHJva2U9IiM2N2U4ZjkiIHN0cm9rZS13aWR0aD0iMiIgZmlsdGVyPSJ1cmwoI3NoYWRvdykiLz4KICA8dGV4dCB4PSI3NjUiIHk9IjI2MiIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMjQiIGZvbnQtd2VpZ2h0PSI3MDAiIGZpbGw9IiMwZjE3MmEiPkxMTTwvdGV4dD4KICA8dGV4dCB4PSI3NjUiIHk9IjI5NiIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMjQiIGZvbnQtd2VpZ2h0PSI3MDAiIGZpbGw9IiMwZjE3MmEiPkNsaWVudDwvdGV4dD4KCiAgPHJlY3QgeD0iOTMwIiB5PSIyMjAiIHJ4PSIyNCIgcnk9IjI0IiB3aWR0aD0iMTkwIiBoZWlnaHQ9IjExMCIgZmlsbD0iI2VmZjZmZiIgc3Ryb2tlPSIjOTNjNWZkIiBzdHJva2Utd2lkdGg9IjIiIGZpbHRlcj0idXJsKCNzaGFkb3cpIi8+CiAgPHRleHQgeD0iMTAyNSIgeT0iMjYyIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBmb250LWZhbWlseT0iQXJpYWwsIEhlbHZldGljYSwgc2Fucy1zZXJpZiIgZm9udC1zaXplPSIyNCIgZm9udC13ZWlnaHQ9IjcwMCIgZmlsbD0iIzBmMTcyYSI+TUNQPC90ZXh0PgogIDx0ZXh0IHg9IjEwMjUiIHk9IjI5NiIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMjQiIGZvbnQtd2VpZ2h0PSI3MDAiIGZpbGw9IiMwZjE3MmEiPlNlcnZlcjwvdGV4dD4KCiAgPHJlY3QgeD0iMzMwIiB5PSI0NTAiIHJ4PSIyNCIgcnk9IjI0IiB3aWR0aD0iNzkwIiBoZWlnaHQ9IjE1MCIgZmlsbD0iI2ZmZjdlZCIgc3Ryb2tlPSIjZmRiYTc0IiBzdHJva2Utd2lkdGg9IjIiIGZpbHRlcj0idXJsKCNzaGFkb3cpIi8+CiAgPHRleHQgeD0iNzI1IiB5PSI1MDAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGZvbnQtZmFtaWx5PSJBcmlhbCwgSGVsdmV0aWNhLCBzYW5zLXNlcmlmIiBmb250LXNpemU9IjI4IiBmb250LXdlaWdodD0iNzAwIiBmaWxsPSIjN2MyZDEyIj5EZXRlcm1pbmlzdGljIFN5c3RlbXM8L3RleHQ+CiAgPHRleHQgeD0iNzI1IiB5PSI1NDUiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGZvbnQtZmFtaWx5PSJBcmlhbCwgSGVsdmV0aWNhLCBzYW5zLXNlcmlmIiBmb250LXNpemU9IjIyIiBmaWxsPSIjN2MyZDEyIj5GaWxlcywgZGF0YWJhc2VzLCBBUElzLCBwb2xpY2llcywgUUMgY2hlY2tzLCBidXNpbmVzcyBsb2dpYzwvdGV4dD4KCiAgPGxpbmUgeDE9IjI2MCIgeTE9IjI3NSIgeDI9IjMzMCIgeTI9IjI3NSIgc3Ryb2tlPSIjMzM0MTU1IiBzdHJva2Utd2lkdGg9IjUiIG1hcmtlci1lbmQ9InVybCgjYXJyb3cpIi8+CiAgPGxpbmUgeDE9IjYwMCIgeTE9IjI3NSIgeDI9IjY3MCIgeTI9IjI3NSIgc3Ryb2tlPSIjMzM0MTU1IiBzdHJva2Utd2lkdGg9IjUiIG1hcmtlci1lbmQ9InVybCgjYXJyb3cpIi8+CiAgPGxpbmUgeDE9Ijg2MCIgeTE9IjI3NSIgeDI9IjkzMCIgeTI9IjI3NSIgc3Ryb2tlPSIjMzM0MTU1IiBzdHJva2Utd2lkdGg9IjUiIG1hcmtlci1lbmQ9InVybCgjYXJyb3cpIi8+CiAgPGxpbmUgeDE9IjEwMjUiIHkxPSIzMzAiIHgyPSIxMDI1IiB5Mj0iNDUwIiBzdHJva2U9IiMzMzQxNTUiIHN0cm9rZS13aWR0aD0iNSIgbWFya2VyLWVuZD0idXJsKCNhcnJvdykiLz4KCiAgPHJlY3QgeD0iNzYwIiB5PSIxNDAiIHJ4PSIxNCIgcnk9IjE0IiB3aWR0aD0iMjY1IiBoZWlnaHQ9IjQ2IiBmaWxsPSIjZGJlYWZlIiBzdHJva2U9IiM5M2M1ZmQiIHN0cm9rZS13aWR0aD0iMS41Ii8+CiAgPHRleHQgeD0iODkyIiB5PSIxNzAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGZvbnQtZmFtaWx5PSJBcmlhbCwgSGVsdmV0aWNhLCBzYW5zLXNlcmlmIiBmb250LXNpemU9IjE4IiBmb250LXdlaWdodD0iNzAwIiBmaWxsPSIjMWQ0ZWQ4Ij5NQ1AgcHJvdG9jb2w6IHN0cnVjdHVyZWQgdG9vbCBjYWxsczwvdGV4dD4KCiAgPHRleHQgeD0iODAiIHk9IjY1NSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMjAiIGZpbGw9IiM0NzU1NjkiPk1lbnRhbCBtb2RlbDogaHVtYW5zIGNob29zZSBzZXJ2ZXJzLCB0aGUgTExNIGNob29zZXMgdG9vbHMsIGFuZCB0aGUgc2VydmVyIGV4ZWN1dGVzIHRoZSBsb2dpYy48L3RleHQ+CiAgPHRleHQgeD0iODAiIHk9IjY4NSIgZm9udC1mYW1pbHk9IkFyaWFsLCBIZWx2ZXRpY2EsIHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iMTYiIGZpbGw9IiM2NDc0OGIiPlRoaXMgaXMgd2h5IE1DUCBmZWVscyBmYW1pbGlhciB0byBhbnlvbmUgd2hvIGFscmVhZHkgdW5kZXJzdGFuZHMgQVBJcywgUlBDLCBvciB3b3JrZmxvdyBvcmNoZXN0cmF0aW9uLjwvdGV4dD4KPC9zdmc+)

*Figure: the model is a client in front of deterministic systems, not a replacement for them.*

The most useful sentence here is simple:

> The LLM is not the system. It is a client of the system.

---

## What the LLM Actually Sees

One of the most common misunderstandings is that the model somehow discovers MCP servers on its own.

That is not how the system works.

What actually happens is much more conventional:

- humans or developers decide which MCP servers are available,
- the runtime connects to those servers and retrieves tool schemas,
- the LLM receives only the tool definitions it is explicitly given,
- the LLM decides which tool to call and with what arguments.

So the model does not scan the network, discover endpoints, or manage authentication. It operates inside the boundary that the runtime and the platform define for it.

That is a feature, not a limitation. It keeps governance and security where they belong.

---

## Where LLMs Can Sit in an MCP Architecture

There are several ways to place LLMs in an MCP-based system.

### 1. LLM as the MCP Client

This is the default pattern.

The model interprets user intent, selects tools, fills arguments, and then interprets returned results.

In this setup, the LLM acts as a planner or orchestrator. MCP is simply the communication contract that lets it invoke server-side capabilities.

### 2. MCP Server with No LLM Inside It

This is usually the cleanest default.

The server is just deterministic code: Python, SQL, Java, Rust, whatever you trust for production operations.

This is ideal for:

- data access,
- file operations,
- quality-control checks,
- policy enforcement,
- reproducible transformations.

For most scientific systems, this should be the starting point.

It is also the version I would recommend to most research and platform teams first, because it preserves the same testability and operational clarity we already expect from normal services.

### 3. LLM Inside the MCP Server

This can be useful, but it should be deliberate.

Examples include summarization, ranking, classification, or explanation services exposed as tools.

The important discipline is to treat that model output as something that must be bounded, validated, and monitored rather than blindly trusted.

### 4. LLM on Both Sides

This is the most advanced pattern.

You might see it in agent-to-agent systems, tool recommendation layers, or meta-reasoning stacks.

It can be powerful, but it is also the hardest version to debug because uncertainty now exists in multiple layers of the interaction.

---

## Why MCP Feels Familiar

It should feel familiar because the pattern has deep historical precedent.

| Era | Technology | Same Underlying Idea |
| --- | --- | --- |
| 1980s | RPC | Remote function calls |
| 1990s | CORBA / DCOM | Interface-defined services |
| 2000s | SOAP / WSDL | Contract-first APIs |
| 2000s | REST | Client-server separation |
| 2010s | gRPC / Thrift | Typed RPC |
| 2010s | Microservices | Service boundaries |
| 2020s | MCP | LLM as client |

The protocol is not rewriting distributed systems. It is adapting a familiar systems pattern to a new type of caller.

One concise way to say it is this:

> MCP is RPC for model-driven tool use.

---

## What MCP Does Not Do

MCP does not do the reasoning for you.

It does not guarantee correctness. It does not replace backend services. It does not eliminate system design. It does not remove the need for governance, authorization, validation, or observability.

What it does provide is a standard way to describe tools, call them, and return results.

That is enough to matter.

Standards are powerful precisely because they solve the boring, repeated integration problem once.

---

## Why a Protocol Helps Instead of Just Calling Python Functions

It is reasonable to ask why this needs a protocol at all.

After all, if you are building a single local application, you can wire model calls directly to Python functions and move on.

The problem appears when the system gets bigger.

At that point, someone has to own:

- tool description,
- versioning,
- authorization,
- safety boundaries,
- multi-client compatibility,
- server lifecycle,
- logging and auditability.

If you do not formalize that layer, every application ends up reinventing a slightly different private tool-calling stack.

MCP exists to prevent that fragmentation.

That part also made the architecture easier for me to respect. Without a protocol, it is very easy to confuse a convenient local demo with something that can actually scale across teams, tools, and environments.

---

## Why This Matters in Scientific and Computational Settings

For computational biology, the value is not that an LLM can call a tool. The value is that you can separate thinking from execution without losing control of the execution layer.

That gives you practical benefits:

- deterministic scientific logic can remain deterministic,
- access to sensitive datasets can stay centralized,
- the same tools can be reused across multiple applications,
- audit trails become easier to preserve,
- probabilistic interpretation does not have to own operational authority.

This is very close to how we already think about workflow systems, analysis services, and platform engineering in research environments.

---

## The Most Useful Mental Model

If I had to compress the entire concept into one sentence, it would be this:

> Humans choose servers. LLMs choose tools. Servers execute logic.

That single sentence removes a surprising amount of confusion.

It clarifies where governance lives, where determinism lives, and where uncertainty lives.

---

## The Main Anti-Pattern to Avoid

The most obvious failure mode is trying to put too much logic inside the model itself.

In scientific computing terms, that is equivalent to burying quality-control rules in comments, embedding business logic in shell history, or treating a notebook narrative as if it were a production control plane.

It feels fast at first. It becomes brittle very quickly.

MCP is useful precisely because it creates a clean seam: the model can decide what to do next, but the actual operations remain explicit, inspectable, and testable.

---

## Final Takeaway

MCP is not mysterious.

It is client-server architecture with explicit tool contracts and an LLM acting as the client.

If you already understand APIs, RPC, pipelines, workflow engines, or service boundaries, you already understand most of the important part. The novelty is not the architecture. The novelty is the kind of software component making the call.

That framing matters because it turns MCP from something vague and trendy into something concrete and engineerable.

And once it becomes engineerable, it becomes much easier to use well.
