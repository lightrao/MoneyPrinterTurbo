# HP450 Deployment

This document describes the LAN-only deployment of the fork's
`feat/siliconflow-cloned-voice-v2` branch on the HP450 Debian 13 Docker host.

A release-by-release summary of what landed in this fork lives in
`../CHANGELOG.md`. The companion Catalog project documents live in
`https://github.com/lightrao/MPTMaterialCatalog` (`docs/architecture.md`,
`docs/api-contract.md`, `CHANGELOG.md`).

## Topology

```text
GitHub Actions (linux/amd64 build)
        |
        v
public GHCR image: ghcr.io/lightrao/moneyprinterturbo:sha-<full-sha>
        |
        v
HP450 /data/projects/moneyprinterturbo
        |
        +-- 127.0.0.1:8501  Streamlit WebUI
        +-- 127.0.0.1:8080  FastAPI
        |
        +-- Mac SSH local forwarding for LAN access
```

The host is not exposed directly on the LAN or Internet. Do not add a
Cloudflare Tunnel, Access application, reverse proxy, or firewall rule as part
of this initial deployment.

## Image policy

- GitHub Actions publishes the fork image to `ghcr.io/lightrao/moneyprinterturbo`.
- The production Compose file must reference `sha-<40-character-commit-sha>`.
- Do not deploy `latest` or a branch tag. Those tags are navigation aids only.
- The image is `linux/amd64`; HP450 is an Intel `amd64` host.
- The workflow uses the official Debian/PyPI mirrors in GitHub Actions. HP450
  must not build this image locally because Docker Hub access is unreliable.
- `Dockerfile.gpu` is not used. HP450 has no configured NVIDIA runtime.

## Build and publish

Push the deployment changes to `feat/siliconflow-cloned-voice-v2`, wait for the
Python 3.11, Python 3.13, and Windows CI jobs to pass, then manually run
`Publish Docker image` from that branch in GitHub Actions.

If GitHub does not expose `workflow_dispatch` for a workflow that exists only
on this feature branch, create a reviewed `v*` deployment tag on the exact
CI-passing commit. The tag event is the release gate used for this fork; the
server still deploys only the generated full `sha-` tag, never the tag itself.

Confirm that the published artifact is:

```text
ghcr.io/lightrao/moneyprinterturbo:sha-<full-commit-sha>
```

The package should be public so HP450 can pull it without a GitHub token. Never
put a PAT in the repository, image, Compose file, or HP450 configuration.

If direct HP450 layer transfer stalls, use the LAN fallback validated on this
host. The Mac does not need Docker Desktop for this path:

```bash
/opt/homebrew/bin/crane pull --platform linux/amd64 --format tarball \
  ghcr.io/lightrao/moneyprinterturbo:sha-<full-commit-sha> \
  /private/tmp/moneyprinterturbo-<short-sha>.tar

tar -tf /private/tmp/moneyprinterturbo-<short-sha>.tar >/dev/null
scp /private/tmp/moneyprinterturbo-<short-sha>.tar \
  hp450-lan:/data/projects/moneyprinterturbo/moneyprinterturbo-<short-sha>.tar.new
```

Compare SHA-256 on both sides before loading the temporary file. After the
image is loaded and healthchecks pass, remove the archive and start Compose
with `--pull never` so a transient GHCR issue cannot trigger another pull.

## HP450 layout

```text
/data/projects/moneyprinterturbo/
├── compose.yml
├── .env                         # MPT_IMAGE only, not tracked
├── config.toml                  # credentials, mode 0600, not tracked
├── storage/                     # task outputs and media cache
└── backups/                     # timestamped config and deployment backups
```

The Docker engine data root remains `/data/docker`. Existing SAU, tusd,
OpenResty, and 1Panel containers are separate and must not be stopped by MPT
operations.

## First deployment

Use `hp450-lan` for LAN operations. Do not rely on `hp450g4.local` as the
production path because macOS mDNS can be degraded by the active network
extensions.

Create the directories on HP450:

```bash
ssh hp450-lan 'set -eu
  sudo install -d -m 0750 -o lightrao -g docker /data/projects/moneyprinterturbo
  sudo install -d -m 0770 -o lightrao -g docker /data/projects/moneyprinterturbo/storage
  sudo install -d -m 0700 -o lightrao -g lightrao /data/projects/moneyprinterturbo/backups'
```

Copy the checked-in Compose file and environment template. The actual `.env`
must contain the final full SHA and nothing else:

```bash
scp deploy/hp450/compose.yml hp450-lan:/data/projects/moneyprinterturbo/compose.yml
scp deploy/hp450/env.example hp450-lan:/data/projects/moneyprinterturbo/env.example
ssh hp450-lan 'chmod 0644 /data/projects/moneyprinterturbo/compose.yml'
```

