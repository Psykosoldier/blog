---
title: "Welcome — Running a Home on an AI Agent, and Writing Down What Breaks"
date: 2026-09-24T00:00:00+02:00
draft: false
categories: ["welcome"]
tags: [ai-agents, home-assistant, project-overview, second-opinions]
cover:
  image: "header-0-introduction.png"
  alt: "Welcome — Running a Home on an AI Agent, and Writing Down What Breaks"
  relative: true
ShowToc: true
TocOpen: false
---

I'm starting this blog for a simple reason: I've been running my home automation system with an AI coding agent acting less like a chatbot and more like a project lead — reading a backlog, opening pull requests, asking for sign-off on the risky stuff — for months now. It's been genuinely useful, occasionally broken in genuinely funny ways, and at this point my AI agent has more commit access to my house than most of my actual family.

> I haven't found much writing that treats this setup as the real, load-bearing infrastructure it actually is, instead of a novelty demo where someone asks a chatbot to turn on a lightbulb and calls it a day.

So I'm writing it down as it happens — the wins and the "why is the alarm system arguing with the vacuum cleaner" moments alike.

---

## Who this is for

Two groups, deliberately: people who live in the **smart home / home automation** world and want to see what "AI-managed" actually looks like day to day, not just in a demo video — and people who work with **AI agents generally** (coding agents, autonomous workflows, anything with real write access to something that matters) and want concrete, specific stories rather than abstract best-practice advice. If you only care about one of the two, you'll still get something out of the other half — the failure modes turn out to be mostly the same regardless of whether the agent is touching a door lock or a production database.

---

## What this actually is

Home Assistant, self-hosted, fully version-controlled, running on real hardware sitting on a shelf in my house — not a cloud instance, not a demo VM. An AI coding agent with real write access to that configuration, a self-hosted issue tracker, and a set of deliberately scoped permissions — some things it does freely, some things need my sign-off, a few things it's not allowed to touch at all, on pain of a stern chat.

It's not a toy. It arms and disarms a real alarm system, controls real door locks, and decides when a robot vacuum should or shouldn't start based on whether anyone's home. Bugs here aren't failed unit tests you shrug off with a red CI badge — they're a door that doesn't lock, or an alarm that goes off because a vacuum cleaner committed the crime of existing.

### The stack, roughly

```
Home Assistant (self-hosted)  ──┐
Git repo, full config in it     ├── AI coding agent, scoped read/write
Self-hosted issue tracker       │   + a dedicated review pass +
CI on every change              ├── external second opinions before
Static status dashboard         │   anything safety-critical ships
Nightly/weekly unattended jobs ─┘
```

Two physical boxes underneath all of it — a low-power mini-PC running Home Assistant itself, and a NAS doing double duty as the Git server and CI runner. The [daily workflow post](../daily-workflow/) has the actual make/model/spec sheet, and the [architecture deep-dive](../architecture-deep-dive/) covers how the agent process stays alive across all of it.

### A rough sketch of what a normal day looks like

- The agent starts each session by reading a running log and a backlog of open work, and checks whether I've flagged anything specifically for it via a small dashboard on my phone.
- Every piece of real work — feature, bug fix, even a documentation change — gets a short ID and a tracked issue. The open issues *are* the project status; there's no separate document to keep in sync.
- Nothing risky ships on one opinion. Changes to anything safety-critical get reviewed by a dedicated review pass plus two independent second opinions from different model providers — and even then, as you'll see in the first real post, all three can be confidently wrong at the same time.
- A handful of unattended jobs run overnight — light audits, health checks, security log reviews — and file findings for me to triage when I'm back, never acting unsupervised on anything non-trivial.
- Long sessions eventually hit a context limit and get cleared; a handoff step writes down exactly what was mid-flight so the next session doesn't have to reconstruct it from git history. A separate, longer-lived memory keeps track of preferences and hard lessons across all of that — not "what were we doing," but "what should never happen again."

That last part is where most of the interesting content on this blog will come from: not the features that worked, but the specific, dated moments where the process caught something — or didn't, and had to grow a new guardrail afterward, usually accompanied by me staring at a diff at an unreasonable hour wondering how a robot vacuum nearly got my house burgled by itself.

---

## What you'll find here

Mostly two kinds of posts. **Incident write-ups** — something broke, here's the actual root cause, here's the fix, here's the guardrail added so the same class of bug can't recur silently. And **pattern posts** — a piece of reusable design that came out of one of those incidents, generalized enough to apply outside my specific setup (state persistence across restarts, how to avoid an agent racing itself, why timeouts are often the wrong signal for "a process is done").

I'll keep every post technical and specific — real config snippets, real reasoning, not a sanitized case study. The one thing I'll deliberately leave out or generalize is anything that would tell someone exactly how my house's security setup behaves — the *pattern* is the point, not the exact parameters of my alarm system.

If you're running something similar, or thinking about it, I'd genuinely like to hear what's broken for you too — that's a big part of why this is public instead of a private notebook.

First real post is up next: [three independent AI code reviews unanimously flagged the same change as critical](../three-ai-reviews-agreed/). All three were wrong, and the actual bug was something none of them found.
