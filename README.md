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

## Storage layout (host)

| Path | Purpose |
| --- | --- |
| `/mnt/roms` | Library root (external HDD, NTFS, UUID `E8CA8FAACA8F739A`) — platform folders live at `/mnt/roms/roms/<platform>/` (RomM Structure A) |
| `/mnt/roms/assets` | Saves, states, uploads — same HDD |
| `/home/rob/romm/config` | `config.yml` — local disk (always available) |

Mount is in `/etc/fstab` (`ntfs3`, `uid=1000,gid=1000`, `nofail`), so the host boots fine
if the drive is unplugged. If the drive is ever missing while the container restarts,
the library/saves will appear empty — nothing is lost, just remount and rescan.

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

## Secret scanning

Secrets are scanned **before** they reach GitHub, not after. `.gitignore` only
covers `.env`; the scanner catches a key that leaks some other way (pasted into
the README, a new file, `ts-serve.json`).

One-time setup per clone:

```bash
pip install pre-commit   # or: pipx install pre-commit
pre-commit install
```

Run every hook against the whole tree at any time:

```bash
pre-commit run --all-files
```

Hooks are gitleaks ([`gitleaks.toml`](gitleaks.toml)) plus private-key and
large-file guards. `gitleaks.toml` extends the built-in ruleset with a
**Tailscale `tskey-…` rule** — gitleaks has no Tailscale rule by default, and a
Tailscale auth key can join a node to your tailnet, so it's the most sensitive
credential in this stack. The third-party API keys this stack uses
(ScreenScraper, SteamGridDB, RetroAchievements) are covered by the stock rules.

`git commit --no-verify` skips the hooks. CI is the backstop for that: the
`secrets` job scans full history on every push and PR.

## CI

[`.github/workflows/ci.yml`](.github/workflows/ci.yml) runs two jobs:

| Job | What it does |
|-----|--------------|
| `secrets` | gitleaks full-history scan (`fetch-depth: 0`) — backstop for `--no-verify` |
| `validate` | `docker compose config`, `ts-serve.json` JSON validity, and a check that `.env.example` documents every variable `compose.yaml` references |

It is read-only on purpose: there is no build, no deploy, and no
`docker compose up`. The live deployment owns the ROM library and volumes.

### Making the checks required for PRs

After the workflow has run at least once, require both jobs in
**Settings → Branches** (or **Rulesets**) for `main`: enable *Require status
checks to pass before merging* and select `secrets` and `validate`.

Two caveats: required checks only gate **pull requests** — pushing directly to
`main` bypasses them, so add a rule blocking direct pushes if you want them
enforced on your own work too. And GitHub only offers a check in the picker
once it has run at least once.

## Files

| File | Purpose |
|------|---------|
| `compose.yaml` | tailscale sidecar + RomM + MariaDB |
| `ts-serve.json` | `tailscale serve` config (tailnet-only) |
| `.env` / `.env.example` | secrets/config (`.env` is git-ignored) |
| `.pre-commit-config.yaml` | gitleaks + hygiene hooks |
| `gitleaks.toml` | gitleaks rules (adds Tailscale keys) |
| `.github/workflows/ci.yml` | `secrets` + `validate` CI |
