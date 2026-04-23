---
layout: post
title: "Running Parallel Claude Code Sessions with Git Worktrees"
date: 2026-04-23
categories: [Software Engineering]
tags: [claude-code, git, workflow]
---

Git worktrees have been around since Git 2.5, but they remain one of the most underused features in the ecosystem. The idea is simple: a single repository can have multiple working trees checked out simultaneously, each on its own branch, each with its own filesystem state. No stashing. No `git checkout` back and forth. Just parallel copies of the repo, all sharing the same `.git` object store.

If you use Claude Code, worktrees pair naturally with it. The important point is that the workflow itself is just Git. You do not need any special magic to benefit from it. Once each task has its own working tree and branch, you can run separate coding sessions in parallel without the usual stash-and-switch overhead.

---

## Why This Helps

Most coding-agent workflows break down for the same reason human workflows do: too many unrelated edits competing for one checkout.

- You start investigating one bug.
- Halfway through, you remember a doc update.
- While doing that, you notice a failing test in another area.

Without worktrees, that turns into branch churn, local diff clutter, and a higher chance of mixing unrelated changes into one commit. With worktrees, each task gets its own directory, branch, terminal, and session state.

That is the real value: not speed for its own sake, but clean isolation.

---

## Three Practical Patterns

### Pattern 1: One Worktree Per Task

Best for: keeping independent changes from bleeding into each other.

Create a new worktree whenever a task deserves its own branch anyway.

```bash
# From your main repo checkout
git worktree add .worktrees/feature-auth -b feature-auth
git worktree add .worktrees/fix-parser -b fix-parser
```

Then treat each directory as a fully separate working copy:

```bash
cd .worktrees/feature-auth
claude
```

This is the simplest mental model. One task, one branch, one directory.

### Pattern 2: Multiple Parallel Sessions

Best for: genuinely independent work streams.

Once the worktrees exist, open a separate terminal for each one and run a separate Claude Code session in each directory.

```bash
cd .worktrees/feature-auth && claude
cd .worktrees/fix-parser && claude
```

This is where worktrees become more than a Git convenience. Each session has its own filesystem state, branch, diffs, and local context. You are no longer forcing one session to juggle unrelated edits in a single checkout.

The tradeoff is operational overhead. Every worktree is a real directory with its own dependencies, caches, and build artifacts. Isolation is clean, but it is not free.

### Pattern 3: Plan First, Implement Second

Best for: larger changes where setup discipline matters.

Before you start coding, decide three things explicitly:

1. What branch name belongs to the task?
2. What setup does the worktree need?
3. What validation will prove the branch started from a clean baseline?

For example:

```bash
git worktree add .worktrees/docs-refresh -b docs-refresh
cd .worktrees/docs-refresh

# project-specific setup
npm install

# or
uv sync

# then run a clean baseline check
npm test
```

This pattern sounds boring, but that is why it works. By the time the coding starts, you already know the branch name, the environment state, and whether the starting point was healthy.

---

## Setup Checklist

If you plan to use worktrees regularly, set up the repo once and avoid preventable mistakes.

### 1. Gitignore the parent directory

```bash
echo ".worktrees/" >> .gitignore
git add .gitignore && git commit -m "chore: gitignore worktrees directory"
```

Without this, a careless `git add .` can accidentally stage the nested worktree as an embedded repository entry.

### 2. Choose a consistent naming scheme

Examples:

- `feature-auth`
- `fix-parser-null-case`
- `docs-api-overview`

Good names make `git worktree list` readable, which matters more than people think.

---

## Gotchas

**Dependencies are not shared.** Each worktree is a physical directory. `node_modules`, `.venv`, `target/` — all duplicated. On a large project, this adds up. Factor in disk space when deciding how many worktrees to run simultaneously.

**One branch per worktree, by default.** Git will not normally let you check out the same branch in two worktrees. If you try to create a worktree from a branch that is already checked out elsewhere, git will refuse. Use `-b` to create a new branch when adding a worktree.

**`.worktrees/` should be gitignored.** This is easy to miss and annoying to clean up later.

**Do not `rm -rf` worktree directories.** Git maintains metadata about active worktrees in `.git/worktrees/`. Deleting the directory without telling git leaves orphaned metadata. Use `git worktree remove` instead. If you have already made the mistake, `git worktree prune` cleans up stale entries.

**Avoid cloud-synced paths when possible.** OneDrive, Dropbox, and iCloud Drive sync at the filesystem level and can interfere with Git file locking. If a repository lives in a synced location, creating worktrees elsewhere on the local disk is usually safer.

---

## Summary

| Pattern | When to Use | Isolation | Manual Control |
| --- | --- | --- | --- |
| One worktree per task | Clean separation for normal feature work | Per task | High |
| Multiple parallel sessions | Several independent tasks at once | Per session | High |
| Plan first, implement second | Larger tasks with setup and validation discipline | Per task | Medium |

The useful distinction is not between different kinds of tooling. It is between levels of isolation and levels of discipline.

Worktrees are not magic. They are just a cleaner unit of parallel work than stash-and-checkout cycles. If you use Claude Code regularly, that cleanliness matters even more: each session gets its own sandbox, and the merge happens when you are ready, not when context-switching forces the issue.
