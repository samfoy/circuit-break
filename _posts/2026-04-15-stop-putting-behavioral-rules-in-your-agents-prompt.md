---
title: "Stop Putting Behavioral Rules in Your AI Agent's Prompt"
date: 2026-04-15 10:00:00 -0700
categories: [AI Agents, Techniques]
tags: [guardrails, steering, pi]
---

Every AI coding agent has rules: "never force push," "use conventional commits," "don't run dev servers that block the agent." Most people put these in the system prompt.

This works 95% of the time. The other 5%, the model forgets — especially in long sessions with context pressure. And that 5% is when the damage happens.

## The fix: deterministic hooks, not probabilistic prompts

Instead of asking the model to remember rules, intercept tool calls before execution and enforce rules programmatically:

```
Agent wants to run: git push --force origin main
↓
Hook intercepts → regex matches "git push.*--force"
↓
Block returned: "Force push rewrites remote history.
                 Use --force-with-lease or create a new commit."
↓
Agent adjusts approach
```

Zero tokens spent. Zero prompt drift. 100% enforcement.

This isn't a new idea — it's how every production system works. You don't put "don't drop the database" in a README and hope developers read it. You revoke the permission. Steering hooks are the same principle applied to AI agents.

## The pattern

A steering rule has three parts:

1. **Pattern** — regex that matches a violation (e.g. `\bgit\s+push\b.*--force`)
2. **Exemptions** — optional regex that cancels the match (e.g. `--force-with-lease`)
3. **Reason** — message shown to the agent explaining why it was blocked and what to do instead

Rules are evaluated on every tool call. If the pattern matches and no exemption applies, the call is blocked before execution. The agent sees the reason and adjusts.

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
      "unless": "(--profile|AWS_PROFILE=|\\baws\\s+(sts\\s+get-caller-identity|configure)\\b)",
      "reason": "Always use --profile or AWS_PROFILE with aws CLI commands."
    }
  ]
}
```

## The override escape hatch

Hard blocks are frustrating when the agent has a legitimate reason. The solution: an override mechanism that allows the action but logs it for audit.

```bash
git push --force origin main  # steering-override: no-force-push — deploying hotfix to unblock prod
```

The override is allowed through but recorded. For truly catastrophic actions (like `rm -rf /`), overrides can be disabled entirely.

This mirrors how production systems work: you can break glass, but it's logged and reviewed.

## Where this pattern applies

Both [pi](https://pi.dev/) and [Claude Code](https://code.claude.com/) support before-tool hooks. The implementations differ:

- **Claude Code**: shell scripts that read JSON from stdin, return JSON on stdout. Powerful but requires writing and maintaining scripts per rule.
- **pi (steering-hooks)**: declarative JSON rules with regex pattern/unless/requires conditions. No scripts — just a config file.

But the principle is universal. Any agent framework with a tool-call interception point can implement this. The key insight:

> If a behavioral rule can be expressed as a pattern match on tool input, it should never be in the prompt. Prompts are for reasoning. Hooks are for enforcement.

## What makes a good steering rule?

Not every rule belongs in a hook. Good candidates:

- **Deterministic** — can be expressed as a regex on the command/path/content
- **Universal** — applies regardless of context (no "unless we're debugging")
- **High-consequence** — the cost of violation is high (data loss, broken deploys)
- **Frequently violated** — the model keeps forgetting this one

Bad candidates:

- **Context-dependent** — "use library X for HTTP calls" (depends on the project)
- **Subjective** — "write clean code" (can't be pattern-matched)
- **Low-stakes** — "prefer const over let" (not worth the interruption)

## Try it

For pi users:

```bash
pi install @samfp/pi-steering-hooks
```

For everyone else: look at your agent's system prompt. Find the rules that are pattern-matchable and high-consequence. Move them to hooks. Your future self will thank you the first time a hook catches something the prompt would have missed.

[GitHub](https://github.com/samfoy/pi-steering-hooks) · [npm](https://www.npmjs.com/package/@samfp/pi-steering-hooks)
