---
name: devoxx-voxxed-cfp
description: >
  Answer questions about speakers, talks, and schedules for any Devoxx or
  VoxxedDays conference by querying the event's public CFP API at
  [event-slug].cfp.dev. Discovers event slugs via the devoxxians.com events
  API. Use whenever the user asks about a Devoxx or Voxxed speaker bio,
  company or socials, a talk or session topic, track, level or description,
  who is presenting, the program or agenda for a day, what is on in a room,
  which talks cover a topic, which Devoxx and VoxxedDays events are
  upcoming or past, or when an event's CFP (call for papers) opens or
  closes. Example triggers: who is speaking about Kubernetes at Devoxx UK,
  what is on Wednesday in Room 8, find AI talks at Voxxed Zurich, tell me
  about a speaker, most popular talks, when is the next Devoxx, when does
  the CFP open for Devoxx Belgium.
---

# Devoxx / VoxxedDays — CFP API

Answer the user's question by querying public, no-auth REST APIs. Each Devoxx and VoxxedDays event has its own CFP instance at `https://{event-slug}.cfp.dev/api/public`. Fetch live data with `curl` in the bash tool and parse JSON with `python3`. Never invent talk titles, times, rooms, or speaker facts — always pull from the API.

## Step 0 — resolve the event slug

Discover events from the Devoxxians registry (no auth):

- `GET https://devoxxians.com/api/public/events/upcoming`
- `GET https://devoxxians.com/api/public/events/past`

Each event has `name` (e.g. "Devoxx UK 2026"), `eventCategory` (`DEVOXX` or `VOXXED`), `fromDate`/`toDate`, `location`, `website`, and `apiURL` (e.g. `https://devoxxuk26.cfp.dev/api/`). The slug is the hostname prefix of `apiURL` — extract with `re.match(r"https://([^.]+)\.cfp\.dev", apiURL)`.

Resolution rules:

- Match the user's event mention against `name` case-insensitively (fetch both lists; "Devoxx UK" without a year → prefer the nearest upcoming edition, else the most recent past one).
- No event mentioned at all → use conversation context; if still ambiguous, ask which event (and offer the upcoming list).
- `apiURL` can be null for announced events whose CFP isn't live yet (e.g. a next-year edition) — say so and fall back to the most recent edition that has an `apiURL`.
- Cache the slug for the rest of the conversation; don't re-resolve on every question.

Questions like "when is the next Devoxx" or "which Voxxed events are coming up" are answered directly from these two registry endpoints.

## Per-event endpoints

Base: `https://{slug}.cfp.dev/api/public` — all GET, no auth.

| Need | Endpoint |
|------|----------|
| Event metadata + CFP open/close dates | `/event` |
| All talks (content) | `/talks` |
| All speakers (summary, paginated) | `/speakers?page=N` |
| One speaker + their talks | `/speakers/{id}` |
| Available schedule days | `/schedules` (returns day links — days VARY per event, never assume monday–friday) |
| Schedule for a day | `/schedules/{day}` |
| Rooms / Tracks | `/rooms`, `/tracks` |

Full field reference, gotchas, and worked `curl`+`python3` recipes are in `references/api.md` — read it before composing a non-trivial query.

## Key joins

- A schedule slot's `proposal.id` equals a talk's `id` in `/talks`.
- A talk's `speakers[]` and a speaker's `proposals[]` link talks ↔ speakers.
- `/talks` carries content (description, track, level, favourites) but its `timeSlots` is usually empty — get times and rooms from `/schedules/{day}`.

## Times

Display times in the event's local timezone. Every schedule slot carries `fromDate`/`toDate` in UTC plus a `timezone` field (e.g. `Europe/London`, `Europe/Brussels`). Convert with `zoneinfo`:

```python
from datetime import datetime
from zoneinfo import ZoneInfo
def local(ts, tz): return datetime.fromisoformat(ts.replace("Z","+00:00")).astimezone(ZoneInfo(tz)).strftime("%H:%M")
```

## Linking to the public website (mandatory)

Every talk and every speaker referenced in an answer MUST be a clickable markdown link to `https://m.devoxx.com`, using the same event slug:

- Talk: `https://m.devoxx.com/events/{event-slug}/talks/{talkId}/{slug(title)}` — e.g. [Top 10 Event-Driven Architecture Pitfalls](https://m.devoxx.com/events/dvbe25/talks/7729/top-10-event-driven-architecture-pitfalls)
- Speaker: `https://m.devoxx.com/events/{event-slug}/speaker/{speakerId}/{slug(fullName)}` (note: `speaker`, singular) — e.g. [Victor Rentea](https://m.devoxx.com/events/dvbe25/speaker/8101/victor-rentea)

Routing is by id (the trailing slug is cosmetic), but generate it properly:

```python
import re
def slug(s): return re.sub(r"[^a-z0-9]+", "-", (s or "").lower()).strip("-")
```

Format as `[Talk Title](url)` and `[Speaker Name](url)` everywhere — including lists and schedule overviews. For event-level answers, link the event's `website` from the registry payload.

## Answering patterns

- "Tell me about <speaker>" → page through `/speakers` to find the id, then `GET /speakers/{id}` for bio + proposals.
- "Who's speaking about <topic>" / "find <topic> talks" → fetch `/talks`, case-insensitively match across `title`, `summary`, `description`, `keywords[].name`, `track.name`.
- "What's on <day>" / "what's in <room>" → `GET /schedules` for valid days, then `/schedules/{day}`, filter by room, sort by `fromDate`.
- "Most popular talks" → sort `/talks` by `totalFavourites` descending.
- "When/where is the next Devoxx or Voxxed event" → registry endpoints on devoxxians.com.
- "When does/did the CFP open or close?" / "is the CFP open for <event>?" → resolve the slug, then `GET /event` and read `cfpOpening`/`cfpClosing` (UTC ISO). Compare to now to label open / closed / opens-later; an event whose `apiURL` is null has no CFP instance yet. The registry does NOT carry CFP dates — always use `/event`.

Keep answers focused on what was asked. Every talk title and speaker name in the answer must be a markdown link per the linking rules above.
