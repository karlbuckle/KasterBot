# Architecture

## Overview

One Python asyncio process (the `app` container) plus a separate TTS container.

```
 Karl's PC (Dota 2 + GSI cfg)          [later: each friend's PC]
   │  HTTP POST JSON (auth token), LAN
   ▼
┌───────────────────────── app container ─────────────────────────┐
│ 1. GSI Receiver     aiohttp POST /gsi, token auth, tag by steamid│
│ 2. State Store      merged MatchState (keyed by steamid)         │
│ 3. Event Detector   diff old vs new snapshot → Events            │
│ 4. Director         priority queue, pacing, merging, interrupts  │
│ 5. Commentary       Claude Haiku, streamed, one line per event   │
│ 6. Voice            TTS client → 48kHz PCM → Discord voice       │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTP (OpenAI-compatible speech API)
                       ┌───────▼────────┐
                       │ tts container  │  Kokoro-FastAPI (CPU)
                       └────────────────┘
```

Outbound only: Discord gateway/voice, Anthropic API. Inbound only: GSI on port 3000.

## Scope (current)

- **One feed:** only Karl's client sends GSI. Friends' feeds come later (see ROADMAP).
- **Same team only:** all friends queue together; no cross-team fairness logic needed.
- **Live:** commentary is heard by players while playing, in the Discord voice channel we all use.
- **Design for multi-feed anyway:** players keyed by steamid, payloads tagged by token, so adding
  friends later is config, not code.

## Tech stack

| Part | Choice | Notes |
|---|---|---|
| GSI receiver | `aiohttp` | Same event loop as the bot |
| Discord bot + voice | `discord.py[voice]` | _(verify)_ DAVE E2EE voice support; fallback `py-cord` |
| LLM | Claude Haiku via Anthropic API | Streamed; prompt caching for persona/lore |
| TTS | Kokoro-FastAPI container (CPU) | OpenAI-compatible API → swappable for paid TTS later. Piper is the light fallback |
| Audio | `numpy` | Resample to 48kHz stereo s16le, 20ms frames |

## Project layout (planned)

```
caster/
  main.py              # starts aiohttp + discord bot on one event loop
  config.py            # env vars, friends.yaml loading
  gsi/
    server.py          # POST /gsi, auth check, /health
    models.py          # tolerant typed parsing of GSI payloads
  state/
    match.py           # MatchState, PlayerState            (SHARED INTERFACE)
    events.py          # Event, EventType                   (SHARED INTERFACE)
    detector.py        # snapshot diff → events
  director/
    director.py        # priority queue, pacing, merge, interrupts
  commentary/
    llm.py             # prompt building + streaming Haiku call
    personas.py        # caster persona, friend nicknames/lore
  voice/
    tts.py             # Kokoro-FastAPI client
    player.py          # custom discord AudioSource, interruptible queue
  tools/
    replay_log.py      # replay recorded GSI logs through the pipeline offline
    install_gsi.py     # (later) friend-side cfg installer
config/
  friends.example.yaml # token → friend name, discord id, lore (real file is gitignored)
data/
  gsi-logs/            # raw recorded GSI payloads (JSONL per match)
```

`tools/replay_log.py` is a top priority: record every payload, then replay full matches through
detector → director → LLM → voice offline, sped up, without launching Dota.

## Component boundaries (shared interfaces)

Draft — finalise in the lead session before splitting work.

- Receiver → State: `state.apply(token_or_steamid, payload: dict) -> None`
- State → Detector: `detector.diff(old: MatchState, new: MatchState) -> list[Event]`
- Detector → Director: `director.submit(event: Event) -> None`
- Director → Commentary: `commentary.generate(event, state, recent_lines) -> AsyncIterator[str]`
  (yields complete sentences)
- Director → Voice: `voice.speak(sentences: AsyncIterator[str], priority) -> None`,
  `voice.interrupt() -> None`, `voice.is_speaking -> bool`

## Data model (draft)

