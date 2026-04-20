---
title: "Obsidian as Your Agent's Second Brain"
date: 2026-04-20 10:00:00 -0700
categories: [AI Agents, Tools]
tags: [pi, obsidian, knowledge-management, memory, assistant]
---

My agent writes to an Obsidian vault hundreds of times a day. Daily journal entries, email summaries, iMessage extractions, movie notes, recipe archives, trip plans, people profiles. Not because I told it to use Obsidian specifically — but because a local folder of markdown files is the ideal substrate for an AI agent that needs persistent, searchable, structured knowledge.

Here's how the setup works, and why I think every agent-based workflow benefits from something like this.

## The vault

My Obsidian vault lives at `~/Documents/Sam/`. It's just a directory:

```
Sam/
├── Daily/          # One file per day (2026-04-20.md)
├── Notes/          # Long-lived reference notes
│   └── Movies/     # One file per movie watched
├── People/         # Contact profiles
├── Recipes/        # Structured recipe files
├── Research/       # Active investigations
├── Projects/       # Project notes
├── TaskNotes/      # Task-linked notes
├── Templates/      # Templater templates
└── Maps/           # Obsidian canvas files
```

Every file is markdown. Every file is on my local filesystem. There's no database, no proprietary format, no sync service I don't control. That matters because my agent interacts with this vault through `read`, `write`, and `edit` tool calls — the same tools it uses to write code. No special integration required.

## Daily notes as the operating log

The most active part of the vault is `Daily/`. Every morning at 5am, a launchd job creates a new daily note from a template. It pulls calendar events via `icalBuddy` and pre-populates the day's structure:

```markdown
# 2026-04-20 — Monday

> [!note]- 📅 Today
> <!-- CALENDAR_START -->
> No events today.
> <!-- CALENDAR_END -->

## 📝 Journal

## 📧 Email
```

Throughout the day, three things write to this file:

1. **The agent itself** — when I'm in a session and something notable happens, it appends a timestamped journal entry. Task completions, decisions, life events.
2. **The email automation** — an hourly Python script checks Gmail and appends summaries under the Email section.
3. **The iMessage scanner** — every 15 minutes, it checks for new inbound messages and logs any extracted tasks.

By end of day, the file is a complete record of what happened. Here's a real excerpt:

```markdown
## 📝 Journal
- 01:00 — Published blog post to circuit-break: "I Built a Native iOS 
  App in Three Days with an AI Agent"
- 01:36 — Set up release-please across 16 pi repos
- 14:30 — Chores day — watered plants, mowed the grass
- 15:48 — ✅ Task → Things: Write Donna's card from Sam and Eryn
- 17:03 — 💬 New address received from family member

## 📧 Email
- 10:32 — 📧 Xfinity: Your bill statement is ready
```

I don't write any of this manually. The agent and the automation scripts write it. I read it in Obsidian when I want to remember what happened on a given day.

## Structured notes the agent creates and queries

When I mention watching a movie, the agent creates a file in `Notes/Movies/`:

```markdown
---
title: Sorry Baby
year: 2025
director: Eva Victor
rating: 8
date_watched: 2026-04-19
tags: [movie]
---

# Sorry Baby (2025)

Eva Victor's feature debut as writer, director, and star...
```

There's a `.base` view file that renders all movies as a sortable table in Obsidian — grouped by watch date, with star ratings. I didn't build that table view in code. Obsidian's native database views handle it from the frontmatter.

Recipes work the same way. When I cook something worth keeping, the agent creates a structured recipe note:

```markdown
---
tags: [recipe]
cuisine: American
category: dinner
protein: chicken
difficulty: medium
time_prep: 30 min (+ 1–4 hr marinate)
time_cook: 15 min
servings: 2–3
equipment: [wok, wire rack, thermometer, tongs]
ingredients_tags: [chicken, harissa, buttermilk, cornmeal]
source: original
made_dates: [2026-03-22]
---
```

Full instructions, notes on what worked, variations, and a "made log" with dates and observations each time I cook it. Next time I ask "what should I make with chicken?" the agent can search through these and suggest something I've actually cooked before.

People notes in `People/` track contacts — email, phone, relationship, key facts. When I tell the agent to text someone, it already knows the number. When I'm planning a trip to visit family, it already knows who lives where.

## The knowledge search layer

