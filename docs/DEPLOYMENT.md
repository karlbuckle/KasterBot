# Deployment

Deployed as a Docker Compose stack managed in **Dockhand**.

## Current scope: LAN only

Only Karl's PC sends GSI, over the local network. No external networking (no Tailscale,
Cloudflare Tunnel, or port forwarding) for now.

- **Docker on the gaming PC:** GSI `uri` = `http://127.0.0.1:3000/gsi`.
- **Docker on another LAN machine:** GSI `uri` = `http://<host-ip>:3000/gsi`. Give the host a DHCP
  reservation and allow inbound TCP 3000 in its firewall.
- Discord and Anthropic API traffic is outbound only — no other ports needed.
- Keep token auth on `/gsi` anyway so adding friends later needs no code change.

## Containers

| Service | Image | Role |
|---|---|---|
| `app` | `ghcr.io/karlbuckle/kasterbot:latest` | Receiver, state, detector, director, LLM, Discord bot |
| `tts` | `ghcr.io/remsky/kokoro-fastapi-cpu:<pinned-tag>` | Kokoro TTS behind an OpenAI-compatible API. Not published to host |

Swap to the GPU Kokoro image if the host has an NVIDIA GPU. Swapping to a paid TTS later = change
`TTS_URL` and key.

## Compose (draft)

```yaml
services:
  app:
    image: ghcr.io/karlbuckle/kasterbot:latest
    env_file: .env
    environment:
      TTS_URL: http://tts:8880/v1
    ports:
      - "3000:3000"            # GSI in
    volumes:
      - ./config/friends.yaml:/app/config/friends.yaml:ro
      - ./data:/app/data
    depends_on: [tts]
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:3000/health')"]
      interval: 30s
    restart: unless-stopped

  tts:
    image: ghcr.io/remsky/kokoro-fastapi-cpu:<pinned-tag>
    restart: unless-stopped
```

## Secrets and config (repo is public)

- `.env` (set in Dockhand's env editor, never committed): `DISCORD_TOKEN`, `ANTHROPIC_API_KEY`,
  later any TTS/network keys. Commit `.env.example` only.
- `config/friends.yaml` (mounted read-only, never committed or baked into the image): GSI token →
  friend name, steamid, Discord ID, lore. Commit `config/friends.example.yaml` only.
- Volumes: `./data/gsi-logs` (recorded matches), `./data/history` (cross-game lore, later).

## Image build

Preferred: GitHub Actions builds on push to `main` → pushes to GHCR (free for public repos) →
Dockhand pulls/redeploys. Alternative: Dockhand builds from the repo Dockerfile, if it supports
git stacks _(see OPEN_QUESTIONS)_.

## Dev loop

- Day-to-day: run the app with plain Python on Windows, pointed at the TTS container. Faster restarts.
- Replay tool runs the same either way, e.g. `docker compose run app python -m tools.replay_log ...`.
- Docker is the deploy target, not required for every edit.

## Ops

- `restart: unless-stopped` on all services.
- `/health` endpoint on the app, used as the compose healthcheck.
- Log to stdout so logs show in Dockhand.

## Later: external networking (when friends join)

Friends' PCs need to reach `/gsi`. Options recorded for later:
- **Tailscale sidecar** (`tailscale/tailscale`, app uses `network_mode: service:tailscale`) — private,
  friends install Tailscale. Note: with shared network mode, app can't reach `tts` by service name; needs
  a network workaround.
- **Cloudflare Tunnel sidecar** (`cloudflared`) — friends install nothing, needs a Cloudflare domain;
  public endpoint protected by tokens. _(verify GSI supports https uris)_
