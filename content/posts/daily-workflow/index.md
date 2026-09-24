---
title: "How I Run a Smart Home With an AI Agent as My Project Manager"
date: 2026-09-24T00:00:00+02:00
draft: false
categories: ["architecture-systems"]
tags: [daily-workflow, git-branching, issue-tracking, hardware, ci-cd]
cover:
  image: "header-4-daily-workflow.png"
  alt: "How I Run a Smart Home With an AI Agent as My Project Manager"
  relative: true
ShowToc: true
TocOpen: false
---

People often ask what it actually looks like day-to-day to have an AI agent manage infrastructure you depend on — not a chatbot you occasionally ask questions, but something closer to a project lead that reads your backlog, opens pull requests, and pings you when it needs a decision.

> Here's the actual stack, the actual hardware it runs on, and the actual daily loop, not the marketing version.

---

## The setup, in one paragraph

My home runs on Home Assistant, self-hosted, with the entire configuration in a self-hosted Git repo (Gitea, running on my NAS). Claude Code — an agentic coding CLI — has full read/write access to that repo and to a set of scoped tools (deploy/reload commands, a self-hosted issue tracker, some read-only APIs for other systems on my network). It does not have access to secrets files directly, cannot restart the core service without asking, and cannot touch a handful of centrally-important files without my explicit sign-off. Everything else — writing automations, fixing bugs, refactoring, documentation — happens largely on its own, inside guardrails I've built up over months.

---

## The hardware underneath it

None of this runs in the cloud. Two boxes, both physically in my house:

| | Home Assistant host | NAS / Git / CI |
|---|---|---|
| Model | Beelink Mini S13 | TrueNAS server |
| CPU | Intel N150, 4 cores | Intel Celeron N5095, 4 cores |
| RAM | 16 GB DDR5 | 64 GB |
| Storage | ~1 TB NVMe | 4× 20 TB + 1× 24 TB HDD pool, plus 2 TB + 500 GB NVMe cache |
| Role | Home Assistant OS + the persistent agent session | Gitea, self-hosted CI runner, nightly backups |

The HA host is a fanless box the size of a paperback — there's no server room here, just a shelf. The NAS is massively over-provisioned for "host a git repo," but it was already doing NAS duty for everything else in the house, so Gitea, the CI runner container, and the nightly backup jobs just moved in as additional tenants on hardware that existed anyway.

Both machines stay on the LAN, nothing agent-facing is exposed to the open internet, and I reach either of them remotely through a VPN back into my own network rather than opening ports — the "reachable from my phone anywhere" part later in this post rides on top of that, not around it.

---

## The daily loop

### 1. Session start

Every work session begins the same way: read a running session log, check a persistent backlog file, and — this is the part that actually matters — check for anything flagged in my self-hosted issue tracker that's assigned specifically to the agent (as opposed to items assigned to me, or items that need both of us). This isn't a suggestion in a document somewhere; it's enforced by a hook that runs on every single message I send, so it's not something that can be quietly forgotten mid-session.

### 2. A dedicated branch per topic, gated by CI before it can even merge

No direct commits to the main branch — the Git server itself refuses the push, not just a convention I try to remember. Every session works on its own branch, and closes out with a pull request that references the tracked issue:

```
git checkout -b session/2026-09-23-geofence-vacuum
# ... work happens ...
git push -u origin session/2026-09-23-geofence-vacuum
# PR body always ends with:
# Closes #220
```

Opening the PR triggers a CI run on a self-hosted runner sitting on the NAS above: a Python test suite against the actual configuration, and the merge button stays greyed out — a required status check enforced server-side — until it goes green.

> Documentation-only changes skip the expensive parts of that run, but the check itself still has to report a result either way, or a purely-docs PR would sit forever in an unmergeable limbo waiting on a check that never fires.

Once it's green, the PR merges itself and the branch gets deleted automatically — no manually cleaning up after every single change, however small.

### 3. Every tracked item gets a number and an issue

Not just features — bug fixes, process changes, documentation updates, all of it. The rule is blunt: if it's real work, it gets a short ID and a corresponding ticket with labels for area, priority, type, and status:

