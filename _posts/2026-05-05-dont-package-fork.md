---
title: "Don't Package. Fork."
date: 2026-05-05 10:00:00 -0700
categories: [AI Agents, Tools]
tags: [pi, extensions, supply-chain, npm, agents, workflow]
---

A pi extension is an npm package that runs code inside your coding agent's process. When the LLM calls one of its tools, that tool is a function with the same permissions as your agent. Whatever your agent can read, the extension can read. Whatever APIs your agent can call, the extension can call.

The community registry has a growing pile of extensions for things like web search, web fetch, clipboard copy, screenshot. Several do nearly the same thing. Most are under 200 lines. Each one is an npm install and a new trust relationship with a stranger on the internet.

[A Reddit thread this week](https://reddit.com/r/PiCodingAgent/comments/1t4ja1e/are_we_overpackaging_simple_pi_workflows/) made a softer version of this argument: the community should share ideas and tradeoffs rather than reusable modules for trivially simple things. I want to push further. The right default for a coding agent extension isn't install. It's fork.

## What you're actually installing

A coding agent that touches your shell, your editor, and your accounts is a different threat model than a library you pull into a web server. The web server has a narrow job. Your agent has root over your workflow.

When pi loads an extension, it imports a JS module and exposes its tools to the LLM. Those tools run in the same process as the agent, with the same environment. If you've given your agent a GitHub token, the extension has it too.

npm has a well-documented history of supply chain attacks and 2025 was ugly. The [Shai-Hulud worm in September](https://www.cisa.gov/news-events/alerts/2025/09/23/widespread-supply-chain-compromise-impacting-npm-ecosystem) compromised over 500 packages by planting a malicious `postinstall` script that harvested tokens and secrets. A [second variant in November](https://securitylabs.datadoghq.com/articles/shai-hulud-2.0-npm-worm/) hit 796 packages representing over 20 million weekly downloads. The pattern is the usual one: a maintainer account gets compromised, a minor release ships with a postinstall that runs on every dev machine that updates, and by the time anyone notices the token inventory is already gone.

A coding agent extension is the highest-leverage place this attack can land. Not because the extension itself is malicious (usually), but because its blast radius covers everything the agent can touch: SSH keys, cloud credentials, git history, whatever cookies your browser-harness is holding.

## The 100-LOC problem

Here's the Reddit OP's actual setup:

> "I quickly realized I needed web search. I initially used a DuckDuckGo MCP setup via Docker, but after digging into it, it felt like unnecessary abstraction since it mostly aggregates results from existing engines. I then simplified things to two lightweight tools: one for search and one for fetch... The full setup is still roughly ~100 LOC."

That's it. One function that calls Perplexity Sonar Pro and returns citations, a second function that fetches those specific URLs. No Docker, no MCP, no npm dependency.

A lot of "extensions" are in this category: screenshot, clipboard paste, save-URL-to-Obsidian, wrap-this-shell-command. Thin wrappers over a system binary or a single HTTP call. These shouldn't be packages. They're scripts that belong in your repo.

The economics of a package only work when the code inside is hard to get right or benefits from many users finding bugs over years. A 100-LOC wrapper around `curl` doesn't clear that bar. You're paying in trust for convenience that isn't worth it.

## Fork the idea, not the code

The pattern I follow now:

1. See a package that does something useful.
2. Read the code. Not skim. Read.
3. Extract the essence. The actual idea usually fits in a paragraph.
4. Have pi rebuild it in my own repo, tuned to my workflow.

"Fork" here is loose. I'm not literally git-forking every repo. I'm treating the original as a reference implementation and writing my own version that lives next to my other code.

```
# Not this
npm install @someone/pi-web-search

# This
mkdir -p ~/.pi/agent/tools/web-search
# have pi read the reference repo
# have pi write my own version inline
# commit it next to my AGENTS.md
```

What I get out of this: code I can change without waiting for upstream, zero postinstall scripts running in my agent's process, and a version that already knows about my vault paths and skill conventions. No lock file entry for a stranger's code I'd have to audit forever.

A concrete version. My session search extension doesn't chunk sessions the way other people's do, because I index by tool call boundaries not token windows. My memory extension uses dotted keys because that's how my queries are shaped. My knowledge search is hybrid BM25 + vector with RRF fusion, not pure vector, because a lot of what I search for is exact tokens: error messages, filenames, proper nouns. None of these would be right if I'd grabbed someone else's package off the shelf. The shape of the tool has to match the shape of the workflow.

## Where packages earn their slot

This isn't absolutism. I install some packages. I also publish some. The line I try to hold:

Packages are reasonable when the code inside would take real time to get right. LSP adapters are a good example. The Language Server Protocol has dozens of message types, streaming semantics, initialization handshakes, and two decades of editor-implementation bugs encoded in the reference clients. I'm not rebuilding that. I'll install pi-lsp-extension and be grateful.

Provider adapters are the same story. Bedrock streaming, Kiro auth, OpenAI token accounting and rate limits. These take days to get right and years to keep working. Install them.

The pi harness itself is in this category. TUI, extension loader, skill system, session format. I didn't write that, Mario did, and I'd be an idiot to reimplement it.

The heuristic I use: if writing my own version would take more than a day, installing is defensible. Under a day, the package is charging me rent in trust I shouldn't be paying.

## The minimum bar either way

For any package that does earn a slot, read the source before you install. Check the dependency tree. Look at the postinstall. Pin the version and audit on every bump.

Supply chain attacks mostly don't hide. Shai-Hulud planted a `bundle.js` and declared it in `postinstall` right there in `package.json`. Visible to anyone who looked. The attack worked because almost nobody did. For agent extensions the cost of looking is trivial compared to the blast radius of not looking.

One extra rule I hold myself to: if a package's install would put a new binary on `$PATH` or register a background service, I read the install script before I run it. That's saved me twice. Both times the package wasn't malicious, just doing something I didn't want, a telemetry daemon one time and a login helper the other. Would've been annoying to discover three months later.

## I publish packages too

I ship pi-memory, pi-session-search, pi-knowledge-search, pi-lsp-extension. I just bundled three of them into a meta-package called pi-total-recall. I'm part of the thing I'm describing.

What I try to do differently: no postinstall scripts, no dependencies I haven't audited, source that's roughly the size you'd expect for the job, docs that say what the tool can touch. The README links the repo at the top, not behind a click.

I also fully expect most readers to not install my packages. I expect them to read the source, copy the interesting bits into their own repo, and throw the rest away. If my implementation doesn't match your workflow, the ideas are still useful and the code shouldn't be running in your agent's trust boundary.

## The punchline

The Reddit OP was right and it generalizes further. Share ideas, approaches, the alternatives you evaluated and rejected. That's what blog posts and discussion threads are for, and it's what coding agents happen to be uniquely good at: describe an idea in prose, have the agent write a version that fits your code.

Don't share binaries if you don't have to. Don't install them if you don't have to.

Your agent is your code. Forking is the default. Packages are the exception, earned by complexity you can't cheaply absorb yourself.
