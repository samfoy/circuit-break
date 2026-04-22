---
title: "Your Agent Needs Three Kinds of Memory, Not One"
date: 2026-04-22 10:00:00 -0700
categories: [AI Agents, Architecture]
tags: [memory, pi, persistence, context, session-search, knowledge-search]
---

I wrote about [persistent memory for AI agents](/posts/your-agent-forgets-everything-between-sessions/) last week. Facts and lessons, extracted from sessions, injected into future ones. 5,000+ people installed it. It works.

But it's one layer of a three-layer problem.

## What memory doesn't cover

pi-memory stores what you've *told* the agent. Preferences, corrections, project patterns. It's good at "use conventional commits" and "don't force-push to main." It's bad at "how did we fix that Lambda timeout last month?" and "what did my notes say about the YouTube API rate limits?"

Those are different retrieval problems. The first needs your session history. The second needs your local files. Neither one lives in a key-value store.

## Layer 1: Facts and lessons (pi-memory)

This is what people mean when they say "agent memory." It answers: *what does the agent know about me?*

```
pref.commit_style     → "conventional commits"
project.rosie.di      → "Dagger, constructor injection"
user.timezone         → "Pacific"
```

Plus lessons: things the agent got wrong that it shouldn't get wrong again. 171 facts and 64 lessons across 300+ sessions, injected into every new conversation. I covered this in detail in [the last post](/posts/your-agent-forgets-everything-between-sessions/).

It's the most popular layer. Also the most bounded.

## Layer 2: Session history (pi-session-search)

Your coding sessions contain answers you've already found. Debugging sessions, architecture decisions, failed experiments, working solutions. All sitting in JSONL files on disk, unsearchable.

pi-session-search indexes those sessions with embeddings and exposes them through `session_search`, `session_list`, and `session_read`. The agent can search its own past.

Here's when it matters. I was setting up a bundled npm package and kept hitting a "Hard link is not allowed" error on publish. Felt new. Ran:

```
session_search("npm publish bundledDependencies hardlink")
```

No prior session (it actually was new). But when I solved it, the fix got indexed. Next time anyone on my machine hits that error, the agent finds it in seconds instead of re-debugging from scratch.

Software work is repetitive in ways that feel unique each time. The deploy script that fails on a specific flag. The test that flakes under certain conditions. The dependency that needs a particular version pin. You've solved these before. Your sessions have the solutions.

Session search answers: *what has the agent done for me before?*

## Layer 3: Knowledge base (pi-knowledge-search)

This connects the agent to your files. Not code (it can already read your codebase), but everything else. Notes, documentation, research, meeting summaries, project plans.

I keep an Obsidian vault with ~800 files. Research notes, project specs, contact info, recipes, travel plans. pi-knowledge-search indexes that directory with embeddings and watches for changes.

```
knowledge_search("YouTube API quota limits")
```

Returns the relevant section from my research notes, with the exact numbers and workarounds I wrote down months ago.

Without this, the agent has two choices: search the web (generic, possibly outdated) or ask me (interrupts my flow). With it, the agent can check my notes first. Often the answer is already there because past-me already researched it.

Knowledge search answers: *what do I already know about this?*

## Three layers, one install

Each layer is useful alone. Together they cover the context an agent actually needs:

| Layer | Answers | Example |
|-------|---------|---------|
| Memory | What do you prefer? | "Use conventional commits" |
| Sessions | What have we done? | "Fixed this timeout by adjusting retry config" |
| Knowledge | What do you know? | "YouTube API quota is 10,000 units/day" |

I published [pi-total-recall](https://github.com/samfoy/pi-total-recall) to bundle all three into one install:

```bash
pi install pi-total-recall
```

One command. You get `memory_search`, `memory_remember`, `session_search`, `session_list`, `session_read`, and `knowledge_search`.

## The adoption gap

Here's what prompted this post. Current npm numbers:

| Package | Monthly downloads |
|---------|----------------:|
| pi-memory | 5,143 |
| pi-session-search | 186 |
| pi-knowledge-search | 31 |

Memory gets 28x the downloads of session search and 166x knowledge search.

I get why. "The AI remembers things about you" sells itself in one sentence. "Semantic search over your past coding sessions" takes a paragraph to explain and weeks of sessions to demonstrate.

But in my own usage, the impact hierarchy is reversed. Knowledge search saves me the most time because it connects the agent to months of accumulated research and notes. Session search is second because it prevents re-solving problems. Memory is third. Useful, but it's storing isolated facts that I could also just put in an AGENTS.md file.

The most useful layers are the ones nobody's installing.

## What this looks like in practice

Today I was building this very package (pi-total-recall). The agent had:

1. **Memory** knew I use conventional commits, that my npm username is samfp, and that pi packages need the `pi-package` keyword
2. **Session search** could find past sessions where I'd published npm packages, including the `bundledDependencies` pattern and the hardlink workaround
3. **Knowledge search** had my vault notes on pi's extension system and package structure

No digging through old terminals. No re-reading docs. It was just there.

## Setup

```bash
pi install pi-total-recall
```

For knowledge search, run the setup command in pi:

```
/knowledge-search-setup
```

Or edit `~/.pi/knowledge-search.json` directly:

```json
{
  "dirs": ["~/Documents/Notes"],
  "provider": {
    "type": "openai",
    "model": "text-embedding-3-small"
  }
}
```

Session search and memory work out of the box. Sessions get indexed automatically. Memory starts learning from your first conversation.

The individual packages still exist if you only want one:

```bash
pi install pi-memory
pi install pi-session-search
pi install pi-knowledge-search
```

All three are MIT licensed and run locally. Your data stays on your machine.

[GitHub](https://github.com/samfoy/pi-total-recall) · [npm](https://www.npmjs.com/package/pi-total-recall)
