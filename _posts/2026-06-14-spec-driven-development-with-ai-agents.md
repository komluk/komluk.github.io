---
title: "Spec-Driven Development With AI Agents"
author:
  name: Łukasz Komosa
  link: https://github.com/komluk
date: 2026-06-14 11:00:00 +0000
categories: [AI, Architecture, Claude Code]
tags: [claude-code, agents, openspec, spec-driven]
render_with_liquid: false
---

I have been driving real implementation work through AI agents for a while now, and I keep relearning the same lesson: the agent is rarely the bottleneck. The bottleneck is intent. If you cannot review what the agent *meant* to do separately from what it *did*, you end up reverse-engineering intent out of a diff. That is slow, error-prone, and it scales badly.

This post is about the fix I settled on: making intent a reviewable artifact *before* any code exists. It is the [OpenSpec](https://github.com/Fission-AI/OpenSpec)-style workflow I wired into my [`scaffolding`](https://github.com/komluk/scaffolding) plugin, and it has changed how much I trust agent output.

## The problem with free-form prompting

Hand an agent a one-line prompt for a non-trivial change and watch what happens. It will:

- **Invent scope.** You asked for a retry on one endpoint; it refactored the whole HTTP client.
- **Forget constraints.** The "must stay backward compatible" you mentioned three turns ago is gone.
- **Bury the *why*.** The reasoning lives in a transient chat log, not in anything you can review or replay.

For a typo fix, none of this matters. For anything with real scope or risk, all of it matters. You only discover the drift when you read the diff — which is exactly the moment when correcting it is most expensive.

The root cause is that **intent and implementation are collapsed into a single step**. The agent decides what to build and builds it in one motion. There is no checkpoint where a human (or another agent) can say "no, that is the wrong scope" cheaply.

## Make intent an artifact

The whole idea of spec-driven development is to split that single step into a chain of reviewable artifacts. Each one is plain Markdown, each one is owned by a specific agent, and each one is a hand-off contract to the next.

### `proposal.md` — the WHY and WHAT

The analyst agent writes this first. It captures the requirements, the scope (explicitly including what is *out* of scope), and the impact. No code, no API signatures — just the problem and the shape of an acceptable solution.

This is the cheapest possible place to catch a misunderstanding. If the proposal says "rewrite the auth layer" and I only wanted a token-refresh fix, I reject it in thirty seconds. No code was harmed.

### `design.md` — the HOW

Once the proposal is accepted, the architect turns it into a technical design: the API shapes, the data flow, the trade-offs considered and rejected. This is where "we will add a `refresh_token` grant to the existing handler rather than a new service" gets decided and *written down with the reasoning*.

The trade-off section matters more than people expect. Six months later, the question is never "what did we build" — the diff answers that. It is always "why did we build it *this* way", and `design.md` is the only place that answer survives.

### `tasks.md` — the ordered checklist

The architect (or analyst) decomposes the design into an ordered, checkable task list:

```markdown
- [ ] Add `refresh_token` grant type to token handler
- [ ] Validate refresh token expiry against store
- [ ] Add integration test for refresh flow
- [ ] Update error responses for expired refresh tokens
```

The developer executes this box by box. There is no ambiguity about "what next" and no room to wander off into an unplanned refactor — the list *is* the scope. When every box is checked, the change is done. Not "feels done", done.

### Review against the spec

Before anything is archived, a reviewer checks the implementation against `design.md` and `tasks.md`. The key word is *against*. The reviewer is not asking "is this code nice in the abstract" — it is asking "does this match what we agreed to build". Drift between the spec and the diff is a defect, full stop.

## Conversation-scoped specs

Multiple changes in flight at once will collide if they share one spec folder. So each change lives under its own conversation directory:

```text
.scaffolding/conversations/{UUID}/specs/
  proposal.md
  design.md
  tasks.md
```

One hard convention here: **the conversation id is a UUID, never a human-readable name.** A name like `auth-fix` feels friendlier, but names collide, get reused, and quietly overwrite each other. A UUID (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`) is unique by construction. If a tool does not hand you one, you generate it:

```bash
uuidgen
```

The cost is that the directory is ugly to type. The payoff is that two parallel changes can never stomp on each other's specs, and the orchestrator never has to guess which conversation an artifact belongs to.

## Quality gates between stages

Artifacts alone are not enough — a confident-sounding-but-wrong proposal still derails everything downstream. So each transition has a gate that low-confidence work cannot pass:

- **Research must clear a score threshold before planning.** If the researcher's findings score below the bar, you do not get to start designing on a shaky foundation.
- **A plan must clear a threshold before implementation.** A vague or incomplete design does not reach the developer.
- **Reviews must surface no criticals.** A critical finding blocks the archive, full stop.

Gates are deliberately boring and mechanical. Their job is to stop the agent's natural optimism from advancing work that has not earned the next stage. An agent will happily report "looks good!" on a half-formed plan; the gate does not care how it feels.

## Prefer minimal, reversible plans

A standing instruction across the whole chain: **prefer the smallest change that works and is easy to roll back.** The architect is biased toward a narrow, reversible design over an elegant big-bang refactor.

This is partly about blast radius — a small change that goes wrong is a small problem — and partly about review cost. A tight, reversible plan is one a human can actually hold in their head and approve. A sprawling refactor is one you rubber-stamp because reviewing it properly is exhausting, which defeats the entire point of the exercise.

## Why this pairs with multi-agent orchestration

The spec chain is not just documentation hygiene. It is the **hand-off contract** that makes multi-agent orchestration work at all:

```text
analyst  → proposal.md
architect → design.md  (reads proposal.md)
developer → code       (reads design.md + tasks.md)
reviewer  → verdict    (reads design.md + tasks.md + diff)
```

Each agent reads the previous artifact and writes the next. No agent needs the others' chat history; it needs the document. That is what makes the pipeline composable — and reproducible, because re-running a stage means re-reading a file, not replaying a conversation. (I have a companion post on the multi-agent architecture itself; this one is just about the spec layer that glues it together.)

## The payoff, and the honest counterpoint

The win is straightforward: agent work becomes **grounded, reviewable, and reproducible.** You review the *intent* once, cheaply, at the proposal stage — instead of reconstructing it from a diff after the fact. The expensive review moves to the cheapest possible moment.

But let me be honest about the cost. Generating a proposal, a design, and a task list for a one-line change is absurd. The overhead is real, and for trivial edits it is pure ceremony — just let the agent fix the typo.

So the rule I actually follow: **reserve the spec workflow for changes with genuine scope or risk.** New subsystems, anything touching security or data, multi-file refactors, anything you would want to be able to explain in six months. For those, the spec pays for itself the first time it catches a scope mistake before a single line of code was written. For everything else, skip it and move on.
