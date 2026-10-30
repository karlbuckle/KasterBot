# KasterBot

A Discord bot that live-casts our Dota 2 unranked turbo games into our voice channel, using
Game State Integration (GSI) for match data, an LLM for commentary, and local TTS for voice.

**Status: planning phase.** No application code yet. Do not start building unless the user says so.

## Planning docs (source of truth — read before working)

| File | What it holds |
|---|---|
| [docs/ROADMAP.md](docs/ROADMAP.md) | Build phases, session split, current status |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Pipeline, components, data model, event detector, director, prompts |
| [docs/DATA_SOURCES.md](docs/DATA_SOURCES.md) | Why GSI, what it does and doesn't give us, config file |
| [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) | Docker / compose / Dockhand, networking, secrets |
| [docs/DECISIONS.md](docs/DECISIONS.md) | Dated log of decisions made and why |
| [docs/OPEN_QUESTIONS.md](docs/OPEN_QUESTIONS.md) | Unresolved questions and things to verify |

## Rules for every session

- **Keep the docs current.** When a decision is made, add it to `docs/DECISIONS.md` and update the
  doc it affects. When a question is answered, move it out of `OPEN_QUESTIONS.md`.
- **The repo is public.** Never commit tokens, API keys, `.env`, or `config/friends.yaml`.
  Commit only `*.example` versions.
- **Shared interfaces** (`state/events.py`, `state/match.py`, and the component boundaries in
  ARCHITECTURE.md) are owned by the lead session. Component sessions don't change them
  unilaterally — raise it with the lead session so dependents get updated.
- **Stay in your folder.** Component sessions work only in the folders the roadmap assigns them.
- **Unverified ≠ fact.** Items marked _(verify)_ in the docs are assumptions until the spike or
  testing confirms them. Update the doc when confirmed.
- Language: Python (asyncio). Dev on Windows; deploy as Docker Compose via Dockhand.
- This folder lives in OneDrive; keep it set to "Always keep on this device" to avoid git lock issues.