Transfer `config.toml` separately. Do not print it or commit it:

```bash
scp -p config.toml hp450-lan:/data/projects/moneyprinterturbo/config.toml.new
ssh hp450-lan 'set -eu
  test -s /data/projects/moneyprinterturbo/config.toml.new
  chmod 0600 /data/projects/moneyprinterturbo/config.toml.new
  if test -f /data/projects/moneyprinterturbo/config.toml; then
    cp -p /data/projects/moneyprinterturbo/config.toml \
      /data/projects/moneyprinterturbo/backups/config.toml.bak-$(date -u +%Y%m%dT%H%M%SZ)
  fi
  mv -f /data/projects/moneyprinterturbo/config.toml.new \
    /data/projects/moneyprinterturbo/config.toml
  chmod 0600 /data/projects/moneyprinterturbo/config.toml'
```

Create `.env` with the immutable image tag after the Actions run succeeds:

```bash
ssh hp450-lan 'umask 077; printf "%s\n" \
  "MPT_IMAGE=ghcr.io/lightrao/moneyprinterturbo:sha-<full-commit-sha>" \
  > /data/projects/moneyprinterturbo/.env'
ssh hp450-lan 'chmod 0640 /data/projects/moneyprinterturbo/.env'
```

Validate the rendered Compose model before starting anything:

```bash
ssh hp450-lan 'cd /data/projects/moneyprinterturbo && \
  docker compose --env-file .env -f compose.yml config'
```

Pull and start only this Compose project:

```bash
ssh hp450-lan 'set -eu
  cd /data/projects/moneyprinterturbo
  docker compose --env-file .env -f compose.yml pull
  docker compose --env-file .env -f compose.yml up -d --pull never --remove-orphans'
```

Do not use `docker system prune`, `docker compose down` in another project,
`--volumes`, or a global Docker restart.

## LAN access

Keep both host bindings on loopback and use an SSH forward from the Mac:

```bash
ssh -N \
  -L 18501:127.0.0.1:8501 \
  -L 18080:127.0.0.1:8080 \
  hp450-lan
```

Open `http://127.0.0.1:18501` for the WebUI and
`http://127.0.0.1:18080/docs` for API documentation. Ports 8501 and 8080
should remain unreachable through the server's LAN address.

## Health and smoke checks

```bash
ssh hp450-lan 'cd /data/projects/moneyprinterturbo && \
  docker compose --env-file .env -f compose.yml ps'
ssh hp450-lan 'curl -fsS http://127.0.0.1:8501/_stcore/health'
ssh hp450-lan 'curl -fsS http://127.0.0.1:8080/openapi.json >/dev/null'
ssh hp450-lan 'docker stats --no-stream moneyprinterturbo-webui moneyprinterturbo-api'
ssh hp450-lan 'df -h /data'
```

The API `/ping` handler is not registered in the current router; use
`/openapi.json` for liveness. The Compose healthchecks use Python's standard
library, so they do not depend on `curl` or `wget` being present in the image.

For the initial application check, verify that the WebUI loads the SiliconFlow
provider and the cloned voice URI field. Do not click a real TTS preview in
this deployment phase. Real SiliconFlow testing is a later, user-approved
manual action and may incur provider charges.

## Configuration safety

- `config.toml` is never copied into the image or Git history.
- Keep it at mode `0600` on HP450 and do not print its contents in logs or
  terminal output.
- The application writes configuration through its runtime save path. The
  existing single-file bind mount has an in-place fallback when the kernel
  rejects an atomic rename across the mount boundary.
- Back up the current config before replacing it. Restore backups with an
  atomic `mv`, not a shell redirection.
- Do not record API keys or complete `speech:` URIs in this runbook.

## Upgrade

1. Push the desired deployment commit.
2. Wait for the branch CI matrix to pass.
3. Manually publish the image and verify its full SHA tag is public.
4. Back up `.env` and record the current healthy SHA.
5. Replace only `MPT_IMAGE` in `.env` with the new SHA.
6. Run `docker compose pull` and `docker compose up -d --pull never --remove-orphans`.
7. Wait for both healthchecks, then inspect logs and preserve the old image.

## Rollback

Rollback changes the image only. It does not delete storage or configuration:

```bash
ssh hp450-lan 'set -eu
  cd /data/projects/moneyprinterturbo
  cp -p .env backups/env.bak-$(date -u +%Y%m%dT%H%M%SZ)
  sed -i -E "s#^MPT_IMAGE=.*#MPT_IMAGE=ghcr.io/lightrao/moneyprinterturbo:sha-<previous-full-sha>#" .env
  docker compose --env-file .env -f compose.yml pull
  docker compose --env-file .env -f compose.yml up -d --pull never --remove-orphans'
```

