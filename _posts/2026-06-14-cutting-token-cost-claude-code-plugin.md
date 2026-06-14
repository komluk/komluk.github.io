---
title: "Cutting Token Cost in a Claude Code Plugin"
author:
  name: Łukasz Komosa
  link: https://github.com/komluk
date: 2026-06-14 09:00:00 +0000
categories: [Claude Code, AI]
tags: [claude-code, agents, tokens, optimization]
render_with_liquid: false
---

I maintain [`scaffolding`](https://github.com/komluk/scaffolding), a multi-agent orchestration plugin for Claude Code. It ships 11 agents, 34 skills, 16 commands, and 9 hooks. As the plugin grew, every new session was starting to feel heavier than it should — a noticeable chunk of the context window was already spent before I typed a single instruction.

When I actually measured where those tokens were going, almost all of it was duplication. The same routing rules, the same safety checks, the same protocol text, injected over and over from different places. This post walks through three concrete fixes I shipped, and the lesson that ties them together.

## The thing nobody measures

In an agent system, the expensive surface is not the size of your repo. It is the context that gets injected *on every turn*. Two sources dominate:

- **SessionStart hooks** — run on every session start, resume, and compaction event, and their `additionalContext` is prepended to the model's context.
- **Agent system prompts** — every spawned subagent loads its full definition as a system prompt.

Both are paid repeatedly. A 1,000-character block in a SessionStart hook is not a one-time cost; it is a per-session-event cost. So that is where I started looking.

> All token figures below are estimates derived from character counts (roughly 4 characters per token). I did not run a formal benchmark.

## Fix 1: the SessionStart hook was re-injecting the whole protocol

The plugin's `SessionStart` hook was emitting a `~3,800`-character `additionalContext` block on every session start. It contained the full agent table, the decision tree, the delegation format, and the key rules.

The problem: that exact protocol *already* lives in the repo's `CLAUDE.md`. `CLAUDE.md` is auto-loaded by Claude Code and survives compaction. So the hook was paying roughly `800-950` tokens of pure duplication on every single session-start event — re-stating something the model already had.

The fix was to make the hook a thin pointer instead of a full copy. Here is the shape of the hook's JSON output, before:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "## Protocol\n\n| Agent | When to Use |\n| analyst | ... |\n| architect | ... |\n[~3,800 chars: full agent table, decision tree, delegation format, key rules]"
  }
}
```

And after:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Routing table and agent list are in CLAUDE.md (auto-loaded). Core rules: never edit code/docs directly — always delegate; gitops handles all commits/merges/pushes; tech-writer owns docs."
  }
}
```

The model still gets the load-bearing reminders, but the full table is fetched from the place that already carries it for free.

## Fix 2: a duplicate hook registration

While auditing the hooks I found a stale `settings.json` that carried its *own* `SessionStart` definition. It pointed at broken `.claude/hooks/...` paths and duplicated the registration that already exists in `plugin.json`.

In setups where both fired, the protocol got injected twice — roughly `1,900` tokens on a compaction event instead of the intended one-time block. Worse, the broken paths meant the duplicate sometimes failed loudly for no benefit.

The fix is the boring correct one: a single source of truth. The hook is registered in `plugin.json` only, and I removed the duplicate block from `settings.json` entirely. If you have ever copied a hook definition between config files "just to be safe," go check that you are not double-injecting.

## Fix 3: ~5,500 tokens of byte-identical agent boilerplate

This was the big one. Six of the agents each carried the same comms-protocol block — `SendMessage` recipient validation plus the `worktreePath` / CWE-59 path-safety checks. Not similar. *Byte-identical.*

Because every spawned subagent loads its full definition as a system prompt, this duplication was multiplied by usage. A session that happened to use 3 of those 6 agents paid roughly `2,700` tokens just for three copies of the same security text.

The instinct is to delete the duplicated block, but that block is security-critical — it is the logic that stops an agent from messaging the wrong recipient or following a malicious symlink out of its worktree. Deleting it outright would be a regression.

So instead I extracted the shared content into a single `agent-comms` skill, which is loaded on demand rather than baked into every prompt. Each agent keeps a compact inline rule pointing at it, something like:

```markdown
## Comms
Validate SendMessage recipients and worktreePath safety (CWE-59) per the
`agent-comms` skill. Never message an unknown recipient; never resolve a
worktree path that escapes its root.
```

Three lines, present in the prompt where it cannot be missed, with the full procedure available the moment an agent needs it. The skill is loaded once when relevant instead of six times unconditionally. Net result: about `6,785` characters of duplicated boilerplate removed from the agent definitions.

## The lesson: pick the right home for shared knowledge

All three fixes are the same mistake wearing different clothes — the same knowledge copied into a place that gets paid for repeatedly. The fix in each case was to move it to a cheaper home:

- **Auto-loaded project files** (`CLAUDE.md`) — loaded once, survive compaction. The right place for routing rules and standing conventions. The hook just points here.
- **On-demand skills** — loaded only when relevant. The right place for detailed procedures like the comms protocol that not every agent needs on every turn.
- **Single registration** — config that lives in exactly one file. No "safety" copies.

What does *not* belong on the per-turn surface is a full copy of anything that already exists elsewhere. Hooks should nudge, not narrate. Agent prompts should state the rule and delegate the detail.

## Make measuring a habit

The reason this duplication survived so long is that nobody was looking at the per-session injection footprint. It is easy to add a hook or copy a prompt block; it is invisible until you add up the characters.

So now I treat the injected footprint as a number worth tracking. A rough `wc -c` over what each hook emits and what each agent prompt contains is enough to catch the worst offenders:

```bash
# Roughly: how many characters does each hook inject?
for h in hooks/*.json; do echo "$h: $(jq -r '.additionalContext // empty' "$h" | wc -c) chars"; done
```

You do not need precise token accounting. You need to notice when the same 3,000 characters show up in three different places.

## Takeaways

- The expensive surface in an agent system is whatever gets **injected on every turn** — session hooks and agent system prompts, not your repo size.
- Move standing rules into **auto-loaded project files** (`CLAUDE.md`); they survive compaction and cost nothing extra per session.
- Put detailed, conditional procedures into **on-demand skills** instead of every agent prompt.
- Register hooks in **exactly one place** to avoid silent double-injection.
- Never delete security-critical text to save tokens — **relocate** it and leave a compact inline pointer.
- **Measure** the per-session injection footprint as a routine; duplication is invisible until you count characters.
