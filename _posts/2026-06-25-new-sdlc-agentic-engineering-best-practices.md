---
title: "The New SDLC: Agentic Engineering Best Practices, Applied to a Real Plugin"
author:
  name: Łukasz Komosa
  link: https://github.com/komluk
date: 2026-06-25 09:00:00 +0000
categories: [Claude Code, AI, Architecture]
tags: [claude-code, agents, agentic-engineering, context-engineering, best-practices]
render_with_liquid: false
---

Google and Kaggle published a whitepaper this month — *[The New SDLC With Vibe Coding](https://www.kaggle.com/whitepaper-the-new-SDLC-with-vibe-coding)* by Addy Osmani, Shubham Saboo and Sokratis Kartakis. I read it through the lens of [`scaffolding`](https://github.com/komluk/scaffolding), the multi-agent orchestration plugin for Claude Code I maintain, and it turned out to be a surprisingly good checklist. Some of its principles I had already stumbled into; others pointed straight at gaps I hadn't named yet.

This post is the synthesis: the whitepaper's durable ideas, each mapped to something concrete in a plugin that real projects run.

## Agent = Model + Harness

The single most useful framing in the paper:

> An agent is a model wrapped in a harness — instructions, rule files, tools, sandboxes, orchestration, hooks, observability. Roughly **10% model, 90% harness**.

The counterintuitive corollary: **most agent failures are configuration failures, not model failures.** When an agent misbehaves, the reflex to blame the model is usually wrong. More often it traces to a missing tool, a vague rule, an absent guardrail, or a context window stuffed with noise. The evidence they cite is blunt — one team moved a coding agent from outside the top 30 to the top 5 on Terminal Bench 2.0 by changing *only* the scaffold, no model change.

This is the whole thesis behind a plugin like `scaffolding`. The harness *is* the product:

- **11 agents**, each a specialised system prompt with a scoped toolset — `developer` gets `Edit`/`Write`, `reviewer` gets read-only + `WebSearch`, `gitops` owns all git.
- **Hooks as enforceable constraints** — `block-env-write.sh`, `block-destructive-rm.sh`, `block-force-push.sh`. These are not suggestions; they sit on `PreToolUse` and can stop the action.
- **35 skills** carrying domain knowledge the base model doesn't reliably have.

The practical takeaway: when your agent does something dumb, debug the harness first. The model is the part you can't fix today anyway — and it gets swapped underneath the harness over time, so harness work is the durable investment.

## Static vs. dynamic context is a cost lever

The paper splits context into what's loaded **every turn** (static — system instructions, rule files like `CLAUDE.md`, core guardrails) versus **on demand** (dynamic — skills, tool results, RAG). Static context is reliable but expensive: you pay for it on every single call. The discipline is to treat that boundary as code — reviewed, versioned — and to push reference material into **progressive disclosure**: the agent sees metadata at startup and loads the full instructions only when a task matches.

I had applied half of this lesson already (see [Cutting Token Cost in a Claude Code Plugin](/posts/cutting-token-cost-claude-code-plugin/)), but the whitepaper made me look again and I found more. My `CLAUDE.md` — injected on every turn — still carried four full sections that *already existed as skills*:

| Static section in `CLAUDE.md` | Already covered by skill |
|-------------------------------|--------------------------|
| Worktree Delegation Protocol  | `worktree-management`    |
| MCP Tools                     | `mcp-tools`              |
| OpenSpec & Specs Path         | `spec-workflow`          |
| Delegation Format (verbatim)  | (redundant with Task syntax) |

That is the anti-pattern in its purest form: paying a static, per-turn token cost for content that was already available dynamically. The skills carry proper trigger metadata in their frontmatter:

```yaml
---
name: worktree-management
description: "Scaffolding worktree lifecycle, diagnostics, and recovery.
  TRIGGER when: debugging worktree isolation, recovering a stuck worktree...
  SKIP: ordinary branch/commit/merge operations (use git-operations)."
---
```

So they load *when the task calls for them* — not before. I cut the four sections, folded the rest into a short pointer, and `CLAUDE.md` dropped from **146 → 71 lines (−44%)** with routing fidelity fully intact — all 11 agents and the decision tree stayed. Reference detail belongs in dynamic skills; the always-on file should hold only what's needed to *route the next message*.

**Rule of thumb:** if a block of your rule file is only relevant to a specific kind of task, it shouldn't be in the rule file. It should be a skill with a trigger.

## Vibe coding and agentic engineering are a spectrum — verification decides where you land

The paper's most quotable line: *"set the bar at the eval, not the demo."* Vibe coding and agentic engineering aren't two tools — they're two ends of one spectrum, and the thing that decides where a task sits is **verification rigor**, not whether you used AI. A demo shows it works once; an eval suite shows it works reliably.

It splits verification into two mechanisms:

- **Tests** for the deterministic parts (input → output).
- **Evals** for the non-deterministic parts, in two flavours:
  - *Output evaluation* — is the result correct?
  - *Trajectory evaluation* — was the path (tool calls, reasoning) sound?

This is where I hold the plugin honestly to account. `scaffolding` has the *scaffolding* for verification — agent gates (`gate: no criticals` for `reviewer`, `gate: validation passes` for `developer`), a `validate-agent-output.sh` validator, a `circuit-breaker.sh`. But today the validator checks that the output **has the right YAML frontmatter**, and the `score` is **self-reported by the agent**. That's a structural check, not an eval. It confirms the shape, not the substance.

Honest gap, named plainly: structure ✅, output eval ❌, trajectory eval ❌. The roadmap item that follows directly from the whitepaper is to make the gates real — a `reviewer` that emits a machine-checkable `criticals=N` that a hook can *block* on, and trajectory checks (did `developer` try to run `git commit` when it shouldn't?). That's the difference between a system that *suggests* review and one that *enforces* it.

## The lifecycle compresses unevenly

A point worth internalising: AI collapses implementation but barely touches judgment work. Generation is largely solved; **specification, evaluation, and architectural direction are the new craft.** The paper backs this with a useful counterweight to the hype — a METR study found experienced developers were *19% slower* on some tasks once you count the checking and fixing. AI turns implementation "from writing into reviewing." And the **80% problem** is real: agents nail the first 80% fast, then the last 20% — edge cases, system seams — needs context the model lacks.

This maps cleanly onto why `scaffolding` separates roles instead of having one mega-agent do everything:

- `analyst` → `architect` produce the **spec and the structural calls** (the slow, human-shaped part).
- `developer` does **generation** (the fast, solved part).
- `reviewer` and `gitops` are the **verification and commit gate**.

The separation isn't bureaucracy for its own sake — it's putting the rigor where the whitepaper says the value moved.

## What carries over to anyone building agents

Strip away the plugin specifics and the durable practices are:

1. **Debug the harness before the model.** Missing tool, loose rule, junk context — that's where the bug usually is.
2. **Audit your static context for duplication.** Anything injected every turn that's only sometimes relevant should be a dynamically-loaded skill with a trigger.
3. **Make guardrails enforceable, not advisory.** A hook that prints "consider reviewing" is noise; a hook that blocks on `criticals > 0` is a constraint.
4. **Set the bar at the eval, not the demo.** Self-reported scores are theatre. Build output and trajectory checks that can actually fail.
5. **Put rigor where judgment lives** — specs, architecture, verification — and let agents have the generation.

The line I keep coming back to: *AI amplifies whatever engineering culture it lands in.* A good harness multiplies a good process. It also faithfully multiplies a sloppy one. The plugin is just my attempt to make the process worth amplifying.

---

*Sources: [The New SDLC With Vibe Coding (Kaggle)](https://www.kaggle.com/whitepaper-the-new-SDLC-with-vibe-coding) · [Addy Osmani — The New Software Lifecycle](https://addyosmani.com/blog/new-sdlc-vibe-coding/) · [scaffolding plugin](https://github.com/komluk/scaffolding)*
