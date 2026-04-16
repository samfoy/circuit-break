---
title: "Your Agent Forgets Everything Between Sessions"
date: 2026-04-16 10:00:00 -0700
categories: [AI Agents, Architecture]
tags: [memory, pi, persistence, context]
---

Your AI coding agent is stateless. Every session starts from zero. It doesn't know you prefer conventional commits. It doesn't remember that `--force-push` broke prod last Tuesday. It doesn't know your project uses Dagger for DI, or that the `cr` CLI can't handle interactive prompts in headless sessions.

You've told it all of this before. Multiple times. It learned nothing.

This is the most underrated problem in AI-assisted development. People obsess over model quality, context window size, tool selection. Meanwhile their agent makes the same mistakes on repeat because it has the memory of a goldfish.

## The real cost of stateless agents

It's not just the annoyance of re-explaining things. Stateless agents have compounding costs:

**Repeated corrections eat your time.** Every time you fix the same mistake, you're spending tokens and attention on something the agent should already know. Over hundreds of sessions, this adds up to hours.

**Mistakes recur at the worst times.** The agent that forgot "don't overwrite the default AWS profile" won't remember during an incident when you're moving fast and not double-checking every command.

**Context setup is a tax on every session.** You start each session re-establishing who you are, what you're working on, and how you like things done. This is dead weight — it's not advancing the task.

**Institutional knowledge stays in your head.** The workarounds, the team conventions, the "we tried X and it didn't work" — none of it transfers to the agent. You become a bottleneck for your own tooling.

## Three tiers of agent memory

I've been running a persistent memory system across ~2,100 coding sessions over the past two months. It stores three kinds of knowledge:

### 1. Facts — what the agent knows about you and your projects

Key-value pairs with dotted namespaces and confidence scores:

```
pref.commit_style      → "conventional commits"           (0.95)
project.rosie.language → "Java, Brazil build system"      (0.90)
user.timezone          → "Pacific (America/Los_Angeles)"  (0.95)
tool.sed.usage         → "use for daily note insertion"   (0.85)
```

Right now I have 372 of these. They cover preferences (`pref.*`), project patterns (`project.*`), tool behaviors (`tool.*`), and identity (`user.*`). Each has a confidence score — higher confidence wins when there's a conflict.

These are the things you'd put in a README for a new teammate. Except the agent writes its own README, continuously.

### 2. Lessons — corrections that stick

When you correct the agent, that correction becomes a permanent lesson:

```
❌ [cr-workflow] Keep formatting-only changes in a separate CR
   from functional changes — don't mix them in one revision

❌ [tmux-agent-launch] Never kill a tmux session named 'autoloop'
   without first checking what it's working on

✅ [incident-response] During production incidents, fix in the
   console first for immediate mitigation, then fix in code
```

I have 198 of these. Some are negative (things to avoid), some are positive (validated approaches). They're categorized by domain — `cr-workflow`, `incident-response`, `slack-mcp`, `bedrock` — so the agent knows which context they apply to.

The key insight: **lessons are bidirectional**. Negative lessons prevent repeated mistakes. Positive lessons reinforce patterns that work. Together they form a ratchet — the agent gets strictly better over time, never worse.

### 3. Events — an audit log

Every memory operation is logged: creation, updates, deletions. 617 events so far. This is less for the agent and more for me — I can trace when a memory was created, what session it came from, and whether it was auto-extracted or manually added.

## How memories are born

Two paths:

**Automatic extraction** happens at session end. If the session had at least 3 user messages, the conversation is sent to a lightweight LLM call that extracts structured knowledge:

```
Session conversation → LLM consolidation → Structured JSON → Store
```

The extraction prompt is specific about what to keep and what to discard. It's told to extract only lasting preferences and corrections, not ephemeral task details. "Today we worked on the auth module" is not a memory. "The auth module uses Dagger DI with constructor injection" is.

Each extracted fact needs confidence ≥ 0.8 to be stored. Lessons are deduplicated using Jaccard similarity — if a new lesson is ≥ 70% similar to an existing one, it's rejected as a duplicate.

**Manual storage** happens when I (or the agent) explicitly calls `memory_remember`:

```
memory_remember({
  type: "lesson",
  rule: "Bedrock InvokeModel has a 5MB limit per image in base64",
  category: "bedrock",
  negative: false
})
```

Manual memories get a higher confidence score (0.95 vs 0.8 for auto-extracted). They're for things the consolidator wouldn't catch — API limits discovered through debugging, workarounds for specific tools, non-obvious patterns.

## How memories are recalled

At the start of every session, before the agent sees your first message, relevant memories are injected into the system prompt as a `<memory>` block:

```xml
<memory>
## Relevant Memory
- commit_style: conventional commits
- rosie.language: Java, Brazil build system
- timezone: Pacific (America/Los_Angeles)

## Learned Corrections
- DON'T: Keep formatting-only changes in a separate CR [cr-workflow]
- DON'T: Never kill a tmux session named 'autoloop' without checking [tmux]

## Validated Approaches
- During production incidents, fix in console first [incident-response]
- When creating a CR, always include a Taskei task link [cr-workflow]
</memory>
```