Do not run `down -v`. If the failure is configuration-related, stop this
Compose project, restore a selected `backups/config.toml.bak-*` file with
permissions `0600`, then start the same image again.

## Scope and future work

MoneyPrinterTurbo is initially outside the HP450 service watchdog. After a
stable observation period, evaluate adding container presence, TCP checks, and
HTTP health checks to the watchdog as a separate change. Public Cloudflare
exposure is also a separate change requiring both tunnel ingress and Access
policy review.

## Live operation baseline (2026-08-08, end-to-end verified)

This fork is in its first stable LAN-only observation window. The items below
were observed directly on HP450 and the Mac during the same calendar day; they
are listed here so future operators do not have to rediscover them.

- Production image reference (loopback-only):
  `ghcr.io/lightrao/moneyprinterturbo:sha-a3bea03dc4420c39943b4de996cd7b7261074350`
- GHCR package: public; no GitHub token is stored on HP450.
- Containers: `moneyprinterturbo-api` and `moneyprinterturbo-webui` are both
  `healthy` from the Compose healthcheck. The API container and the WebUI
  container each expose their inner port on `127.0.0.1`; only the WebUI's
  `8501` is published to the host loopback by the Compose file.
- `/data` available space: approximately `856 GiB`; storage directory
  `/data/projects/moneyprinterturbo/storage` was `189 MiB` after the first
  successful end-to-end render and `199 MiB` after a second configuration
  save; bulk growth comes from `cache_videos/` and per-task `tasks/<uuid>/`.
- API liveness check: `GET /openapi.json` (OpenAPI `3.1.0`, 12 paths).
  `/ping` is not registered in the current router.
- WebUI liveness check: `GET /_stcore/health` returns `ok`.
- Mac SSH forward that is the only production entry point in this phase:
  `127.0.0.1:18501 -> HP450 127.0.0.1:8501` and
  `127.0.0.1:18080 -> HP450 127.0.0.1:8080`. Both ports stay unreachable
  through the host LAN address (`192.168.10.109`).
- API SSH forward also serves static media at
  `http://127.0.0.1:18080/tasks/<task_id>/<file>` with HTTP `206` for
  Range requests, so a Mac browser can stream `final-1.mp4` directly without
  any container-side launcher.
- Pexels reachability from HP450 and from inside the WebUI container was
  verified with `--noproxy "*"` and without any `HTTP_PROXY` /
  `HTTPS_PROXY` / `ALL_PROXY` environment variables; `api.pexels.com`
  answered with `401` (expected without a key) and the TLS handshake
  succeeded. The fork can use Pexels without the host proxy.
- The Mac `gh` token used in this session had `write:packages` scope to
  publish and inspect GHCR images. Rotate or revoke it on the GitHub
  settings page if the workstation is shared.

## Known WebUI limitations in this deployment shape

The WebUI is designed for a same-machine desktop session, but HP450 exposes
the WebUI only through the SSH tunnel. Two control buttons in the task
manager therefore do not work as their label suggests, even though the
generated video itself is fully playable:

- "Play" / "Open Video" in the task manager calls `xdg-open` (or `open`
  / `os.startfile`) inside the WebUI container. The slim image does not
  include `xdg-utils`, and there is no display in the container, so the
  call raises `FileNotFoundError: 'xdg-open'` and is logged in
  `webui/Main.py`. The user sees no visible effect.
- "Open Task Folder" calls `webbrowser.open("file://...")` against the
  server's filesystem path. The container has no browser, so the call
  silently no-ops. The path is also not reachable from the Mac's file
  manager over the SSH forward.

The video file itself is intact and browser-playable:

- Path on HP450:
  `/data/projects/moneyprinterturbo/storage/tasks/<task-id>/final-1.mp4`
- File system type: bind mount from `/data/projects/moneyprinterturbo`
  (`0770` storage root, container `root`, mode `0644` for generated files).
- Container `ffprobe` report for a 49.1 s render: `H.264 High` video at
  `1080x1920`, `yuv420p`; `AAC-LC` audio; `mov,mp4,m4a,3gp,3g2,mj2`
  container. `ffmpeg` decode-to-null check: pass.
- Mac browser playback works directly through the API forward
  (`http://127.0.0.1:18080/tasks/<task-id>/final-1.mp4`).
- `scp` from HP450 is the supported way to copy the artifact to the Mac
  for local playback or upload.

