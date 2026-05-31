---
layout: post
title: "AI-Assisted Development Is Harness Engineering"
date: 2026-05-31
categories: [Software Engineering]
tags: [claude-code, ai-coding, mcp, agents, workflow]
---

The most useful thing I took from Addy Osmani's *Mastering AI-Assisted Development* is a name for what the course is mostly teaching: **harness engineering**. The model is increasingly a commodity input. The artifact that holds the value — the thing that compounds across projects and across model upgrades — is the scaffolding around the model.

That reframing is what makes the eighteen demos cohere. Each one adds a different layer to the same rig.

---

## The Load-Bearing Argument

The course's quiet thesis, restated:

> The quality of AI-assisted development is not bounded only by the model's raw capability. It is bounded by how well you architect the *information environment* — what the agent can access, where its working state lives, and how clearly you specify what "done" looks like.

Read in sequence, the chapters trace a progression from *vibe coder* to *conductor* to *orchestrator* to *systems designer*. The patterns are additive. A tight `CLAUDE.md` from Chapter 1 becomes the style guide every subagent inherits in Chapter 5. A custom MCP server from Chapter 3 gives a swarm of agents access to your team's runtime state. The multi-phase plan from Chapter 4 is how a team-lead agent decomposes work and dispatches it to peers.

The harness compounds. The model gets swapped out next quarter.

---

## The Two-File Discipline

The single most transferable pattern from Chapter 1 is the split between two files:

- **`CLAUDE.md`** — the project constitution. Read on every invocation. Answers *"how do we work here?"* — tech stack, file layout, build/test commands, things never to do.
- **`SPEC.md`** — the feature blueprint. Pulled in only when you're working on this feature. Answers *"what are we building right now?"* — requirements, constraints, acceptance criteria for one task.

The check, in one line: *would this still be true six months from now, after this feature ships?* If yes, it belongs in `CLAUDE.md`. If no, it belongs in `SPEC.md`.

Two failure modes when you collapse the split:

1. **Standards leak into task notes.** Conventions get buried inside last month's feature spec. New work has no idea they apply.
2. **Task constraints harden into law.** "Maintain 60 FPS with 50 animals" lives in the constitution forever, and six months later the agent is still optimizing for instanced GPU rendering on a 2D dashboard.

Same project, two altitudes. Don't confuse them.

---

## The Harness Layers

Each chapter is one more layer of the rig:

| Layer | What it harnesses | Mechanism |
|-------|-------------------|-----------|
| Instruction | What the agent is told, every invocation | `CLAUDE.md`, `SPEC.md` |
| Knowledge | What the agent knows about your domain | Skills |
| I/O | What the agent can see and touch | MCP servers |
| Safety | What the agent cannot do | `permissions.deny`, hooks |
| Workflow | Repeatable multi-step procedures | Slash commands |
| State | Where memory lives when context overflows | RALPH loops (Read -> Act -> Log -> Plan -> Halt), native task lists, multi-phase plans |
| Concurrency | How many agents work without colliding | Subagents, swarms, teams |
| Coordination | The contracts that prevent integration breakage | Shared types, ownership boundaries |
| Quality | What gets verified before merge | `/verify`, `/review`, `/ux-audit` |
| Feedback | How the harness improves itself | Pattern logs, retros |

A few of these deserve specific framing.

**Skills** are the cleanest answer to "AI slop." A `frontend-design.md` skill loaded into context turns a generic CSS dump into one that uses semantic tokens, a typography system, and consistent hover states. The skill is portable. The prompt that invokes it is short. Encoded knowledge beats clever prompting.

**MCP servers** are the agent's I/O bus. Context7 fetches current library docs at prompt time so the agent stops confidently writing against APIs that no longer exist. Chrome DevTools MCP gives the agent eyes — screenshots, console errors, performance traces — and collapses the narrate-and-iterate debug loop. The framing I keep coming back to: *skills teach the agent **how** to work; MCP servers teach the agent **what** to access.*

**State externalization** is the answer to context-window degradation. Instead of one long session that gets fuzzier the longer it runs, you architect for ephemerality: short fresh sessions reading state from the filesystem, writing back to the filesystem, with `progress.txt` accumulating gotchas across iterations. Git becomes the agent's memory.

**Permissions** are the safety floor. A `permissions.deny` list with `Bash(rm -rf *)`, `Bash(git push --force)`, and `Bash(npm publish)` is what turns Claude Code from "an assistant that can do dangerous things" into "an assistant for which dangerous things are constitutionally impossible." It is also what makes everything in Chapter 5 viable. You cannot run three agents in parallel if any one of them might `rm -rf` the repo.

---

## The Orchestrator Paradigm

Chapter 5 is the conceptual hinge. The progression from one agent to many is not a productivity trick — it is a shift in role.

The four questions to ask before launching multiple agents:

1. Is the work divisible into independent pieces?
2. Are interfaces between pieces explicit?
3. Do you have tests or CI that can validate output?
4. Can the team tolerate asynchronous completion?

If any answer is no, do not orchestrate. Coupled work runs faster sequentially.

When the answers are yes, the orchestrator's job stops being to write code. It becomes work-graph design, contract definition, and integration leadership. The pivotal move in the capstone demo is that you don't write the agent definitions yourself — you describe the team you want, and the agent generates the `.claude/agents/*.md` files for you.

The course names this the **Factory Model**: you've shifted from writing all the code to building the rig that builds the code. Precise specs are precise factory inputs. Vague specs replicate defects across the agent fleet — three agents amplify every contract ambiguity by 3x.

