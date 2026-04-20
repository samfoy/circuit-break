---
title: "Lie to Your Agent (It'll Write Better Code)"
date: 2026-04-20 10:00:00 -0700
categories: [AI Agents, Techniques]
tags: [prompting, testing, adversarial, code-review]
---

"Write tests for this function" produces mediocre tests. "This function has a bug — find it" produces tests that actually catch things.

Same agent. Same code. Different framing. Wildly different results.

## The generation trap

When you ask an agent to write tests, it enters generation mode. It reads the function signature, infers the happy path, and produces tests that confirm the code does what it already does. These tests pass immediately and catch nothing.

The agent isn't lazy — it's doing exactly what you asked. "Write tests" implies the code is correct and needs coverage. The agent obliges with tests shaped like the implementation.

You end up with a test suite that's a mirror of the source code. Every assertion is a tautology. The tests pass today and will pass tomorrow even if someone introduces a subtle off-by-one, a null dereference on an edge case, or a race condition under load.

## Investigation mode finds real bugs

Tell the agent something is broken — even when it isn't — and it shifts into a fundamentally different mode. Instead of confirming behavior, it's hunting for inconsistencies. It reads more carefully. It traces data flow. It asks "what if this input is null?" instead of "what does this input look like in the happy path?"

```
❌ "Write unit tests for the OrderProcessor class"

✅ "The OrderProcessor has a bug that surfaces when orders
    contain duplicate line items. Find it and write a
    regression test."
```

The first prompt produces 5 tests that exercise the obvious paths. The second produces a test that actually constructs duplicate line items, traces the deduplication logic, and — often — finds a real bug you didn't know about.

You lied about knowing there's a bug. The agent doesn't know that. It just works harder.

## Adversarial personas multiply the effect

The technique scales beyond testing. When reviewing your own design docs or code, give the agent a persona with a specific axe to grind:

```markdown
Review this design doc from TWO perspectives:

1. A skeptical engineering manager who cares about operational
   risk, staffing feasibility, and timeline confidence.

2. A pedantic senior engineer who challenges every technical
   claim, questions cost assumptions, and finds gaps in the
   evaluation methodology.

Be adversarial. Find real problems, not surface-level style
issues. If the methodology is flawed, say so. If the cost
model has hidden assumptions, call them out.
```

Without the persona framing, you get "this looks good, consider adding more detail to section 3." With it, you get specific questions your actual reviewers will ask — before the review meeting.

The key phrase is **"be adversarial."** It's permission to stop being helpful and start being critical. Agents default to agreeable. You have to explicitly override that.

## The pattern: frame tasks as investigations, not generations

This works across the board:

| Generation framing | Investigation framing |
|---|---|
| "Write tests for X" | "X has a concurrency bug — find it" |
| "Review this PR" | "This PR will break something in production — what?" |
| "Document this API" | "A new engineer will misuse this API — how?" |
| "Check this config" | "This config will cause an outage at scale — why?" |

The investigation framing forces the agent to:
- **Read more carefully** — it can't skim when it's hunting
- **Consider edge cases** — it needs to find the failure, not describe the success
- **Challenge assumptions** — the code is guilty until proven innocent
- **Trace data flow** — surface-level reading won't find the "bug"

## When the agent actually finds something

Here's the thing: about 30% of the time, the agent finds a real issue. You lied about knowing there's a bug, but there was one anyway. The adversarial framing just made the agent look hard enough to find it.

The other 70% of the time, you get tests and reviews that exercise the actual edge cases — the ones that will break in six months when someone changes an upstream dependency or passes unexpected input.

Either way, you win.

## The cost is one sentence

This technique costs nothing to implement. No tools, no frameworks, no configuration. You change one sentence in your prompt and get meaningfully better output.

Next time you're about to type "write tests for," stop. Type "there's a bug in" instead. Your test suite will thank you.
