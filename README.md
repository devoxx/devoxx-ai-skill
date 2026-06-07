# Devoxx Plugins

A Claude plugin marketplace for Devoxx and VoxxedDays conferences.

## Installation

In Claude Code:

```
/plugin marketplace add devoxx/claude-plugins
/plugin install devoxx-voxxed-cfp@devoxx-plugins
```

In the Claude desktop app (Cowork), add the marketplace under Settings → Capabilities.

## Plugins

### devoxx-voxxed-cfp

Ask plain-language questions about any Devoxx or VoxxedDays conference — speakers,
talks, schedules, rooms, tracks — answered live from the events' public CFP APIs
(`{event-slug}.cfp.dev`), with event discovery via devoxxians.com. Every talk and
speaker in an answer links to its page on m.devoxx.com.

Examples:

- "When is the next Devoxx?"
- "Who is speaking about Kubernetes at Devoxx UK?"
- "What's on Wednesday in Room 8 at Devoxx Belgium?"
- "Most popular AI talks at Voxxed Zurich"

## Releasing updates

Bump `version` in `plugins/devoxx-voxxed-cfp/.claude-plugin/plugin.json` on every
release — users only receive updates when that field changes.
