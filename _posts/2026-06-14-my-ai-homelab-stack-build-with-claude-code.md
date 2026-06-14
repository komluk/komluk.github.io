---
title: "My AI Homelab Stack: Building With Claude Code"
author:
  name: Łukasz Komosa
  link: https://github.com/komluk
date: 2026-06-14 14:00:00 +0000
categories: [AI, Homelab, Claude Code]
tags: [claude-code, homelab, self-hosted, ai, mcp]
render_with_liquid: false
---

I have written a handful of posts about individual pieces of my setup -- the LLM gateway, the multi-agent plugin, the spec workflow, the VPN. This one is the tour that connects them. It is the answer to a question I get asked a lot: why run the *whole* AI stack yourself when you could just point [Claude Code](https://www.anthropic.com/claude-code) at a hosted API and be done?

Three reasons, in order of how much they actually drive my decisions:

- **Control.** I decide which models are reachable, where memory lives, and what an agent is allowed to touch. Nothing changes under me without my say-so.
- **Cost.** I run Claude Code through a self-hosted gateway backed by a `claude.ai` OAuth session rather than a metered API key, so I am not watching a token counter on every experiment.
- **Privacy.** Code, notes, and decisions stay inside my own network. The data plane is mine.

The framing for this post comes from Anthropic and ByteByteGo's [Build with Claude Code](https://bytebyteai.com/c/build-with-claude-code) course. It lays out a set of working principles for building *with* agents rather than just prompting them, and those principles map almost one-to-one onto components I had already built in my homelab. So I will walk the principles, and for each one point at the concrete thing in my stack that implements it.

First, the physical layer, because everything below sits on it. There is one Proxmox host, `REDACTED` (`REDACTED`), on a flat `REDACTED` LAN with an internal domain of `REDACTED`. Every service is an LXC container, which means each one is cheap to snapshot, rebuild, and reason about in isolation. That is the whole homelab: one box, many small containers.

## The agentic loop: gather context, act, verify

The course's central mental model is the **agentic loop** -- an agent gathers context, takes an action, then verifies the result, and repeats. The verify step is the one people skip, and it is the one that separates a useful agent from a confident liar.

My [`scaffolding`](https://github.com/komluk/scaffolding) plugin is built around making that loop explicit and giving each phase to a different specialist. A `researcher` or `analyst` gathers context. A `developer` acts. A `reviewer` verifies against what was actually agreed. The loop is not implicit in one assistant's head; it is a routed pipeline of eleven markdown-defined agents, each with a narrow mandate. I covered the architecture in [Turning Claude Code Into a Team](/posts/multi-agent-orchestration-claude-code/), so I will not repeat it here -- the point is that the agentic loop is the organizing principle behind the whole plugin.

## Context engineering and memory: keep it fresh and condensed

The second principle is that context is a budget you spend badly by default. The course's phrasing -- keep context **fresh and condensed** -- is the thing I think about most. A bloated context window does not make an agent smarter; it dilutes its attention.

I attack this from three directions:

- **`CLAUDE.md` as always-loaded protocol.** A single, deliberately minimal file holds the routing rules and hard constraints. It is the one thing every agent sees, and I keep it short on purpose -- detail lives in linked docs, not in the protocol.
- **Semantic memory over MCP.** A self-hosted semantic-memory MCP server gives agents cross-session vector recall: they can pull up a past decision or a known gotcha by similarity query instead of re-deriving it. It is scoped per project so one repo's memory does not leak into another's.
- **File-based markdown memory as fallback.** When the MCP server is not running, agents fall back to plain markdown notes in the repo. The orchestration never *depends* on a live service.

The discipline is the same in all three: load the right thing at the right moment, and nothing else.

## Skills: reusable workflows on demand

Principle three is **skills** -- packaged, reusable knowledge that loads only when relevant. The plugin ships a library of them: API design, language patterns, testing strategy, security checklists. Each one carries a short `TRIGGER` / `SKIP` description so it loads at the right moment and stays out of the way otherwise.

```
TRIGGER: designing or reviewing a REST/GraphQL API surface
SKIP:    internal helper functions, no external contract
```

This is the same fresh-and-condensed idea applied to expertise. You do not want every guideline in every prompt. You want lazy-loaded competence -- the skill shows up when the task needs it and is invisible the rest of the time.

## MCP: integrating the stack

The fourth principle is the **Model Context Protocol** -- a standard way to plug external tools and data into an agent. MCP is where my homelab stops being a pile of containers and starts being a connected system.

Two MCP-shaped integrations matter here:

- The **semantic-memory MCP server** I mentioned above is the obvious one -- it is literally an MCP server giving agents a memory tool.
- The **self-hosted gateway** is the other half. `aiproxy` lives in REDACTED at `REDACTED`: [LiteLLM](/posts/self-hosting-llm-gateway-litellm/) for the OpenAI-compatible front door, a `REDACTED` so a `claude.ai` OAuth session stands in for an API key, and Caddy terminating TLS. Every agent and tool points at that one endpoint instead of holding its own credentials.

The gateway is the reason "no paid API key" is true. I covered the LiteLLM half in [Self-Hosting an LLM Gateway With LiteLLM](/posts/self-hosting-llm-gateway-litellm/); the OAuth-proxy layer in front of it is what lets Claude Code authenticate through my session rather than a metered key.

## Parallel development: worktrees and multi-agent workflows

The last principle is **parallelism** -- running multiple agents at once without them stepping on each other. The course pairs this with git worktrees, and that is exactly the mechanism I use.

For non-trivial work, `developer` and `architect` agents run inside isolated git worktrees, each on its own branch, so two implementation threads never collide on the working tree. The hard rule that keeps this safe: the implementer writes code, and a dedicated `gitops` agent owns every git mutation.

```
developer  (in worktree)  ->  writes code, runs tests, STOPS
gitops     (main repo)    ->  add, commit, merge, push, clean up
```

The work itself is grounded by a spec chain -- proposal, design, tasks -- so parallel changes stay scoped rather than wandering. That layer has [its own post](/posts/spec-driven-development-with-ai-agents/); here it is just the thing that makes parallelism trustworthy instead of chaotic.

## How a task actually flows through it

Concretely, here is what happens when I ask for a non-trivial change, end to end:

1. The request hits Claude Code, which authenticates through the `aiproxy` gateway over my Tailscale mesh -- so I can drive the whole thing from a laptop that is nowhere near the LAN.
2. `CLAUDE.md` loads as the always-on protocol, and the orchestrator routes the task instead of answering it.
3. Agents pull relevant history from the semantic-memory MCP server -- past decisions, known gotchas -- so they start grounded rather than blank.
4. `analyst` and `architect` produce a proposal and a design; I review *intent* before any code exists.
5. `developer` implements inside a git worktree against the task list, running tests as it goes.
6. `reviewer` checks the diff against the spec; gates block anything with a critical finding.
7. `gitops` commits, merges the worktree branch, and pushes -- to my own [Gitea](https://about.gitea.com/) instance in REDACTED, not a public host.

Every hop in that chain runs on hardware I own, through a model endpoint I control, against memory that stays in my network. The only thing crossing the boundary is the model inference itself, and even that goes through my gateway.

## Networking ties it together

None of this is useful if I can only touch it from the couch. A Tailscale mesh stitches the host, the containers, and my devices into one private network, so I reach the gateway, Gitea, and the Proxmox UI by stable names from anywhere without port-forwarding anything to the public internet. I wrote up the setup in [Setting Up Tailscale for Your Homelab Infrastructure](/posts/tailscale-setup-homelab/). The mental shift is that the homelab has no public edge to defend -- access is an identity question, and the AI stack inherits that posture for free.

It is also where the always-on services live. `scribe`, an ambient PL/EN note-taking listener that turns what it hears into notes, runs as one such service -- a small example of the larger pattern: an AI service that is just *up*, reachable over the tailnet, feeding the same memory layer everything else uses.

## Closing

The thing the course crystallized for me is that "building with Claude Code" is not a prompting skill -- it is a systems-design one. The agentic loop, context discipline, skills, MCP, and parallelism are architectural choices, and a homelab is a great place to make them deliberately because you own every layer.

My version is opinionated and self-hosted to the studs: one Proxmox box, a gateway that swaps a paid key for an OAuth session, a memory server agents actually consult, and an eleven-agent plugin that routes work through a reviewable pipeline. None of the pieces are exotic on their own. What makes it a *stack* is that they share a protocol, a memory, and a network -- and that I can read every line of how it behaves before I trust it. The plugin is open source at [github.com/komluk/scaffolding](https://github.com/komluk/scaffolding) if you want to start from the same place I did.
