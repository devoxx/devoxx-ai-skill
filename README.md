# Devoxx / VoxxedDays Plugin

A Claude plugin marketplace for Devoxx and VoxxedDays conferences.

## Installation

In Claude Code:

```
/plugin marketplace add devoxx/devoxx-ai-skill
/plugin install devoxx-voxxed-cfp@devoxx-plugins
```

In the Claude desktop app (Cowork), add the marketplace under Settings → Capabilities.

## Using with Codex CLI / Gemini CLI

The skill follows the cross-agent `SKILL.md` standard, so it also works in OpenAI
Codex CLI and Google Gemini CLI. Only the plugin/marketplace wrappers are
Claude-specific — copy the skill folder itself:

```bash
git clone https://github.com/devoxx/devoxx-ai-skill

# Codex CLI (personal skills; use .codex/skills/ in a project for per-repo)
cp -r claude-plugin/plugins/devoxx-voxxed-cfp/skills/devoxx-voxxed-cfp ~/.codex/skills/

# Gemini CLI (user skills; use .gemini/skills/ in a workspace for per-repo)
cp -r claude-plugin/plugins/devoxx-voxxed-cfp/skills/devoxx-voxxed-cfp ~/.gemini/skills/
```

Then just ask a Devoxx/Voxxed question and the skill auto-activates. In Codex CLI
you can also invoke it explicitly with `$devoxx-voxxed-cfp` (list skills with
`/skills`). The skill only needs `curl` and `python3` shell access — no API keys.

## Plugins

### devoxx-voxxed-cfp

Ask plain-language questions about any Devoxx or VoxxedDays conference — speakers,
talks, schedules, rooms, tracks — answered live from the events' public CFP APIs
(`{event-slug}.cfp.dev`), with event discovery via devoxxians.com. Every talk and
speaker in an answer links to its page on m.devoxx.com.

Examples:

- "When is the next Devoxx?"
- "When are the CFPs opening for these events?"
- "Who is speaking about Kubernetes at Devoxx UK?"
- "What's on Wednesday in Room 8 at Devoxx Belgium?"
- "Most popular AI talks at Voxxed Zurich"
- "What are the main topics on Wednesday at Devoxx Belgium 2025?"

### Example Prompt

"When are the CFPs opening for these events?"

<img width="1437" height="388" alt="Screenshot 2026-06-07 at 12 55 14" src="https://github.com/user-attachments/assets/1a55eea2-d5d5-44d8-a6d5-52577e0133bf" />

## Releasing updates

Bump `version` in `plugins/devoxx-voxxed-cfp/.claude-plugin/plugin.json` on every
release — users only receive updates when that field changes.

## License

[MIT](LICENSE)