```
[Presence] B-225: Extended geofence for automatic vacuum start
labels: area-presence, priority-medium, status-in-progress, type-feature
```

The reason this matters more than it sounds: an open ticket *is* the state of the project. I can glance at what's open and know exactly what's in progress versus what's just an idea sitting in a backlog file. No separate status document to keep in sync — the ticket state *is* the source of truth. Once the linked PR merges, the issue closes automatically; a closed issue is the audit trail, not a deleted one.

### 4. Second opinions before anything risky ships

For anything touching more than a few files, deleting something, or reaching into a handful of designated critical subsystems (things where a bug means a real-world consequence — a door unlocking, an alarm not arming), one internal review pass isn't enough. The rule is: a dedicated review-focused agent inside the same session, plus a second call out to an entirely different model on a separate API, explicitly not the same vendor as the main agent — and for anything touching the riskiest subsystems, a small panel of two or three more models from different providers on top of that, deliberately spread across different companies and ecosystems so a shared blind spot in one lab's training data doesn't just get echoed back three times.

Genuine diversity of failure modes, not the same reasoning reviewing itself with a different font. (Worth its own article — sometimes all of them are wrong at once. [That's the previous post.](../three-ai-reviews-agreed/))

### 5. A dashboard that fits in my pocket

My self-hosted issue tracker isn't reachable from outside my home network, so I don't get a live view of it on my phone. Instead, a small script rebuilds a static status page once an hour, pulled from the tracker's API, and that page is served from my Home Assistant instance — reachable from anywhere I can reach my smart home dashboard. It's grouped by "waiting on me" versus "agent is handling it" versus "needs both of us," with two tap-targets per item: one flags something for nightly autonomous work, one flags it as needing a real conversation. Zero JavaScript build step, just static HTML regenerated on a cron.

### 6. Nightly and weekly unattended passes

A handful of scheduled, unattended runs happen while I'm asleep: a lightweight nightly audit around 5am on a cheap model looking for anything obviously broken, a weekly deeper pass on a Sunday morning checking documentation and process consistency against the actual codebase, and a set of rotating "optimizer" modules on their own weekly or monthly cadence — one week it's checking system health, the next it's reviewing network stability, the next it's a security log review. None of them are allowed to *act* unattended on anything non-trivial; they file findings, I triage them when I'm back.

> One early lesson that stuck: the nightly job runs inside a container that occasionally restarts for unrelated reasons, and a cron entry doesn't survive that on its own — so the script re-arms its own schedule at the top of every run instead of assuming a static crontab will always be there when it wakes up.

### 7. Memory that survives between sessions

Long-running agent sessions eventually hit a context limit and get cleared; a watchdog process forces a structured handoff before that happens, and a separate, much longer-lived memory layer persists preferences and hard-won lessons *indefinitely*, across all sessions. That mechanism — the actual handoff format, the two-tier memory split, why a plain inactivity timeout wasn't enough — is deep enough to deserve its own post: [the architecture deep-dive](../architecture-deep-dive/) covers it in full.

---

## What I actually do myself

Approving anything that touches the handful of centrally-critical files. Actually reading and clicking "merge" on pull requests when the process calls for a second signature beyond CI — mostly for genuinely judgment-call changes, since routine ones auto-merge once green. Making the calls that are genuinely mine — trade-offs, priorities, "is this worth building at all." Physical hardware installation, obviously; nobody's automating a screwdriver yet. And periodically correcting course when a process rule turns out to be wrong or too rigid, which happens more often than the neat description above makes it sound.

---

## Why this level of process, for a home automation hobby project

The honest answer: because the failure mode without it is worse than the overhead of having it. A system that controls door locks and a security alarm isn't a toy project just because it's "for the home" — a bad automation there has real consequences, not a failed unit test. The process (branches, required CI, tracked issues, second opinions, audit trails) isn't bureaucracy for its own sake; every single piece of it exists because something went wrong once *without* it, and got added afterward as a guardrail. That's also, honestly, the most reusable part of all this for anyone running AI agents on infrastructure that matters — smart home or otherwise: the specific rules matter less than the habit of turning every real incident into a permanent, enforced guardrail instead of a lesson you hope you remember next time.
