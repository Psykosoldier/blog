---
title: "I Have a Problem: Every New AI Model Release Makes Me Want to Bolt It Onto My House"
date: 2026-09-24T00:00:00+02:00
draft: false
categories: ["tooling-experiment"]
tags: [model-evaluation, ai-agents, risk-scoping, decision-models]
cover:
  image: "header-5-new-ai-toys.png"
  alt: "I Have a Problem: Every New AI Model Release Makes Me Want to Bolt It Onto My House"
  relative: true
ShowToc: true
TocOpen: false
---

I'll admit it upfront: I have a mild, ongoing addiction to trying new AI tools the week they show up, and then finding some corner of my home automation where they might actually earn their keep. Most of them don't survive contact with reality. This is the story of one that did, and what it cost to get there.

## The candidate: a decision model, not a chatbot

A new model showed up on my usual model marketplace — not a chat model, a *decision* model. Instead of prompting it with text and parsing whatever prose comes back, you send it a state description plus a set of typed questions, and it hands back a typed answer with a probability attached. No prompt engineering, no hoping the JSON parses. Small context window, and — the number that actually got my attention — a few cents per million input tokens and effectively free output.

The itch: somewhere in my nightly automation review pipeline, a bundle of findings gets turned into a "should this actually interrupt Dennis right now, or can it wait for the morning digest" decision. That's a yes/no urgency call, made over and over, and it had been running on a hand-rolled heuristic.

> A cheap, purpose-built yes/no model felt like a suspiciously good fit.

---

## Reality check: the docs lied, gently

The community example floating around didn't match what the API actually wanted. Three surprises in a row:

```python
# What the public example suggested:
POST https://openrouter.ai/api/v1/chat/completions
model: "typesafe/jev-latest"
# response: questions.<key>.probability

# What actually worked:
POST https://openrouter.ai/api/alpha/decisions   # different endpoint entirely
model: "~typesafe/jev-latest"                    # yes, that tilde is load-bearing
# response: answers.<key>.noul                   # not .probability, .noul
```

Miss the tilde and you get a cheerful `400: model does not exist`, which is a very polite way of saying "you're speaking the wrong dialect to the wrong endpoint." One of those "read the actual response body instead of assuming the blog post you found was accurate" afternoons.

---

## The part where I nearly shot myself in the foot

Every new decision point in this system gets a fail-safe: if the call errors out or times out, default to the *safe* outcome — in this case, "yes, notify" rather than silently swallowing something important. Good instinct. Badly tested instinct, briefly: while writing a test for that exact fail-safe path, I pointed it at what I *thought* was a throwaway file and it turned out to be the real, live credentials file — which the test then happily emptied out. Nothing caught fire, but I did have to go generate a new API key and mutter something about naming things more carefully.

> Lesson filed away in permanent memory, not just my own head: test failure paths against a fake file you can afford to lose, never the real one sitting three directories up that looks similar enough at 11pm.

---

## Where it landed

After all that: one narrow, well-defined job. A probability threshold set deliberately conservative (better to interrupt someone unnecessarily than to stay quiet about something that mattered), sitting alongside — not replacing — the existing notification path, which still runs regardless. It did **not** get anywhere near anything time-sensitive or safety-critical: presence detection, cover ownership, camera-based detection (it doesn't even do vision) all stayed exactly where they were, deterministic and boring on purpose.

---

## The actual pattern, if you strip the specific model out

This is less "here's a cool model" and more "here's how I decide whether a cool model gets to touch anything important," which is the more reusable part:

1. **Find the narrowest possible job for it first.** Not "improve the automation system" — one specific, bounded decision, ideally one where being wrong is annoying, not dangerous.
2. **Assume the docs are aspirational.** Verify the actual request/response shape yourself before writing a single line of production code against it.
3. **Fail-safe toward the boring, safe default**, and test that fail-safe path somewhere it genuinely cannot cause damage if you screw up the test itself.
4. **Write down what you tried and rejected, not just what you kept.** Half the value of evaluating a new toy is not re-evaluating it for the same job six months later when the hype cycle brings it back around.

Not every new model release deserves a wire into a system that controls physical locks. But somewhere in most home automation setups there's a boring, low-stakes decision that's currently running on a heuristic someone wrote in five minutes and never revisited — those are exactly the right place to let the new, shiny thing prove itself before it earns access to anything that matters.

Deciding *where* the wire is even allowed to go at all is its own recurring theme here — see [the daily workflow post](../daily-workflow/) for how that scoping gets enforced process-wise, not just per-model.
