# Devoxx / VoxxedDays CFP API — Reference

Two API families, both public (no auth):

1. **Event registry** — `https://devoxxians.com/api/public/events/{upcoming|past}`
2. **Per-event CFP** — `https://{event-slug}.cfp.dev/api/public/...`

The Swagger/OpenAPI docs (`/api/v3/api-docs`, `/api/swagger-ui`) and non-`public`
endpoints on cfp.dev instances require auth and return 401 — ignore them.

Always fetch with `curl -s -m 30` and parse with `python3`. Responses can be
large (a big event's talks payload is ~840 KB), so filter and project fields
rather than printing whole bodies.

Examples below use `dvbe25` (Devoxx Belgium 2025); substitute the resolved slug.

---

## Event registry (devoxxians.com)

`GET https://devoxxians.com/api/public/events/upcoming`
`GET https://devoxxians.com/api/public/events/past` (~20 most recent)

Each event:

| Field | Notes |
|-------|-------|
| `name` | e.g. "Devoxx UK 2026", "Voxxed Days Zurich 2026" |
| `eventCategory` | `DEVOXX` or `VOXXED` |
| `apiURL` | e.g. `https://devoxxuk26.cfp.dev/api/` — **may be null** if the CFP isn't live yet; slug = hostname prefix |
| `fromDate` / `toDate` | UTC ISO timestamps |
| `location` | `{name, city, country{name}, timezone}` (timezone here can be malformed, e.g. `krakow/europe` — prefer the `timezone` on schedule slots) |
| `website` | event homepage |
| `imageURL` | event image |

List events with slugs:

```bash
python3 - <<'PY'
import json, subprocess, re
def get(u): return json.loads(subprocess.run(["curl","-s","-m","20",u],capture_output=True,text=True).stdout)
for kind in ("upcoming","past"):
    for e in get(f"https://devoxxians.com/api/public/events/{kind}"):
        m = re.match(r"https://([^.]+)\.cfp\.dev", e.get("apiURL") or "")
        print(kind, e["eventCategory"], m.group(1) if m else "(no CFP yet)", "-", e["name"], e["fromDate"][:10])
PY
```

---

## GET /talks

`https://{slug}.cfp.dev/api/public/talks` — JSON array of all sessions with
their content. Source of truth for descriptions, tracks, levels — but NOT for
times/rooms (`timeSlots` is normally empty; use `/schedules/{day}`).

Each talk object:

| Field | Notes |
|-------|-------|
| `id` | integer; matches `proposal.id` in `/schedules/{day}`; used in m.devoxx.com talk URLs |
| `title` | string |
| `description` | string, may contain HTML, may be null |
| `summary` | plain-text abstract, may be null |
| `audienceLevel` | `BEGINNER` / `INTERMEDIATE` / `ADVANCED` |
| `totalFavourites` | integer — popularity signal |
| `track` | `{id, name, description, imageURL}` |
| `sessionType` | `{id, slug, name, duration, ...}` (Conference, Deep Dive, Keynote, BOF, Quickie, ...) |
| `speakers` | array of speaker objects (`id`, `fullName`, ...) |
| `keywords` | array of `{"name": "..."}` objects (NOT plain strings) |
| `timeSlots` | usually empty — do not rely on it |

Example — find talks matching a topic, with mandatory m.devoxx.com links.
Gotchas: `summary`/`description` can be `null` (coerce with `or ""`),
`keywords` items are dicts, and never put backslash-escaped quotes inside an
f-string (use `.format()` or pre-assign):

```bash
curl -s -m 30 "https://dvbe25.cfp.dev/api/public/talks" | python3 -c '
import sys, json, re
EVENT = "dvbe25"
term = "kubernetes"  # case-insensitive
def slug(s): return re.sub(r"[^a-z0-9]+", "-", (s or "").lower()).strip("-")
d = json.load(sys.stdin)
def kw(t): return " ".join((k.get("name") or "") for k in (t.get("keywords") or []))
def hit(t):
    parts = [t.get("title"), t.get("summary"), t.get("description"),
             (t.get("track") or {}).get("name"), kw(t)]
    return term in " ".join(p or "" for p in parts).lower()
m = sorted([x for x in d if hit(x)], key=lambda x: -(x.get("totalFavourites") or 0))
print("matches:", len(m))
for t in m:
    turl = "https://m.devoxx.com/events/{}/talks/{}/{}".format(EVENT, t["id"], slug(t["title"]))
    spk = ", ".join("[{}](https://m.devoxx.com/events/{}/speaker/{}/{})".format(
        s.get("fullName",""), EVENT, s["id"], slug(s.get("fullName"))) for s in (t.get("speakers") or []))
    print("- [{}]({}) ({}, {}) - {} [{} fav]".format(
        t["title"], turl, (t.get("track") or {}).get("name"),
        t.get("audienceLevel"), spk, t.get("totalFavourites")))
'
```

Most popular talks: sort by `totalFavourites` descending, same link format.

---

## GET /speakers?page=N

20 speakers per page, `page` is 0-based. **Important:** past the last page the
API returns an *empty body* (zero bytes), not `[]` — the loop must stop on an
empty/blank response or `json.loads` will throw. Summary fields:

`id, firstName, lastName, fullName, bio (HTML), anonymizedBio, company,
imageUrl, twitterHandle, linkedInUsername, blueskyUsername, mastodonUsername,
countryName`

Find a speaker id by name (curl-based paging; handles the empty-body
terminator):

```bash
python3 - <<'PY'
import json, subprocess
SLUG = "dvbe25"
name = "abdel"  # case-insensitive substring
out, page = [], 0
while page < 60:
    raw = subprocess.run(
        ["curl","-s","-m","20",
         f"https://{SLUG}.cfp.dev/api/public/speakers?page={page}"],
        capture_output=True, text=True).stdout.strip()
    if not raw:           # empty body => past the last page
        break
    data = json.loads(raw)
    if not data:          # explicit [] => also stop
        break
    out += data
    page += 1
for s in out:
    if name in (s.get("fullName") or "").lower():
        print(s["id"], s["fullName"], "-", s.get("company"))
PY
```

## GET /speakers/{id}

One speaker with full detail, including `proposals` — the talks they're giving
(same shape as `/talks` items). Fields: `id, firstName, lastName, bio, company,
imageUrl, twitterHandle, linkedInUsername, blueskyUsername, mastodonUsername,
userName, proposals[]`.

---

## GET /schedules  and  GET /schedules/{day}

`/schedules` returns links to the event's days. **Days vary per event** (e.g.
Devoxx Belgium 2025: monday–friday; Devoxx UK 2026: wednesday–thursday) —
always fetch `/schedules` first to learn the valid day names from the `links`
hrefs.