Raw files in a folder get you persistence, but not retrieval. The agent can `read` any file, but it doesn't know which file to read unless you tell it. That's where [pi-knowledge-search](https://github.com/samfoy/pi-knowledge-search) comes in.

It indexes the vault using vector embeddings (Titan via Bedrock in my case), watches for file changes in real-time, and exposes a `knowledge_search` tool. The agent calls it with natural language queries:

```
knowledge_search("what did I write about the Easter trip")
→ Notes/Easter Visit 2026.md (0.89 similarity)

knowledge_search("harissa chicken recipe")  
→ Recipes/Harissa Fried Chicken.md (0.92 similarity)

knowledge_search("when did I last talk to my sister")
→ People/Kathryn.md (0.85 similarity)
```

This is semantic search over my personal files. It costs almost nothing to run — the embeddings are generated once per file and updated on change. The index is a local SQLite database. No cloud service, no subscription.

The config is minimal:

```json
{
  "dirs": ["~/Documents/Sam"],
  "fileExtensions": [".md", ".txt"],
  "excludeDirs": ["node_modules", ".git", ".obsidian", ".trash"],
  "provider": {
    "type": "bedrock",
    "model": "amazon.titan-embed-text-v2:0"
  }
}
```

## Why Obsidian specifically

You could do all of this with a folder of markdown files and never open Obsidian. The agent doesn't use Obsidian — it uses the filesystem. But Obsidian gives me three things the agent can't:

**Visual browsing.** I can scroll through daily notes, click into linked people profiles, browse the recipe collection. The agent writes the data; Obsidian is how I read it as a human.

**Database views.** The `.base` files let me create filtered, sorted, grouped tables from frontmatter properties — movies by rating, recipes by cuisine, people by last contact date. The agent writes structured frontmatter; Obsidian renders it.

**Graph and linking.** `[[Bailey Pendergast]]` in a daily note links to her people profile. Trip notes link to the people who were there. This wikilink structure costs nothing to create (the agent just writes `[[name]]`) but builds a navigable web of context over time.

None of this requires Obsidian Sync, Obsidian Publish, or any paid feature. The free app reading a local folder is all you need.

## The feedback loop

The part that makes this more than a note-taking app: the agent reads back what it writes.

When I start a new session, the agent can search my vault to recover context from days, weeks, or months ago. "What did we plan for that trip?" pulls up the trip note. "When did I last make that salmon recipe?" finds the recipe with `made_dates` in the frontmatter. "What's my dentist's number?" finds the relevant contact note.

This is different from [persistent memory](https://github.com/samfoy/pi-memory), which stores atomic facts and corrections. The vault stores *documents* — full recipes, trip itineraries, daily logs, research notes. Memory is for "Bailey's address is X." The vault is for "here's the entire Easter trip plan with meal schedules and activities."

Both feed into the agent. Memory is checked automatically on relevant queries. Knowledge search is called when the agent needs to find something in my files. They complement each other — memory is fast and precise, knowledge search is broad and fuzzy.

## What I'd change

The daily note template has sections that were designed for a different era (the OpenClaw migration). The "Claw Log" callout is vestigial. The health section is empty because I haven't wired up Garmin data yet. I'd simplify the template.

File naming is inconsistent. Some notes use spaces, some use hyphens. The agent doesn't care, but Obsidian's link resolution can get confused when you have both `Bailey Pendergast.md` and a potential `bailey-pendergast.md`.

I don't have a good archival strategy for daily notes. The folder currently has months of files. Obsidian handles it fine, but searching through all of them will get slower as the vector index grows.

## If you're starting from scratch

1. Make a folder. Put markdown files in it. That's the vault.
2. Set up a daily note template with sections your automations can write to. Use HTML comments as markers (`<!-- CALENDAR_START -->`) so scripts can find their insertion points.
3. Pick a structure for recurring content — recipes, people, projects, whatever you track — and create a template with frontmatter properties. Consistency in frontmatter means you can query it later.
4. Install [pi-knowledge-search](https://github.com/samfoy/pi-knowledge-search) and point it at your vault. Now your agent can find things by meaning, not just filename.
5. Open the folder in Obsidian. Use it to browse and visualize. Let the agent use it to read and write.

The vault isn't a product. It's a directory of text files that happens to be legible to both humans and AI. That's the whole point.

[Obsidian](https://obsidian.md) · [pi-knowledge-search](https://github.com/samfoy/pi-knowledge-search) · [pi-memory](https://github.com/samfoy/pi-memory)
