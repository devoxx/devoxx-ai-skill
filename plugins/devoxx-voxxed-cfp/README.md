# Devoxx / VoxxedDays CFP

Ask plain-language questions about any Devoxx or VoxxedDays conference and get
answers from the events' live public APIs. No setup, no login.

## What you can ask

- **Events** — "When is the next Devoxx?", "Which Voxxed Days are coming up?"
- **Speakers** — "Tell me about Victor Rentea", "What is Venkat giving at Devoxx UK?"
- **Talks** — "Find Kubernetes talks at Devoxx Belgium", "Most popular AI talks at Voxxed Zurich?"
- **Schedule** — "What's on Wednesday at Devoxx UK?", "What's in Room 8 on Thursday?"

## How it works

The plugin adds one skill, **devoxx-cfp**. It discovers events (and their API
slugs) from the devoxxians.com registry, then queries that event's public CFP
API (`{slug}.cfp.dev`) for speakers, talks, schedules, rooms, and tracks.
Every talk and speaker in an answer links to its page on m.devoxx.com, and
times are shown in the event's local timezone.

## Notes

- Data is read-only and public; nothing is written back.
- Newly announced events may not have a live CFP yet — the skill says so and
  offers the most recent edition instead.
