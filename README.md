# romm

RomM (game ROM manager + emulator frontend) on the Tailscale-connected host at
`192.168.4.37`, reachable **only** over the tailnet.

Uses the same pattern as `neon-crate`: a Tailscale sidecar container whose
network namespace is shared by the app. `tailscale serve` (not funnel) fronts
the UI, so there is no public exposure — neither a Funnel URL nor any published
host ports on the LAN interface.

## Result

| Item | Value |
| --- | --- |
| URL | `https://romm.<your-tailnet>.ts.net` |
| Access | Tailnet only (MagicDNS + HTTPS cert) |
| App port (inside netns) | `8080` |
| DB | MariaDB, same netns (`127.0.0.1:3306`) |

## Quickstart (on the host)

1. Copy this directory to the host, e.g. `~/romm`.

2. Create `.env` from `.env.example` and fill in:
   - `TAILSCALE_AUTH_KEY` — tailnet pre-auth key (the node joins as `romm`)
   - `ROMM_AUTH_SECRET_KEY` — `openssl rand -hex 32`
   - DB passwords
   - Optional: Screenscraper / RetroAchievements / SteamGridDB API keys

3. Adjust `ROMM_LIBRARY_DIR` to wherever your ROMs live (or will live).
   RomM expects the folder structure `library/<platform>/<roms>` (e.g.
   `library/gba/`, `library/snes/`).

4. Bring it up:

   ```bash
   docker compose up -d
   ```

5. Verify:

   ```bash
   docker compose ps
   docker exec romm-tailscale tailscale status   # node "romm" should be up
   curl -sk https://romm.<your-tailnet>.ts.net | head   # reachable via tailnet
   ```

## Notes

- No `ports:` are published anywhere — the stack shares one network namespace
  with the Tailscale container, so it is unreachable from the LAN/internet
  except through `tailscale serve`.
- `tailscale serve` uses `TS_SERVE_CONFIG` (`ts-serve.json`) proxying
  `443 -> http://127.0.0.1:8080` inside the netns. HTTPS certificates are
  auto-issued; make sure **HTTPS Certificates** is enabled in the tailnet
  admin console (it is, since `neon-crate` already relies on it).
- To change the hostname: edit `hostname:` under the `tailscale` service,
  re-run `docker compose up -d` — the URL becomes `https://<hostname>.<tailnet>.ts.net`.
- Updating: `docker compose pull && docker compose up -d`.
- Secrets in `.env` are consumed via compose variable substitution
  (`${VAR}`) — do not commit the real `.env`.