These are design limitations, not blockers. The user has indicated they do
not need in-browser playback or in-page downloads in this phase, so no code
change is required to proceed. If browser-side playback is added later, it
must live in `webui/Main.py` and respect the existing SSH tunnel topology
without exposing loopback ports.

## Recording discipline for this fork

The fork's CI matrix runs on every push to
`feat/siliconflow-cloned-voice-v2` (Python 3.11, Python 3.13, Windows
smoke). Publishes use a reviewed `v1.3.3-hp450.<date>` tag plus the
auto-generated `sha-<commit>` tag. Never:

- record API keys or full `speech:` URIs in commits, comments, or this
  runbook;
- copy `config.toml` into the image, the Compose file, or the repository;
- print the contents of `config.toml` in terminal output or CI logs;
- run a global Docker prune or restart to "fix" MPT problems.

Always:

- use the full SHA tag in `.env` and never overwrite it with a
  semver-only marker;
- back up `.env` and `config.toml` before any planned change;
- run `docker compose --env-file .env -f compose.yml config` to validate
  the rendered model before `up -d`;
- keep WebUI/API on `127.0.0.1` and use the documented SSH forward for
  any user-side testing.

## Private material catalog integration

`MPTMaterialCatalog` is a separate internal Compose project. It is not a
replacement for the existing MPT storage directory and must be staged before
MPT is switched to the new source.

| Item | Value |
| --- | --- |
| Catalog project | `/data/projects/mpt-material-catalog` |
| Catalog API | `127.0.0.1:8085` on HP450 |
| Admin tunnel | Mac `127.0.0.1:18085 -> HP450 127.0.0.1:8085` |
| Internal Docker network | `mpt-catalog-net` |
| Catalog videos | `/data/projects/mpt-material-catalog/videos` |
| MPT read-only mount | `/MaterialCatalog/videos:ro` |
| MPT source | `private_catalog` |

The catalog API and worker use an independent SHA-pinned GHCR image and
secrets. MPT receives only the catalog URL/read key and a read-only bind mount;
it does not receive the Pexels import key. Search misses stay local and never
fall back to `api.pexels.com`.

Create the external network and start catalog first. Only after `/healthz`,
`/readyz`, the admin capacity view, and the 50 asset validation batch pass
should the MPT Compose project be restarted with its catalog network and
read-only volume. Keep the existing MPT ports on loopback; do not add a
Cloudflare route or watchdog entry.

The MPT `[material_catalog]` section reads:

```toml
[material_catalog]
mode = "local"
base_url = "http://mpt-material-api:8080"
api_key = "replace-with-catalog-read-key"
share_volume = "/MaterialCatalog/videos"
```

The `private_catalog` source appears in the WebUI source dropdown only when
`mode`, `base_url`, and `share_volume` are all configured. Tasks that select
it must generate videos without touching the official Pexels API; an MPT
log line such as `searching videos on pexels` should not appear in those
runs.

To roll back, restore the previous MPT image/config, select `pexels` again,
remove the catalog config/network/volume from the MPT Compose project, and
leave the catalog data directory intact. Never use `down -v` or Docker global
prune.

## 2026-08-09 catalog staging snapshot (3-asset pilot accepted)

On 2026-08-09, the MPT side and Catalog release were validated locally and the
Catalog empty deployment plus a 3-asset HP450 pilot were accepted. The
following checks passed:

- `uv run pytest -q` reports `544 passed, 11 skipped, 4153 subtests passed`
  including new `TestPrivateMaterialCatalog` cases for the Pexels-shaped
  search response, Catalog read-key scoping, the read-only shared-volume path,
  the catalog path traversal fallback, and a `private_catalog` cache
  round-trip.
- `uv run ruff check app cli.py webui docs/skill test` is clean.
- `uv run python -m compileall -q app cli.py webui docs/skill test` is clean.
- `docker compose --env-file .env -f deploy/hp450/compose.yml config`
  resolves to two services plus the external `mpt-catalog-net` network
  with the read-only bind mount under `${MPT_MATERIAL_VIDEO_DIR}`.

The Catalog repository is private at `lightrao/MPTMaterialCatalog`, release
`v0.1.2`, with the `linux/amd64` manifest digest
`sha256:33162f2e1d956c3745fb89d5b0db17c00ecb74ddb55b3d83d97dc6bd91e24ae7`.
GitHub Actions built and published it; the package is currently private
pending the manual package visibility change. HP450 Catalog containers are
healthy on the external network and the production Catalog holds exactly 3
pilot assets; Pexels Key runtime cleanup passed. MPT was not restarted or
joined to the Catalog network, and its source remains `pexels`.
