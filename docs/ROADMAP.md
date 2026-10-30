# Roadmap

**Current phase: Planning.** Update the status column as work progresses.

| # | Phase | Status |
|---|---|---|
| 0 | Planning docs | Done |
| 1 | Spike: record raw GSI from one real turbo game | Not started |
| 2 | Lead session: finalise shared interfaces from spike data | Not started |
| 3 | Parallel component builds (sessions below) | Not started |
| 4 | Integration: replay a recorded match end-to-end into Discord voice | Not started |
| 5 | Live test in a real game, tune pacing and thresholds | Not started |
| 6 | Friends: installer, multi-feed merge, external networking | Later |
| 7 | Extras | Later |

## Phase 1 — Spike

Tiny aiohttp server (or container on the Docker host) that logs every GSI payload as JSONL.
Install the GSI cfg on Karl's PC, play one turbo game, then review:
- which fields actually exist (update DATA_SOURCES.md)
- update frequency and payload size
- whether `-gamestateintegration` is needed
- the PC → Docker host network path

Must happen before splitting into component sessions — the interfaces should be built on real data.

## Phase 2 — Shared interfaces (lead session)

Write real code for `state/events.py`, `state/match.py`, and stub the component boundaries in
ARCHITECTURE.md. Also: repo skeleton, `pyproject`/requirements, `.env.example`,
`config/friends.example.yaml`.

## Phase 3 — Component sessions

Each in its own git worktree / branch, owning only its folders. Report to the lead session.

| Session | Owns | Depends on |
|---|---|---|
| **Data** | `gsi/`, `state/match.py` impl, `state/detector.py`, `tools/replay_log.py` | Interfaces + spike logs |
| **Commentary** | `director/`, `commentary/` | Interfaces (test with fake events) |
| **Voice & Deploy** | `voice/`, bot side of `main.py`, `Dockerfile`, `compose.yaml`, GHCR workflow | Interfaces (mostly independent) |

## Phase 4 — Integration (lead session)

Merge branches; run `tools/replay_log.py` on a recorded match through the full pipeline and hear
it in Discord.

## Phase 6 — Friends (later)

- `tools/install_gsi.py`: finds Dota install via Steam registry + `libraryfolders.vdf`, writes cfg with
  the friend's token and server URL, reminds about launch option.
- Multi-feed merge and kill attribution across feeds.
- External networking (Tailscale or Cloudflare Tunnel — see DEPLOYMENT.md).

## Phase 7 — Extras (later)

- Post-game recap from OpenDota once the match is parsed.
- Dual casters (play-by-play + color) with two voices.
- Cross-game lore / running jokes (`data/history`).
- Bot commands (`/cast start`, `/cast mute`, `/cast persona ...`) vs auto-join on match start.
- Paid TTS option.