`/schedules/{day}` returns an array of scheduled slots. Each slot:

| Field | Notes |
|-------|-------|
| `fromDate` / `toDate` | UTC ISO timestamps (`...Z`) |
| `timezone` | IANA zone, e.g. `Europe/Brussels`, `Europe/London` — use for display |
| `room` | `{id, name, weight, capacity}` |
| `sessionType` | `{name, slug, duration, pause, ...}` — `pause:true` marks breaks/lunch/registration |
| `proposal` | the talk (`id, title, description, audienceLevel, totalFavourites, track, speakers, ...`) — null for breaks; `proposal.id` joins to `/talks` and is the talkId for links |
| `speakers` | array (may be empty on the slot; also check `proposal.speakers`) |

Example — a day's program in event-local time, optionally filtered by room,
with talk links:

```bash
curl -s -m 30 "https://dvbe25.cfp.dev/api/public/schedules/wednesday" | python3 -c '
import sys, json, re
from datetime import datetime
from zoneinfo import ZoneInfo
EVENT = "dvbe25"
room_filter = None  # e.g. "Room 8"
def slug(s): return re.sub(r"[^a-z0-9]+", "-", (s or "").lower()).strip("-")
d = json.load(sys.stdin)
def local(ts, tz):
    return datetime.fromisoformat(ts.replace("Z","+00:00")).astimezone(ZoneInfo(tz or "UTC")).strftime("%H:%M")
rows = []
for s in d:
    if room_filter and (s.get("room") or {}).get("name") != room_filter: continue
    p = s.get("proposal") or {}
    tz = s.get("timezone")
    title = p.get("title") or (s.get("sessionType") or {}).get("name") or "-"
    if p.get("id"):
        title = "[{}](https://m.devoxx.com/events/{}/talks/{}/{})".format(title, EVENT, p["id"], slug(title))
    spk = ", ".join(x.get("fullName","") for x in (s.get("speakers") or p.get("speakers") or []))
    rows.append((s["fromDate"], local(s["fromDate"], tz), local(s["toDate"], tz),
                 (s.get("room") or {}).get("name",""), title, spk))
for _, a, b, room, title, spk in sorted(rows):
    line = "{}-{}  {}  {}".format(a, b, room, title)
    if spk: line += " - " + spk
    print(line)
'
```

## GET /rooms and GET /tracks

`/rooms`: array of `{id, name, weight, capacity}`.
`/tracks`: array of `{id, name, description, imageURL}`. Fetch live per event.

---

## Public website links (mandatory in answers)

Every talk and speaker mentioned in an answer must link to `m.devoxx.com`
using the resolved event slug:

- Talk: `https://m.devoxx.com/events/{event-slug}/talks/{talkId}/{slug(title)}`
- Speaker: `https://m.devoxx.com/events/{event-slug}/speaker/{speakerId}/{slug(fullName)}` — path segment is `speaker` (singular)

Routing is by id (the slug segment is cosmetic), but generate a clean slug:
lowercase, collapse runs of non-alphanumerics to single hyphens, strip edge
hyphens:

```python
import re
def slug(s): return re.sub(r"[^a-z0-9]+", "-", (s or "").lower()).strip("-")
```

Verified examples:
- `https://m.devoxx.com/events/dvbe25/talks/7729/top-10-event-driven-architecture-pitfalls`
- `https://m.devoxx.com/events/dvbe25/speaker/8101/victor-rentea`

## Tips

- Cache within a task: fetch `/talks` and the relevant `/schedules/{day}` once
  and reuse; cache the resolved event slug for the whole conversation.
- Bios and descriptions contain HTML — strip tags before showing to the user.
- For "is X speaking?" search `/talks` speakers or page `/speakers`; for
  "what is X giving?" use `/speakers/{id}` → `proposals`.
- An announced event with `apiURL: null` has no CFP data yet — say so and offer
  the most recent edition instead.
