---
layout: post
title: "tmux for Long-Running Remote Agents"
date: 2026-04-27
categories: [Software Engineering]
tags: [tmux, claude-code, ssh, workflow, ai-agents]
---

tmux has been around since 2007, and most developers treat it as a fancier version of terminal tabs. That sells it short. The real reason tmux exists is to **separate session state from the terminal emulator**: a shell can keep running after the window closes, the SSH drops, or the laptop lid slams shut — as long as that shell lives on a machine that stays awake.

That last clause is the whole point. If you only run agents on your laptop, tmux is a nice-to-have for organization. If you SSH into a desktop, workstation, or cloud VM to run long agent tasks, tmux is the tool that makes the workflow possible at all. Without it, every dropped SSH connection kills your agent. With it, the agent keeps grinding while you close the lid and go somewhere else.

This article focuses on that case: running AI coding agents on a remote machine, in tmux, so the work survives you walking away.

---

## Why Shells Die When SSH Drops

Before the tooling, the mechanism. When you SSH into a remote machine, the process tree looks like this:

```
sshd
 └── sshd (your connection)
      └── your login shell
           └── claude agent
```

Your shell's **controlling terminal** is a pseudo-terminal (PTY) that `sshd` allocated for this connection. When the connection dies — Wi-Fi blip, laptop sleep, closed terminal — `sshd` exits, the PTY goes away, and the kernel sends **SIGHUP** to everything attached to it. The shell exits. The agent exits. Hours of work, gone.

This is 1970s Unix behavior. "HUP" literally stands for "hang up," from the era of phone-line terminals. It is correct behavior — if the user is gone, clean up their processes — but it is the opposite of what you want when an AI agent is mid-task.

tmux breaks the chain by running as a **long-lived daemon** on the remote machine. The agent's shell is a child of the tmux daemon, not of sshd. When sshd exits, the tmux daemon does not notice and does not care. The shell's PTY is owned by tmux, so no SIGHUP fires. The agent keeps running until you reconnect and attach to it.

That is the one feature that matters. Everything else tmux does is icing.

---

## The Five-Minute Primer

Three concepts, in order of containment:

- **Session**: the top-level container. It survives detach, disconnect, and network drop. This is the thing that makes tmux useful.
- **Window**: like a browser tab inside a session.
- **Pane**: a split inside a window.

All tmux keybindings go through a **prefix key**, default `Ctrl-b`. tmux is a middle layer between your keyboard and the shell, so it needs a way to distinguish "command to tmux" from "text for the shell." The prefix solves that collision. Press `Ctrl-b`, release, then press a command key — tmux interprets it. Everything else passes straight through.

The essential commands from your local laptop:

```bash
ssh user@remote
tmux new -s work        # start a new named session on the remote
```

Inside tmux, after the prefix:

```
d           detach (session keeps running on the remote)
c           new window
n / p       next / previous window
,           rename current window
%           split pane vertically
"           split pane horizontally
arrow       move between panes
z           zoom current pane (toggle)
[           enter scrollback / copy mode (q to exit)
```

From your laptop again, after detaching or after a dropped connection:

```bash
ssh user@remote
tmux ls                 # list sessions still running
tmux attach -t work     # reattach to the one you want
```

That is the whole workflow. No plugins, no 200-line config. Extend later if you feel a specific missing thing.

---

## The Core Pattern: Agent on Remote, Client on Laptop

This is the one pattern that justifies learning tmux for most people.

```
[your laptop]                    [remote machine, always on]
 terminal + ssh  ── network ─>   tmux daemon
                                  └── shell
                                       └── claude agent
```

The workflow:

```bash
# from the laptop
ssh user@remote
tmux new -s refactor
claude                  # or whatever agentic CLI
# give the agent a task, let it start working
# press Ctrl-b d to detach — session keeps running
# close the terminal, close the laptop, walk away
```

Hours later, from anywhere:

```bash
ssh user@remote
tmux attach -t refactor
# scrollback intact, agent still running or waiting for input
```

The agent never knew you left. The remote machine never slept. The tmux daemon never hung up.

This is the single most valuable thing tmux does, and it is worth installing for this reason alone.

---

## Why Local tmux Does Not Solve This

Closing the lid on a laptop is **not** the same as detaching from tmux. Lid close triggers suspend-to-RAM: the kernel freezes every userspace process, including tmux, and cuts power to the CPU, disk, and network interface. When you open the lid again:

