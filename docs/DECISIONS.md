# Decisions

Newest at the bottom. Format: date — decision — why.

- **2026-10-30 — Use Dota 2 Game State Integration as the live data source.** Our unranked turbo
  games aren't covered by live APIs (league/top-game only); post-match APIs are too late; spectator
  GSI has a ~2 min delay. GSI is official, sub-second, and VAC-safe.
- **2026-10-30 — Player-client GSI feeds, not a spectator client.** Casting is heard live by the
  players, so the spectator delay is unacceptable.
- **2026-10-30 — Same-team only.** We always queue together; no cross-team info-leak handling needed.
- **2026-10-30 — Live voice in our Discord voice channel.** That's where we all are while playing.
- **2026-10-30 — Python, single asyncio process** for receiver, state, detector, director, LLM, and bot.
  Karl's most familiar language.
- **2026-10-30 — Claude Haiku for commentary.** Fast first token; a local LLM would compete with Dota for
  the GPU. Cost per game is small.
- **2026-10-30 — Free local TTS to start: Kokoro.** Paid TTS can be revisited later.
- **2026-10-30 — Kokoro via the Kokoro-FastAPI container** rather than in-process. Keeps the app image
  small, and its OpenAI-compatible API makes swapping to paid TTS a config change.
- **2026-10-30 — Deploy with Docker Compose via Dockhand.** Karl's preferred Docker manager.
- **2026-10-30 — LAN only, single feed (Karl's PC) for the early build.** Simplicity. No external
  networking yet. Data model stays multi-feed-ready (keyed by steamid, token-tagged payloads).
- **2026-10-30 — Repo `karlbuckle/KasterBot` (public), cloned into this OneDrive Caster folder.**
  Secrets never committed; `.env` and `config/friends.yaml` are gitignored.
- **2026-10-30 — Multiple Claude Code sessions per component, coordinated by a lead session.**
  Planning docs and shared interface code in the repo are the source of truth; session messages are
  for coordination only.