I haven't run a three-agent swarm on real work yet, so I'm holding my opinions on the operational ergonomics until I have. The conceptual frame, though, is the part that transfers immediately: even when you're driving one agent, you're already designing a contract. Every prompt is a one-agent factory input. The factory frame just makes that explicit.

---

## Three Implications of the Harness Frame

**1. The bottleneck moves outward.** Once the harness is good, the model becomes nearly fungible. Swap Sonnet for Opus, Claude for something else, and the rig still produces. Investment in `.claude/` compounds across model upgrades instead of being burned by them. A directory full of skills, commands, and agent definitions is more durable than any single prompt.

**2. It's a software discipline, not a prompting trick.** Harnesses get versioned, tested, reviewed, shared. `.claude/` is source code. `progress.txt`, `RESEARCH.md`, `migration-plan.md`, and `pattern-log.md` are commit-worthy artifacts. The "AI engineer" role is mostly systems engineering with an LLM-shaped component in the loop.

**3. Failure modes are usually systems failures, not just model failures.** When a multi-agent run produces inconsistent output, the contract is often ambiguous. When the agent hallucinates a Next.js 15 API, missing docs in the I/O harness make that failure mode far more likely. When a long session drifts, the state harness is often missing or too weak. *"Where in the harness did the signal degrade?"* is the diagnostic question that scales.

---

## The Substrate Question

The harness frame raises a question the course never quite answers: if AI-assisted development is harness engineering, what does that mean for people without an engineering background?

The honest answer is that harness engineering has prerequisites, and the prerequisites are software engineering itself. The course teaches which knobs in the harness exist. It does not teach the concepts the knobs implement — it assumes them.

| Harness layer | The concept it silently assumes |
|---------------|---------------------------------|
| `CLAUDE.md` / `SPEC.md` split | Durable invariants vs. disposable requirements |
| Skills | What a "domain best practice" actually is |
| MCP servers | Process boundaries, stdio, schemas, request/response contracts |
| Hooks + permissions | Side effects, blast radius of destructive operations |
| RALPH loops (Read -> Act -> Log -> Plan -> Halt) | Why context windows fill, why state belongs in git |
| Multi-phase plans | What a "safe rollback point" is |
| Subagents → swarms → teams | Concurrency, race conditions, contract-first design |
| Shared types as contracts | Why interfaces decouple |

None of those concepts are taught by the course. They are the substrate the harness rests on. Without the substrate, the harness becomes a magic incantation — copying templates, installing servers you don't understand, running `/verify` because someone said to. It works until it doesn't, and when it doesn't there's nowhere to go.

The marketing pitch — *"AI lets non-engineers build software"* — is true at the demo layer and progressively less true the closer you get to production. AI made code generation cheap. It did not make reliability engineering, architectural judgment, or calibrated trust cheap. Those still take years. They are now worth dramatically more, not less, because everything around them got cheap.

---

## Frameworks Are Perishable, Concepts Are Not

A related point that's worth saying out loud, because it changes how unfamiliar stacks should feel:

> You don't learn frameworks. You learn concepts, and frameworks become surface variations of concepts you already understand.

What feels like *"I'm shaky on Next.js"* is often a vocabulary gap, not a substrate gap. Server components can often be reasoned about as server-rendered units with embedded data. Middleware is a request/response interceptor. The App Router is URL dispatch. `fetch` caching is HTTP caching behavior re-skinned. The concepts are decades old. The vocabulary is two years old.

This matters because it changes the cost of working with AI on an unfamiliar stack from *"buy 200 hours of expertise"* to *"spend an afternoon mapping vocabulary onto concepts I already own."* You can encode that map directly into the harness:

```markdown
## Translation map for this project
- `app/api/route.ts` is what I'd call a Flask blueprint route.
- `middleware.ts` runs in the same lifecycle slot as Flask's `before_request`.
- Server components ≈ server-rendered Jinja templates with embedded data.
- Hydration = the JS bundle re-attaching event handlers to server-rendered HTML.
```

Now the harness compensates for the gap. The agent stops generating code in funny clothes and starts generating code in concepts you can grade.

---

## Five Inseparable Principles

Reading the whole course as one body of work, five principles hold at every scale:

1. **Specification is architecture.** Every autonomous decision the agent makes flows from how clearly you describe outcomes and constraints.
2. **Externalize state.** Filesystem, git, task lists, and committed plans are durable; chat context is not.
3. **Encoded knowledge beats clever prompting.** Skills, MCP servers, and slash commands turn one-off cleverness into shared infrastructure.
4. **Contracts before parallelism.** Shared types, explicit interfaces, and ownership boundaries are what let multiple agents work without colliding.
5. **Quality at integration points, not keystrokes.** Don't review every step; review tests, reports, and PRs. Trust the autonomy you architected for.

These compound. A great spec (#1) committed to a plan file (#2) defines a contract (#4) that lets agents work in parallel while a `/verify` command (#3) enforces quality at the PR boundary (#5).

---

## What I'm Taking Forward

The artifact to leave a course like this with is not a head full of patterns. It's your own `.claude/` directory — populated with the skills, commands, and agent definitions you actually use; pointed at the MCP servers you actually need; and paired with a folder of pattern logs that records what works.

That's the factory. The factory is what compounds.

Treat your `.claude/` like infrastructure, because that's exactly what it is.

---

*Reflecting on Addy Osmani's* [Mastering AI-Assisted Development](https://www.linkedin.com/learning/mastering-ai-assisted-development) *(LinkedIn Learning).*
