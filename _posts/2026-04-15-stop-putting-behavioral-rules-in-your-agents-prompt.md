---
title: "Stop Putting Behavioral Rules in Your AI Agent's Prompt"
date: 2026-04-15 10:00:00 -0700
categories: [AI Agents, Techniques]
tags: [guardrails, steering, pi]
---

Every AI coding agent has rules: "never force push," "use conventional commits," "don't run dev servers that block the agent." Most people put these in the system prompt.

This works most of the time. Then it doesn't, and the agent drops your production database.

## The prompting treadmill

You start with a simple prompt. The agent does something wrong, so you add a rule. It does something else wrong, so you add another. Before long your system prompt is a wall of "DO NOT" statements that the model usually follows, sometimes ignores, and occasionally interprets creatively.

You test a dozen times on your laptop and everything looks great. Then the agent runs hundreds of times and the long tail of unexpected inputs finds every gap.

The Strands Agents team at AWS [ran the numbers](https://strandsagents.com/blog/steering-accuracy-beats-prompts-workflows/): across 600 evaluation runs, prompt-based instructions achieved **82.5% accuracy**. That means roughly 1 in 5 runs, the agent ignored at least one rule. The most common failures? Skipping required steps (43% of failures) and forgetting follow-up actions (40%).

For low-stakes tasks, 82.5% is fine. For `git push --force` or `rm -rf /`, it's not.

## Hooks beat prompts for enforcement

The fix is old news in every other domain: don't tell someone not to do something — make it impossible. You don't put "don't drop the database" in a README. You revoke the permission.

Steering hooks apply this principle to AI agents. Instead of asking the model to remember rules, intercept tool calls before execution and enforce rules programmatically:

```
Agent wants to run: git push --force origin main
↓
Hook intercepts → pattern matches "git push.*--force"
↓
Block: "Force push rewrites remote history.
        Use --force-with-lease or create a new commit."
↓
Agent adjusts approach
```

Zero tokens spent on enforcement. Zero prompt drift. 100% reliability.

In the Strands evaluation, steering hooks achieved **100% accuracy** across all 600 runs — compared to 82.5% for prompts and 80.8% for graph-based workflows. And they used 66% fewer input tokens than the next most accurate approach (detailed SOPs at 99.8%).

## Anatomy of a steering rule

A good steering rule has three parts:

**Pattern** — what to catch. A regex on the tool input.
```
\bgit\s+push\b.*--force
```

**Exemptions** — what to allow through. An optional regex that cancels the match.
```
--force-with-lease
```

**Reason** — what to tell the agent. This is the guidance that replaces the prompt instruction.
```
Force push rewrites remote history and can destroy teammates' work.
Use --force-with-lease if you must, or create a new commit.
```

Rules compose as a simple list. Here's a config with two rules — one for git safety, one for AWS credential hygiene:

```json
{
  "rules": [
    {
      "name": "no-force-push",
      "tool": "bash",
      "field": "command",
      "pattern": "\\bgit\\s+push\\b.*--force",
      "reason": "Force push rewrites remote history. Use --force-with-lease or create a new commit."
    },
    {
      "name": "aws-requires-profile",
      "tool": "bash",
      "field": "command",
      "pattern": "\\baws\\s+[a-z]",
      "unless": "(--profile|AWS_PROFILE=)",
      "reason": "Always use --profile or AWS_PROFILE with aws CLI commands."
    }
  ]
}
```

The `unless` field is the exemption — if it matches, the rule doesn't fire. There's also a `requires` field for AND conditions (pattern must match AND requires must match).

## The override escape hatch

Hard blocks are frustrating when the agent has a legitimate reason. Production systems solve this with break-glass mechanisms: you can override, but it's logged and reviewed.

Same idea here. When a rule fires, the agent can retry with an override comment:

```bash
git push --force origin main  # steering-override: no-force-push — deploying hotfix to unblock prod
```

The override is allowed through but logged to the session for audit. For truly catastrophic actions (like `rm -rf /`), overrides can be disabled entirely with `noOverride: true`.

## What makes a good steering rule?

Not every rule belongs in a hook. The Strands post makes an important distinction: steering is for rules that are **easy to express in code but hard to enforce in a prompt**.

Good candidates:

- **Deterministic** — expressible as a pattern match on the command, path, or content
- **High-consequence** — the cost of violation is real (data loss, broken deploys, security issues)
- **Frequently violated** — the model keeps forgetting this one despite prompt instructions
- **Universal** — applies regardless of context

Bad candidates:

- **Context-dependent** — "use library X for HTTP calls" (depends on the project)
- **Subjective** — "write clean code" (can't be pattern-matched)
- **Nuanced** — "maintain a professional tone" (use an LLM judge for this instead)

Leave the nuanced stuff in the prompt. Move the deterministic, high-consequence rules to hooks.

## The principle

> If a behavioral rule can be expressed as a pattern match on tool input, it should never be in the prompt. Prompts are for reasoning. Hooks are for enforcement.

This isn't about replacing prompts — it's about using each mechanism for what it's good at. Prompts guide reasoning and tone. Hooks enforce invariants. Together, they cover more ground than either alone.

I built [pi-steering-hooks](https://github.com/samfoy/pi-steering-hooks) to make this easy for [pi](https://pi.dev/) users — declarative JSON rules, no scripts to write. But the pattern works in any agent framework with a tool-call interception point. Look at your agent's system prompt, find the rules that are pattern-matchable and high-consequence, and move them to hooks.

[GitHub](https://github.com/samfoy/pi-steering-hooks) · [npm](https://www.npmjs.com/package/@samfp/pi-steering-hooks)