Two injection modes:

**Selective** (when the user's first message is available): search semantic memory using the prompt as a query, pull in project-scoped facts based on the working directory, and always include all lessons.

**Fallback** (no prompt yet): dump top entries by prefix — preferences, project context, tool preferences, lessons.

The entire block is capped at 8KB. That's roughly 2,000 tokens — a small fraction of the context window, but enough to carry forward the most important knowledge.

## What NOT to remember

This matters more than what to remember. Early versions of the system hoarded everything — file paths, git history, debugging commands, "today we fixed X" summaries. The memory filled up with noise that was either wrong (file moved), redundant (derivable from the project), or useless (one-off commands).

The consolidation prompt now explicitly filters these out:

- **File paths and project structure** — derivable by reading the project. The project is the source of truth, not a cached memory of what the project looked like last week.
- **Git history** — `git log` is authoritative. A memory saying "we merged feature-X on March 15" adds nothing.
- **Debugging recipes** — "when error X, run command Y" is too specific to generalize. The fix is in the code; the commit message has context.
- **Activity summaries** — "today we worked on the auth module" is ephemeral. Instead, extract what was surprising or non-obvious about it.

The filter also runs programmatically on extracted entries. Keys containing `filepath`, `git.history`, or `current_task` are rejected before they hit the store. Lesson rules starting with "we fixed" or "we deployed" are discarded.

The goal: **memory should contain things the agent can't derive from the current state of the world.** Preferences, corrections, non-obvious patterns, validated workarounds. Everything else is noise.

## Staleness and drift

Memories go stale. A preference from 90 days ago might no longer apply. A project pattern from before a major refactor could be actively misleading.

Two defenses:

**Staleness indicators** on injected memories. Facts older than 30 days get an age tag; facts older than 90 days get a warning:

```
rosie.deploy_target: personal account ⚠️ 95d old — verify before acting
```

**A drift caveat** appended to every memory block:

> Memory records can become stale. If a memory names a file, function, or flag — verify it still exists before recommending it. "The memory says X exists" is not the same as "X exists now."

This shifts the agent from "the memory says X, so X is true" to "the memory says X, let me check." It's a small prompt addition but it prevents the most dangerous failure mode: confidently acting on outdated information.

## What I've learned from 2,100 sessions

**Lessons are more valuable than facts.** Facts are nice-to-have context. Lessons change behavior. A single lesson like "don't overwrite the default AWS profile" has prevented dozens of mistakes that would each cost 5-10 minutes to recover from.

**The consolidator needs a strong negative list.** Without explicit "don't extract this" rules, it fills memory with derivable junk. I spent more time tuning the rejection filters than the extraction prompt.

**Confidence-based conflict resolution works.** When the consolidator extracts `project.rosie.language = "Java"` at 0.8 confidence, and I manually set it to `"Java, Brazil build system"` at 0.95, the manual version wins. Simple rule, no complexity.

**Deduplication is essential.** Without Jaccard similarity checking, you get 15 variants of the same lesson. "Use conventional commits." "Always use conventional commits." "Commit messages should follow conventional commits format." The 0.7 threshold catches these without being so aggressive it rejects genuinely different lessons.

**8KB is enough.** I worried the memory block would need to be huge. It doesn't. 372 facts and 198 lessons compress into a focused block that fits comfortably. The selective injection mode helps — you don't need all memories in every session, just the relevant ones.

## The feedback loop

The real power isn't any single component — it's the loop:

```
Session → Correction → Lesson stored → Future sessions improved
                                              ↓
                                    Fewer corrections needed
                                              ↓
                                    Faster, more reliable work
```

Session 1: The agent force-pushes. You correct it.
Session 50: The agent hasn't force-pushed in 49 sessions because the lesson is injected every time.
Session 100: You've forgotten you ever had this problem.

This is the compound interest of agent memory. Each correction makes every future session slightly better. Over hundreds of sessions, the agent becomes meaningfully personalized — not through fine-tuning or RLHF, but through a SQLite database and 8KB of injected context.

## Try it

[pi-memory](https://github.com/samfoy/pi-memory) is an extension for [pi](https://pi.dev/). Install with:

```bash
pi install pi-memory
```

It's ~500 lines of TypeScript, a SQLite database, and an LLM call at session end. No cloud services, no training data uploaded, no vendor lock-in. Your memories stay in `~/.pi/memory/memory.db` on your machine.

The pattern works in any agent framework. The core requirements are:
1. A session lifecycle with start/end hooks
2. A way to inject text into the system prompt
3. An LLM call for consolidation (or manual entry)
4. A local store (SQLite, JSON, whatever)

If your agent framework has those four things, you can build persistent memory in a weekend. If it doesn't have them, that's a more interesting problem.

[GitHub](https://github.com/samfoy/pi-memory) · [npm](https://www.npmjs.com/package/pi-memory)
