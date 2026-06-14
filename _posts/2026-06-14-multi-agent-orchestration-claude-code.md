---
title: "Turning Claude Code Into a Team: Multi-Agent Orchestration"
author:
  name: Łukasz Komosa
  link: https://github.com/komluk
date: 2026-06-14 10:00:00 +0000
categories: [Claude Code, AI, Architecture]
tags: [claude-code, agents, orchestration, scaffolding]
render_with_liquid: false
---

When I use [Claude Code](https://www.anthropic.com/claude-code) for anything beyond a quick edit, the single-assistant model starts to strain. One context window holds the requirements, the API design, the implementation, the tests, the security review, and the git history all at once. It works, but the assistant is a generalist juggling roles, and the quality drops exactly where you need it most: the handoffs.

So I built [`scaffolding`](https://github.com/komluk/scaffolding) — a spec-driven multi-agent orchestration plugin for Claude Code. It is markdown-only: every agent, skill, and hook is a plain `.md` file. No backend, no daemon, no database. You install it as a plugin and Claude reads the markdown to know how to behave. It is listed on ClaudePluginHub and currently sits at `#6` on the [Stack Overflow for Agents leaderboard](https://agents.stackoverflow.com/agents/leaderboard).

This post is the architectural overview. There is a companion post on the spec-driven workflow itself.

## The core idea: route, don't generalize

Instead of one assistant doing everything, every task is routed to a **specialized agent** with a focused role and a deliberately narrow tool set. The plugin defines `11` agents:

- **analyst** — requirements, scope assessment, feasibility, proposals
- **architect** — system design, API design, implementation planning, multi-file refactoring
- **researcher** — library questions, API integration, best practices
- **developer** — implementation, bug fixes, features, tests, UI
- **debugger** — bug reports, unexpected behaviour, stack traces
- **reviewer** — security analysis, threat modeling, quality gates
- **optimizer** — performance, database design, schema, queries
- **tech-writer** — README, CHANGELOG, docs (and *only* docs)
- **devops** — CI/CD, deployment, infrastructure
- **gitops** — all git operations
- **coordinator** — decomposes a task into a sequence of agent steps

The win is not that any single agent is smarter. It is that each one operates with a clean, scoped context and a clear mandate. The `reviewer` is not also trying to be helpful about implementation; it is looking for problems. The `tech-writer` owns documentation and nothing else touches it. Narrow roles produce sharper output.

## Routing protocol

The orchestrator never answers directly — it routes. A decision tree maps task types to agents:

```
bug report        -> debugger -> developer
complex feature   -> analyst  -> architect -> developer
simple feature    -> developer
review / security -> reviewer
docs / library    -> researcher -> tech-writer
git operations    -> gitops
ambiguous request -> analyst
```

Each arrow is a structured handoff. The `debugger` produces a diagnosis; the `developer` receives it and implements the fix. The output of one agent becomes the grounded input of the next, instead of everything sloshing around in one undifferentiated context.

There is also a list of **blocked agent types** — generic catch-all subagents that conflict with the custom roles. If you let a generalist agent slip into the chain, it quietly undoes the specialization you set up. The protocol forbids it explicitly.

## Worktree isolation and the git separation rule

For non-trivial work, `developer` and `architect` agents can run inside an isolated **git worktree**. Parallel agents then work on separate branches without colliding on the working tree. This matters once you start running more than one implementation thread at a time.

The rule I consider most important in the whole plugin: **the implementer writes code, a dedicated `gitops` agent owns every git mutation.** The developer never commits. It writes files and runs tests, and that is all. When it finishes, the orchestrator spawns `gitops` to commit, merge the worktree branch back, and push.

```
developer  (in worktree)  ->  writes code, runs tests, STOPS
gitops     (main repo)    ->  add, commit, merge, push, clean up worktree
```

This clean separation prevents the worst failure mode I hit early on: merging a worktree before anything was committed, which silently throws away all the work. By making git a single agent's exclusive responsibility, that class of bug disappears. One owner, one set of mutations, no races.

## Skills: knowledge on demand

The plugin ships `34` **skills** — reusable, on-demand knowledge modules. Think `api-design`, `python-patterns`, `react-patterns`, `testing-strategy`, security checklists, and so on. Each is markdown, and each carries a short `TRIGGER` / `SKIP` description:

```
TRIGGER: designing or reviewing a REST/GraphQL API surface
SKIP:    internal helper functions, no external contract
```

The point is context economy. You do not want every guideline loaded into every prompt — that bloats the window and dilutes attention. The `TRIGGER`/`SKIP` wording lets the right skill load at the right moment and stay out of the way otherwise. It is lazy loading for expertise.

## Hooks: guardrails that don't depend on the model behaving

Agents are probabilistic, so I do not trust them to never do something destructive. The plugin includes `9` **hooks** — deterministic guardrails that fire around tool calls:

- block destructive `rm` invocations
- block `git push --force`
- pre-commit validation before a commit lands
- file-staleness checks so an agent does not edit a stale copy
- post-edit review trigger
- a `SessionStart` reminder that re-injects the routing protocol

Hooks are the floor. No matter how the model reasons, these run as code and either allow or deny. They are what make me comfortable handing agents real write access to a repository.

## Spec-driven workflows

For larger work, agents produce OpenSpec-style artifacts: a **proposal**, then a **design**, then a **tasks** breakdown, stored under a conversation-scoped path. This keeps the work grounded and reproducible — you can read why something was built before reading what was built, and you can replay the chain. The companion post covers this in detail.

## Optional semantic memory

There is also opt-in cross-device **semantic memory** via an MCP server: agents can recall past decisions and gotchas by similarity query, scoped per project. It is entirely optional. Without it, agents fall back to plain file-based memory in the repo. I did not want the core orchestration to depend on a running service.

## Is it worth it?

Let me be honest about the trade-off, because "more agents" is not automatically better.

The cost is real. There are more moving parts. There is a routing protocol to maintain. Every handoff is coordination overhead, and for a one-line fix that overhead is pure waste — just edit the file. The system also adds latency: spawning a `debugger` then a `developer` then a `gitops` is slower than one assistant doing all three.

The payoff shows up on **non-trivial work**. Specialization keeps each step focused. Structured handoffs make the flow reviewable — you can see the diagnosis, the design, the implementation, and the review as discrete artifacts. Guardrails make destructive mistakes hard. The result is more reliable and more auditable than a single mega-prompt trying to hold everything in its head.

So my rule is simple: trivial change, stay single-assistant. Anything with design, review, or git risk attached, route it. The plugin is open source at [github.com/komluk/scaffolding](https://github.com/komluk/scaffolding) — it is all markdown, so it is easy to read exactly what each agent is told to do before you trust it with your repository.
