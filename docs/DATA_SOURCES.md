# Data Sources

Our games are unranked turbo matchmaking games — not league or top-tier live games — so most
Dota data APIs don't help in real time.

| Source | Live? | Usable for us? |
|---|---|---|
| OpenDota / STRATZ API | No — post-match, after replay parse | Post-game recaps only (later) |
| Steam `GetLiveLeagueGames` / `GetRealtimeStats` | Yes | No — league / top live games only |
| Screen capture + vision model | Yes | Possible but slow, costly, brittle. Not planned |
| **Game State Integration (GSI)** | **Yes, sub-second** | **Yes — primary source** |
| Spectator client GSI (all 10 heroes) | ~2 min broadcast delay on matchmade games | No — too delayed for live casting to players |

## Game State Integration

Valve's official, sanctioned feature. The Dota client POSTs JSON game state to an HTTP URL we
configure, whenever state changes. No memory reading, no VAC risk.

### Config file

Path: `...\steamapps\common\dota 2 beta\game\dota\cfg\gamestate_integration\gamestate_integration_kasterbot.cfg`

```
"KasterBot"
{
    "uri"       "http://<docker-host-ip>:3000/gsi"
    "timeout"   "5.0"
    "buffer"    "0.1"
    "throttle"  "0.1"
    "heartbeat" "30.0"
    "auth" { "token" "<per-player-secret>" }
    "data"
    {
        "provider"  "1"
        "map"       "1"
        "player"    "1"
        "hero"      "1"
        "abilities" "1"
        "items"     "1"
        "events"    "1"
        "buildings" "1"
        "draft"     "1"
    }
}
```

- `uri`: `http://127.0.0.1:3000/gsi` if Docker runs on the gaming PC; otherwise the Docker host's LAN IP.
- Steam launch option `-gamestateintegration` is likely required _(verify)_.

### What a player's client provides _(verify all in spike)_

- **Global:** game clock, day/night, Radiant/Dire kill scores, game state, winner, Roshan state, match id.
- **Own player:** K/D/A, last hits, denies, gold, GPM/XPM, kill streak.
- **Own hero:** level, HP/mana, alive/respawn, position, buffs (stunned/silenced etc.), items, ability cooldowns.
- **Own team's buildings**, draft info, some `events` (Roshan kill, Aegis, bounty runes).

### What it doesn't provide

- Other players' data (teammates or enemies) — by design, to prevent cheating.
- Who killed whom, beyond inference from counter changes.
- Full 10-player data (`allplayers`, minimap) is spectator/observer only.

### Multi-feed (later)

Each friend runs GSI with their own token, all pointing at our server. Merging feeds gives full
data for our whole team and allows kill attribution by correlating counter changes in the same
update window. Enemy team (randoms) remains mostly opaque. Since we all queue on one team, there's
no info-leak concern.

## Spike findings

_Not yet run. Record findings here: actual fields present, update frequency, quirks, sample
payloads (store full logs in `data/gsi-logs/`, gitignored)._