- tmux survives as a *process*, because nothing was killed.
- Your scrollback survives.
- Any open network connection — including the agent's HTTPS stream to the Anthropic API — has been dead for minutes. The remote server tore it down during the suspend. The agent sees a read error on wake and stops mid-task.

Locally, tmux saves you from accidentally closing a terminal tab. It does not save you from the laptop going to sleep mid-run.

If your only machine is the laptop and you need to walk away, the fix is to prevent sleep, not to rely on tmux:

```bash
caffeinate -dims claude        # macOS: blocks display, idle, and system sleep
```

On Linux:

```bash
systemd-inhibit --what=idle:sleep claude
```

Or run the agent on a machine that does not sleep, which brings us back to the remote pattern.

---

## What Else tmux Is Good For, Honestly

Once you have tmux installed for the remote-persistence case, a few smaller patterns become free. These are nice, not essential.

**Scrollback per pane.** Each agent run has its own history. Raise the default with `set -g history-limit 50000` in `~/.tmux.conf`.

**Parallel sessions on the same remote.** One session per task, each with its own working directory:

```bash
tmux new-session -d -s feature-auth -c ~/projects/repo/.worktrees/feature-auth
tmux new-session -d -s fix-parser   -c ~/projects/repo/.worktrees/fix-parser
tmux send-keys   -t feature-auth 'claude' C-m
tmux send-keys   -t fix-parser   'claude' C-m
tmux ls
```

This pairs well with git worktrees — one worktree per task, one tmux session per worktree, one agent per session. Clean filesystem and process isolation.

**Reattach from multiple devices.** Your laptop attaches; your desktop attaches to the same session from the office. Both see the same shell, same cursor, same output. Useful when you move between locations mid-task.

**Scriptable layouts.** If you run the same agent configuration often, put it in a shell script:

```bash
#!/usr/bin/env bash
set -euo pipefail
SESSION="work"
PROJECT="$HOME/projects/repo"

tmux has-session -t "$SESSION" 2>/dev/null && exec tmux attach -t "$SESSION"

tmux new-session -d -s "$SESSION" -c "$PROJECT" -n agent
tmux send-keys   -t "$SESSION:agent" 'claude' C-m
tmux attach      -t "$SESSION"
```

Run it locally with `ssh user@remote -t ~/bin/start-agent.sh` and you are inside the session in one command. `has-session` makes it idempotent — rerun it to reattach instead of duplicating.

These are conveniences. If you only need the persistence, you can skip them.

---

## Gotchas

**tmux must be installed on the remote, not the laptop.** This surprises people. Your laptop is just running SSH and a terminal emulator. Install tmux where the work lives.

**Nested tmux is a trap.** If you run tmux locally *and* tmux on the remote, they compete for the same prefix. Either run it on only one end (the remote), or bind a different prefix for the outer session. For the pattern in this article, keep tmux on the remote only.

**TUI rendering sometimes breaks.** Some agent CLIs use rich ANSI output that misbehaves under tmux's default `TERM`. Add this to `~/.tmux.conf` on the remote:

```
set -g default-terminal "tmux-256color"
```

If the local terminal does not recognize `tmux-256color`, fall back to `screen-256color`.

**Scrollback is per-pane and finite.** Default is 2000 lines, which an agent eats quickly. Raise it, or redirect important output to a file.

**Detach is not kill.** `Ctrl-b d` leaves the session running on the remote. If you actually want to stop an agent, exit the agent first. `tmux ls` before creating new sessions is cheap and worth the habit.

**Session names go into scripts.** Stick to lowercase letters, digits, and dashes. Names with spaces or colons break `tmux attach -t` lookups.

---

## Summary

The short version, one sentence at a time:

- Shells die on SSH drop because the kernel sends SIGHUP when the controlling terminal disappears.
- tmux runs as a daemon on the remote machine, so it owns the shell's terminal instead of sshd.
- When SSH drops, the daemon does not get SIGHUP, and the shell — and the agent — keep running.
- You reconnect later from any machine and `tmux attach` to pick up where you left off.
- This only works on a machine that stays awake. On a laptop that sleeps, tmux preserves scrollback but not in-flight network work.

If you SSH into remote machines to run agents, install tmux there today. If you only run agents locally, stick with your terminal emulator's tabs and splits — tmux is not the tool you need. The persistence is the feature; everything else is a bonus.

<!-- buddy: the dragon guards the hearth on the remote mountain, not the one in the traveler's pocket -->
