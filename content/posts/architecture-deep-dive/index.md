---
title: "The Actual Architecture Behind \"My House Runs on an AI Agent\""
date: 2026-09-24T00:00:00+02:00
draft: true
categories: ["architecture-systems"]
tags: [session-architecture, context-limits, memory, remote-control]
cover:
  image: "header-6-architecture.png"
  alt: "The Actual Architecture Behind \"My House Runs on an AI Agent\""
  relative: true
ShowToc: true
TocOpen: false
---

[The introduction post](../welcome/) gave you the elevator pitch. This one is the wiring diagram — how a coding agent ends up running semi-persistently against real home infrastructure, how I talk to it from my phone on the other side of the planet, and what happens when a session runs for hours and hits a wall it wasn't supposed to hit.

## It's not a chat window. It's a long-running process.

The mental model that trips people up first: this isn't "I open an app, type a question, get an answer, close the app." The agent runs inside a coding CLI, and that CLI process needs to *keep existing* between the moments I'm actually typing to it — otherwise every session starts from zero, re-reading the whole repository to figure out what's going on.

So it runs inside a persistent terminal multiplexer session on the machine that also hosts the home automation server, wrapped by a small launcher script rather than invoked directly:

```
/opt/bin/claude   # tmux-wrapped launcher — this is what a human attaches to
/usr/bin/claude   # plain binary — used for one-shot, non-interactive (-p) invocations
```

Two entry points, two very different use cases. The wrapped one expects a real terminal (TTY) and is what a live, ongoing conversation runs inside — it survives me disconnecting, because tmux doesn't care whether anyone's attached. The plain one is for cron jobs: spin up, run one prompt in headless mode, print the result, exit.

> A nightly audit script doesn't need a persistent session; it needs to run for two minutes at 5am and go away.

---

## Getting to it from anywhere

The part that actually matters day to day: I'm not always at a terminal on that machine. Most of the time I'm on my phone, on a couch, nowhere near the server. The coding CLI's remote-control feature lets a session running somewhere else — a web or mobile client — attach to that persistent tmux session and pick up exactly where the conversation left off. Same context, same running process, just a different window looking into it. Close the laptop, open the phone an hour later, the agent has no idea anything changed.

```
# roughly:
local machine:  tmux session running the wrapped CLI, attached to the repo
remote client:  Claude Code (web/mobile) → remote-control → attaches to
                the SAME running session, not a new one
```

This is the single piece of infrastructure that makes the whole "project lead" framing actually work instead of being a gimmick. A session that resets every time you close a tab can't hold a backlog in its head across a week. A session that's just... still running, whenever you check back in, can.

---

## What happens when a session gets too big

Long-running sessions eventually hit a context limit — there's only so much conversation history a model can hold at once. Left alone, that would mean either the session breaking mid-task, or losing everything and starting cold with zero memory of what was in progress.

### The forced handoff

Instead, a watchdog process monitors context usage (and separately, idle time), and when either crosses a threshold, it interrupts the running session with essentially: *"write down everything that matters before you disappear."* The agent writes a structured handoff to a file — what's mid-flight, what decisions are still open, what NOT to re-litigate, links to the actual git branch/PR state — and then the watchdog clears the context and immediately re-primes the fresh session by pointing it at that handoff file.

```python
# rough shape of the watchdog loop
while session_running:
    if context_usage > THRESHOLD or idle_seconds > IDLE_LIMIT:
        prompt_session("write a full handoff to session-status.md, "
                        "cover what's mid-flight, pending decisions, "
                        "and anything the next session must NOT redo")
        clear_context()
        resume_with_prompt(f"read {handoff_path}, continue from there")
```

### Why not just a timeout

The failure mode this replaced was worse than it sounds: without an active handoff step, a context-limit hit would just silently drop everything, and the next session would either re-derive state from scratch by reading the whole git log (slow, and it misses anything that wasn't committed yet), or — worse — a resume prompt would occasionally not make it into the new session at all, and I wouldn't notice until something was clearly missing.

---

## Two layers of memory, on purpose

Separate from the per-session handoff, there's a second, much longer-lived memory layer that persists across *all* sessions indefinitely — not "what were we doing yesterday," but "how do I like to work, and what specific mistakes should never happen again."

> The distinction matters: a handoff file is disposable once its session resumes cleanly. The long-term memory is the opposite — it's the thing that stops the same lesson from having to be relearned every few weeks.

When a review process catches something, or I correct an approach, that's a memory write, not just a mention in a log file that'll scroll away.

| Layer | Lifespan | Answers |
|---|---|---|
| Session handoff | Disposable, one resume | "What was mid-flight when context ran out?" |
| Long-term memory | Indefinite, across all sessions | "How do I like to work, what should never happen again?" |

---

## Unattended work, on a schedule

A handful of jobs run through the plain (`-p`, non-interactive) entry point on cron, completely separate from the persistent session: a cheap nightly pass looking for anything obviously broken, a deeper weekly review checking documentation against the actual code, and a rotating set of health-check modules — one week network stability, the next a security log review. None of them are allowed to act unsupervised on anything non-trivial. They file findings into the same backlog the persistent session reads, and get triaged like anything else.

---

## The actual lesson

None of this is specific to home automation. Strip out the door locks and the vacuum cleaner and what's left is a general pattern for running an agent against anything that needs continuity: separate the *interactive* long-lived process from the *scheduled* one-shot process, make the long-lived one reachable from wherever you actually are, and build the "oh no, we're about to lose context" moment into an explicit, forced handoff step instead of hoping it never happens. The tmux wrapper and the remote-control attach are almost incidental — the real design decision is treating "the agent forgot everything" as a bug to engineer around, not an inevitability to shrug at.

(Curious what the machine actually running all this — and the NAS behind it — looks like spec-wise? [The daily workflow post](../daily-workflow/) has the hardware rundown; this one stayed deliberately hardware-agnostic since none of it is specific to this particular box.)

Next post: [the time I asked one of these background jobs to only read, and it decided reading was more of a suggestion](../subagent-pushed-to-main/).
