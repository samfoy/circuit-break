---
title: "I Built a Native iOS App in Three Days with an AI Agent"
date: 2026-04-19 10:00:00 -0700
categories: [AI Agents, Projects]
tags: [ios, swift, pi, mobile, pi-dashboard]
---

I don't know Swift. I've never opened Xcode with the intent to ship something. Last week I had a web app, [Pi Dashboard](https://github.com/samfoy/pi-dashboard), a browser-based UI for the [pi coding agent](https://github.com/mariozechner/pi-coding-agent). By Friday I had a native iOS app with 10,700 lines of Swift across 50+ files, a Share Extension, a WidgetKit widget, Siri Shortcuts, and integrations with HealthKit, Calendar, Contacts, Reminders, Location, and Speech Recognition.

84 commits in three days. I wrote maybe a dozen lines of Swift by hand. The rest was the agent. Some of it went well. Some of it went memorably wrong.

## Starting point

Pi Dashboard already had a web frontend (React/TypeScript) and an Electron desktop wrapper. I wanted to use it from my phone. The first attempt was making the web app mobile-responsive and serving it over Tailscale. That took a couple hours via [autoloop](https://github.com/MikeyOBrien/autoloop), an autonomous coding loop that plans, builds, critiques, and iterates without human input.

It worked. It was also the typical mobile web experience: janky keyboard handling, no push notifications, no share sheet, Safari auto-zooming the text input because the font was 14px instead of 16px.

So I asked: "Can we build a native iOS app with the polish of the ChatGPT or Claude apps, but talking to pi over Tailscale?"

Three research subagents spun up in parallel. One mapped the backend API surface, one studied ChatGPT and Claude's iOS UX patterns, one researched SwiftUI and the platform APIs we'd need. They produced 124KB of research docs before a single line of Swift was written. Then autoloop took those specs and scaffolded 29 Swift files in one pass. It compiled on the first try.

That was the end of day one.

## How the work actually went

The pattern was consistent: I'd describe a feature in plain English, the agent would scaffold it, I'd test on my phone, describe what was wrong, and the agent would fix it. Repeat.

A typical exchange:

> Me: "Add a share extension so I can send URLs and images to pi from any app. Include quick action chips: chat, summarize, extract, vault, research."
>
> Agent: creates the extension target, wires it into the Xcode project, adds the App Group for shared UserDefaults, builds the ShareView with action chips, handles URL and image payloads, commits.

That feature took about 30 minutes. The agent wrote the `ShareViewController`, the `ShareView` and `ShareViewModel`, modified the `.pbxproj` to add the new target, created the `.entitlements` file, and added the `xcscheme`. I couldn't have done that manually in a day. I don't know how Xcode project files work, and I don't want to.

### The slice pattern

Early on I found that one big feature request produced messy results. The code would compile but the architecture was tangled. Breaking features into explicit slices fixed this:

"Slice 1: artifact cards and FileViewerSheet stub. Just the card UI and a sheet that opens. No file content yet."

"Slice 2: full-screen file viewer with markdown rendering, syntax-highlighted code, and image support."

"Slice 3: share sheet and copy toolbar for the file viewer."

"Slice 4: diff viewing with a versions tab and LineDiffView."

Each slice was a commit. Each built on the last. The agent could focus on one concern at a time, and I could catch problems before they compounded. I used this pattern for every major feature after discovering it.

### The parallel sprint

On day two, I kicked off four autoloops simultaneously:

1. Share Extension quick actions
2. Siri Shortcuts via AppIntents
3. WidgetKit home screen widget
4. Full-text conversation search

All four completed in about 90 minutes, build passed clean. Then two more autoloops for inline image rendering and iPad split view. Then a per-session working directory picker. Then the agent wrote 566 tests (49 backend, 430 frontend, 87 iOS) in about 30 minutes across three parallel subagents.

A solo developer doing this traditionally would be looking at weeks, probably months.

## Where it broke

### AI creates, AI destroys

The most telling bug: an autoloop adding a "per-session working directory picker" feature rewrote 536 lines of the Xcode project file and silently deleted the share extension target. The share extension files were still on disk. The target was just gone from `project.pbxproj`. The autoloop had no awareness that other features depended on those entries.

I didn't notice until the next day when I tried to share a URL to PiDash and it wasn't in the share menu. The fix required a Ruby xcodeproj gem to reconstruct the target properly.

Autonomous coding loops running in parallel each optimize locally without global awareness. The agent that adds feature X doesn't know about feature Y, and Xcode's project file doesn't tolerate partial rewrites.

### SwiftUI concurrency fights

The agent kept getting tripped up by Swift's structured concurrency. `CancellationError` would bubble up as user-visible error banners because SwiftUI cancels `Task` blocks when views disappear. The agent would fix one instance and create the same pattern somewhere else. It took three rounds: suppress in async paths, use unstructured `Task` for initial loads, prevent wrapping in `APIError`. The pattern was clear in hindsight but the agent didn't generalize from the first fix.

### The data contract you didn't think about

The iOS app sent images as `{data: "<base64>", mimeType: "image/jpeg"}`. The backend expected `{type: "image", data, mimeType}`. One missing field. Pi hit "Unknown user content type" and the session file grew to 6.8MB of malformed data. Each attempt to repair it introduced new breakage, string content where arrays were expected, broken tool_use/tool_result pairing. We eventually had to delete the slot and start fresh.

The fix was a `normalizeImages()` function on the server to accept any format from any client. Obvious in hindsight. But when the agent builds the iOS client and the web client in separate sessions, neither one thinks about the other's payload format. The server needs to be the defensive layer, and it wasn't.

## What your job becomes

When the agent writes the code, your role shifts. You're deciding what to build, in what order, and how to evaluate the output. The engineering decisions still matter (you need enough taste to spot bad architecture) but the day-to-day is closer to product management.

I spent most of my time testing on my phone, noticing things that felt off, and describing what I wanted instead. "The connection banner shouldn't flash on normal reconnects." "I got stuck in the file picker and couldn't scroll out." "The empty state looks bad." My phone was the test device; screenshots and specific descriptions were the bug reports. When I pasted actual error messages, the agent fixed things in one shot. When I described problems vaguely ("it's not working right"), it guessed wrong.

### You don't need to know the language

I picked Swift/SwiftUI because it's the right tool for iOS. I didn't learn Swift first. I described what I wanted and read the code the agent wrote to build a mental model. By day two I could spot wrong patterns, like re-creating a WebSocket connection on every view appear, even though I couldn't write the fix myself.

The barrier to building native apps used to be "learn the language and the platform." Now it's "know enough to review the output and describe what's wrong." I don't think I'd pass a Swift interview, but I shipped an app.

### The testing gap

The app has 566 tests. The agent wrote them all. The real QA was me tapping around on my phone. AI-generated tests tend to test what the agent thinks it built, which is circular. The tests pass, but they're testing the happy path of code the agent wrote.

For a personal project, fine. For shipping software, you'd want to write the test cases yourself (at least the edge cases) and have the agent implement them. The agent is good at writing test code. It's bad at knowing what to test.

## The numbers

| Metric | Value |
|--------|-------|
| Calendar days | 3 |
| Commits | 84 |
| Lines of Swift | ~10,700 |
| Swift files | 50+ |
| Tests written | 566 |
| Lines I wrote by hand | ~12 |
| Prior Swift experience | Zero |

Features: multi-session chat, Share Extension with quick actions, WidgetKit widget, Siri Shortcuts (Ask Pi, Send to Pi, status, chat list), HealthKit, Calendar, Contacts, Reminders, Location, Speech Recognition, full-text search, iPad split view, inline images, diff viewer, command palette, background refresh, push notifications, per-session working directory.

## What I'd do differently

I wouldn't run parallel autoloops against the same Xcode project file again. The `.pbxproj` is fragile and each loop rewrites it without knowing what the others added. Sequential loops with distinct scopes are safer.

The image format mismatch that corrupted a session was preventable. If I'd thought about the data contract between clients up front, or at least added server-side normalization from day one, that whole debugging session wouldn't have happened.

And I'd use the slice pattern from the start instead of discovering it after a few messy big-prompt commits. Small, sequential, one-concern slices with one commit each.

## The honest take

The agent is a force multiplier on taste and judgment. I knew what a good iOS chat app felt like because I use ChatGPT and Claude daily. I could test on real hardware and describe what was wrong in specific terms. The agent turned that into working code faster than should be possible for someone who doesn't know Swift.

If I didn't have that judgment, if I couldn't tell good UX from bad or spot an architectural problem in code I can't write, the output would have been bad fast instead of good fast. The velocity is conditional on having something worth multiplying.

The code is open source: [github.com/samfoy/pi-dashboard](https://github.com/samfoy/pi-dashboard). The iOS app lives in `ios/PiDash/`.