```python
@dataclass
class PlayerState:          # one per feed
    steamid: str
    name: str               # friend nickname from friends.yaml
    hero: str
    level: int; alive: bool; respawn_seconds: int
    health_pct: float; mana_pct: float
    kills: int; deaths: int; assists: int; kill_streak: int
    gold: int; last_hits: int; gpm: int; xpm: int
    pos: tuple[int, int]
    items: list[str]; ult_ready: bool; buyback_cd: int
    last_update: float      # wall clock, for feed-dropout detection

@dataclass
class MatchState:
    match_id: str
    game_state: str         # draft / pre-game / in-progress / post-game
    clock: int; daytime: bool
    our_team: str           # radiant / dire
    our_score: int; their_score: int
    roshan_state: str
    players: dict[str, PlayerState]   # keyed by steamid
    buildings: dict                   # our towers / rax
    history: deque[Event]

@dataclass
class Event:
    type: EventType
    priority: int           # 0 filler … 100 game-ending
    game_time: int
    created_at: float
    ttl: float              # seconds until stale
    actors: list[str]       # friend names
    data: dict
```

Reset state on new `match_id` / game_state transitions.

## What the caster can know (single feed)

| Known | Unknown |
|---|---|
| Everything about Karl's hero: K/D/A, HP, position, items, level, ult, buyback | Friends' heroes, items, HP, positions |
| Team kill scores → "we got a kill", "we lost someone" | Which friend got the kill / died |
| Clock, day/night, Roshan, our buildings, draft | Fights Karl isn't in |

The LLM prompt must state these limits so it doesn't invent friend actions or enemy victims.

## Event detector

Diffs consecutive snapshots. Thresholds are starting points, tuned against replayed logs.

| Event | Detection | Priority |
|---|---|---|
| Karl kill | Karl's `kills` up (and `our_score` up) | 50 |
| Teammate kill | `our_score` up, Karl's kills not up (assist if Karl's assists up) | 35 |
| Karl dies | `deaths` up | 50 |
| Teammate dies | `their_score` up, Karl's deaths not up | 35 |
| Multi-kill / rampage | Karl kill increments within ~10s | 80–95 |
| Teamfight | ≥3 total score changes within ~12s (merges individual kills) | 85 |
| Streak ended | Karl's `kill_streak` ≥3 → 0 | 60 |
| Close escape | HP <10%, alive 4s later with HP rising | 55 |
| Ult online | Level 6 (maybe 12/18 — turbo is fast) | 30 |
| Big item | Item from a shortlist appears (Blink, BKB, Aghs, Refresher, Rapier…) | 40 (Rapier 90) |
| Buyback | Dead → alive with respawn time remaining + buyback on cooldown | 75 |
| Roshan / Aegis | `roshan_state` change or GSI `events` | 70 |
| Our building lost | Building gone / health 0 | 60 (rax 75, ancient 100) |
| Game end | `win_team` set | 100 |
| Draft pick | Hero locked during draft | 40 |
| Lull | No event for ~25s | 10 (color: farm, clock, banter) |

Feed dropout: no update from a feed for >10s → mark that player unknown, don't commentate on them.

## Director

- **One speaker:** only one line plays at a time.
- When the current line ends, pick the highest-priority non-stale event; merge bursts (five kills →
  one teamfight call); drop expired events (TTL).
- **Interrupt** if an incoming event outranks the current line by ≥30: hard-stop audio, short cut-off.
- **Cooldowns:** e.g. same player mentioned at most every 15s; item callouts at most every 30s.
- **Filler** only after silence, lowest priority, never blocks real events.
- **Segments:** draft/pre-game intro ("here's our lineup"), post-game wrap-up from final stats.
- Turbo is event-dense: most of the director's job is deciding what *not* to say.

## Commentary (LLM)

- Model: Claude Haiku (fast first token).
- **System prompt (cached):** caster persona, rules, friend lore. Rules include: one line, ≤25 words;
  only state facts in the provided data; never name an enemy hero unless provided; never invent numbers;
  respect single-feed limits.
- **User turn:** compact state JSON + last ~5 spoken lines (avoid repetition) + the event.
- Stream output; split on sentence boundaries; send each sentence to TTS as soon as complete.
- Later: dual casters (play-by-play + color) with different voices.

## Voice

- Kokoro-FastAPI returns audio (24kHz mono). Convert to 48kHz stereo s16le, 20ms frames.
- Custom `discord.AudioSource` reading from a queue; `interrupt()` clears queue with a brief fade.
- TTS calls are async HTTP; no blocking the event loop.

## Latency budget

| Stage | Estimate |
|---|---|
| GSI delivery | 0.1–0.5s |
| Detector + director | ~0 |
| Haiku first sentence | 0.5–1s |
| Kokoro first sentence (CPU) | 0.2–0.5s |
| Discord playback | ~0.1s |
| **Total** | **~1–2s** |
