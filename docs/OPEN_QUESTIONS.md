# Open Questions

Move items to DECISIONS.md (or the relevant doc) once answered.

## Need Karl's answer

- **Docker host:** is it the gaming PC (Docker Desktop) or a separate LAN machine? CPU? NVIDIA GPU?
  (Decides CPU vs GPU Kokoro, and whether Kokoro load competes with Dota.)
- **Dockhand deployment style:** paste compose, git-based stack, or image auto-update? (Decides the
  GHCR workflow vs building in Dockhand.)
- **Caster persona:** classic esports caster, roast-your-plays, or something else? How much friend
  lore to write?
- **Bot control:** slash commands (`/cast start`, `/cast mute`) or auto-join when a match starts?

## To verify (spike / testing)

- Exact GSI fields present for a player client in a turbo game; update frequency.
- Whether the `-gamestateintegration` launch option is required.
- Whether `discord.py` supports Discord's DAVE E2EE voice protocol (else use `py-cord`).
- Kokoro-FastAPI: current image tag, voice list, latency on the chosen host's CPU.
- (Later) whether GSI accepts `https://` URIs — matters for Cloudflare Tunnel.
