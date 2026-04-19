---
title: "Turning a Coding Agent into a Personal Assistant"
date: 2026-04-19 14:00:00 -0700
categories: [AI Agents, Architecture]
tags: [pi, assistant, skills, automation, openclaw]
---

I used to run [OpenClaw](https://openclaw.dev) as my personal assistant. It read my iMessages and extracted tasks into Things 3. It checked Gmail hourly and appended summaries to my Obsidian daily notes. It sent me a morning briefing on Telegram with weather, calendar, and health data from Garmin. It had a whole cron system that kicked off jobs throughout the day.

I'd been using [pi](https://github.com/mariozechner/pi-coding-agent) at work for months as a coding agent, and I'd already built extensions for the hard parts — [persistent memory](https://github.com/samfoy/pi-memory), [session search](https://github.com/samfoy/pi-session-search), [knowledge search](https://github.com/samfoy/pi-knowledge-search). At some point I realized I could build a better personal assistant from pi's pieces than what OpenClaw gave me out of the box.

Pi has no built-in iMessage integration, no calendar access, no task manager, no notification system. But it has an extension API and a skill system that let you build all of that yourself. It took about two days to get everything working.

## Skills: teaching the agent new domains

Pi has a concept called skills — markdown files in `~/.pi/agent/skills/` that get loaded into context when relevant. Each skill is a reference card: what CLI tools are available, what API endpoints exist, what the expected formats look like. The agent reads the skill and then uses the tools described in it.

I wrote 12 of these:

| Skill | What it does |
|-------|-------------|
| `bluebubbles` | Send/receive iMessages via BlueBubbles HTTP API |
| `gmail` | Read, search, send email via the `gog` CLI |
| `macos-calendar` | Read Google Calendar events via `gog`, Apple Calendar via `icalBuddy` |
| `things` | Add and query tasks in Things 3 via the `things` CLI |
| `ynab` | Check budget, spending, account balances via YNAB REST API |
| `peekaboo` | macOS UI automation — screenshots, clicks, typing, app control |
| `reddit` | Browse, search, read, comment via `rdt-cli` |
| `obsidian-vault` | Interact with my Obsidian vault — tasks, daily notes, meeting notes |
| `humanizer` | Strip AI writing patterns from text before publishing |
| `gh-issues` | File and manage GitHub Issues via `gh` CLI |
| `marp-slides` | Build slide decks from markdown |
| `md2docx` | Convert markdown to Word docs with rendered mermaid diagrams |

Each skill file is maybe 50-100 lines of markdown. It's just documentation that tells the agent how to use existing tools on my machine. The BlueBubbles skill, for example, documents the HTTP endpoints for querying chats and sending messages. The agent reads it, then makes `curl` calls.

Any CLI tool or HTTP API on your machine can become a skill this way. Most of mine took about 10 minutes to write.

## Porting the automations

OpenClaw had a built-in cron system. Pi doesn't. So I ported the three most valuable automations to standalone Python scripts running on macOS `launchd`:

**iMessage → Things** (`scan_messages.py`, every 15 minutes): Polls the BlueBubbles API for new inbound messages, sends them to Claude via Bedrock to extract actionable items, creates tasks in Things 3 with fuzzy dedup to avoid duplicates, and logs everything to the daily note.

**Email checker** (`check_email.py`, hourly): Calls `gog` to fetch unread Gmail, summarizes anything important, appends to the daily note's email section.

**Daily note creation** (`create_daily_note.sh`, 5am): Creates the Obsidian daily note with calendar events pulled from `icalBuddy`, pre-formatted sections for the day.

These run via LaunchAgents in `~/Library/LaunchAgents/`. The scripts themselves are maybe 150 lines each. They call Bedrock directly for LLM inference instead of going through an assistant framework. State is tracked in flat files under `~/pi-automations/state/`.

These are just Python scripts and a shell script. When the iMessage scanner misclassifies something, I fix the prompt or the parsing logic directly.

## The Telegram bot

OpenClaw had a Telegram interface built in. Pi doesn't, but I have a [Telegram bot](https://github.com/samfoy/pi-telegram-bot) that spawns pi sessions per conversation thread. It handles slash commands, streaming responses, file uploads, and Markdown rendering. I can text it from my phone and get a pi session without opening my laptop.

Getting it stable took more work than I expected. Telegram's `editMessageText` API has rate limits that the streaming updater would hit during fast model responses. The Markdown dialect Telegram uses isn't CommonMark, so code blocks with certain characters would break message edits silently. Dead pi sessions would sit in the session map and swallow messages. Each of these took a debugging session to find and fix.

But it works, and I now message my agent from my phone more than from my terminal.

## Memory carries over

The thing that makes this feel like an assistant rather than a stateless tool is [persistent memory](https://github.com/samfoy/pi-memory). I'd already built this for work — it stores facts (key-value pairs with confidence scores) and lessons (corrections that stick across sessions). I pulled OpenClaw's stored knowledge out of its SQLite database and `MEMORY.md` files and imported everything into pi-memory. About 370 facts and 200 lessons.

Now when I tell the agent to send a message, it knows who I'm talking about and which skill to use. When I ask about my budget, it remembers my YNAB credentials. It knows my location, my preferences, my project context. It adds up. The agent remembers enough context that I don't have to re-explain myself every session.

I wrote about how the memory system works in a [previous post](https://samfoy.github.io/circuit-break/posts/your-agent-forgets-everything-between-sessions/).

## What's fragile

If the BlueBubbles server goes down, the iMessage scanner fails silently (I added logging after this happened twice). The LaunchAgent plists are finicky about environment variables. The Telegram bot needs to be running as a background process on my Mac, which means it goes down when my laptop sleeps.

Everything runs locally though, and I chose every dependency. When the email checker started duplicating entries, I found the bug in `check_email.py` in two minutes.

## What I'd tell someone starting from scratch

If you want a pi-based personal assistant:

1. Start with the skills that match tools already on your machine. If you use Things, write a Things skill. If you use Todoist, write a Todoist skill. The pattern is the same: document the CLI or API, let the agent figure out the calls.

2. Install [pi-memory](https://github.com/samfoy/pi-memory). Without memory, the agent is a stranger every time you open it.

3. Don't try to build a full automation pipeline on day one. Start with the interactive case — ask the agent to do things manually. Once you know what you ask for repeatedly, automate those specific things.

4. Keep the automations simple and independent. A Python script that does one thing, runs on a timer, and logs to a file is much easier to debug than a framework with hooks and middleware.

5. The [AGENTS.md](https://samfoy.github.io/circuit-break/posts/stop-putting-behavioral-rules-in-your-agents-prompt/) file in `~/.pi/agent/` is where you define who the agent is. Mine has communication preferences, when to use which memory tool, daily note conventions, and a list of available skills with their file paths. This is the closest thing to a personality config.

The whole setup took about two days. Pi did most of the implementation — I described what I wanted, it wrote the skills and scripts, I tested them and said what was broken, it fixed things. Using a coding agent to build your own assistant is a little recursive, but it works.

[pi](https://github.com/mariozechner/pi-coding-agent) · [pi-memory](https://github.com/samfoy/pi-memory) · [pi-telegram-bot](https://github.com/samfoy/pi-telegram-bot)
