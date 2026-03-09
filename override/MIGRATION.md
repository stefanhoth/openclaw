# Migration: Move to Standalone openclaw-custom Repo

## Why

The fork of `openclaw/openclaw` was created solely to override the `BASE_IMAGE` Docker build arg. This is more complexity than it's worth:

- Syncing with upstream is fragile (rebase conflicts, SHA mismatches on push)
- The official `ghcr.io/openclaw/openclaw-browser:latest` image already exists with Chromium pre-installed — there is no need to rebuild it
- All custom additions (whisper, blogwatcher, mcporter) only need a thin layer on top

## Target: New standalone repo

A new repo (e.g. `openclaw-custom`) with exactly three files:

```
openclaw-custom/
├── Dockerfile
├── docker-compose.yml
└── .github/
    └── workflows/
        └── build.yml
```

### Dockerfile

Key differences from the fork version:
- Full `playwright` package (not `playwright-core`) so the playwright API works correctly
- Proper three-step playwright install: global install as root → `npx playwright install` as node (browser binaries) → `npx playwright install-deps` as root (system deps)
- `OPENCLAW_VERSION`/`OPENCLAW_BUNDLED_VERSION` dropped — only needed when building openclaw itself; the official image already has its version baked in
- UID/GID changes kept for VPS deployment

Layer caching optimisations applied:
- UID/GID changes combined into one `RUN` (stable, rarely busts cache)
- `apt-get` for custom packages runs before npm/pip work (stable)
- Whisper pip install + model download combined into one `RUN` with `--no-cache-dir`
- mcporter + playwright npm installs combined into one `RUN`, cache cleaned in the same layer (no `npm install -g npm` — unnecessary layer)
- `apt-get clean` after `playwright install-deps` (that step pulls additional system packages)
- Order: most-stable layers first → least-stable last, so a version bump of mcporter/playwright doesn't bust the whisper or blogwatcher cache

```dockerfile
# ARG must be declared before first FROM to be usable in FROM instructions
ARG BASE_IMAGE=ghcr.io/openclaw/openclaw-browser:latest

FROM golang:1.26-alpine AS blogwatcher-builder
RUN go install github.com/Hyaxia/blogwatcher/cmd/blogwatcher@latest

FROM ${BASE_IMAGE}

USER root

# UID/GID in one layer (almost never changes)
RUN usermod -u 1500 node \
    && groupmod -g 1500 node \
    && groupadd --gid 2500 obsidian \
    && usermod -aG obsidian node \
    && chown -R 1500:1500 /app \
    && chown -R 1500:1500 /home/node

# System packages (rarely changes)
RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg \
    python3-pip \
    curl \
    git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Whisper — install + pre-download model in one layer, no pip cache
RUN pip3 install openai-whisper --break-system-packages --no-cache-dir \
    && python3 -c "import whisper; whisper.load_model('base')"

# blogwatcher (busts only when the Go binary changes)
COPY --from=blogwatcher-builder /go/bin/blogwatcher /usr/local/bin/blogwatcher

# npm packages in one layer; clean cache immediately to keep layer small
# Full playwright (not playwright-core) — required for playwright API to work correctly
RUN npm install -g mcporter playwright \
    && npm cache clean --force

# Playwright browser binaries as node, system deps as root
USER node
RUN npx playwright install

USER root
RUN npx playwright install-deps \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

USER node
```

### docker-compose.yml

Copy of the current `override/docker-compose.yml` with:
- `image: ghcr.io/stefanhoth/openclaw-custom:latest`
- Remove `group_add` from `openclaw-cli` (doesn't mount the obsidian vault)

### .github/workflows/build.yml

Triggers:
- `push` to `main` — rebuild on any file change
- `schedule: '0 4 * * 1'` — weekly rebuild to pick up base image updates
- `workflow_dispatch` — manual trigger

Key build flag: `--pull` so Docker always fetches the latest base image. No tag-detection or sync logic needed.

OCI labels to include:
- `org.opencontainers.image.base.name: ghcr.io/openclaw/openclaw-browser:latest`
- `org.opencontainers.image.revision`
- `org.opencontainers.image.created`
- `ai.openclaw.base.version` — resolved from the latest stable git tag of openclaw/openclaw at build time

## Migration steps

1. Create new GitHub repo `openclaw-custom`
2. Add the three files above and push
3. Trigger `workflow_dispatch` → verify `ghcr.io/stefanhoth/openclaw-custom:latest` is published
4. Update VPS: `docker compose pull && docker compose up -d openclaw-gateway`
5. Archive the fork `stefanhoth/openclaw` (no longer needed)

## Verification

```bash
# Check base image label
docker inspect ghcr.io/stefanhoth/openclaw-custom:latest \
  --format '{{ index .Config.Labels "org.opencontainers.image.base.name" }}'

# Confirm whisper is available
docker exec openclaw-gateway python3 -c "import whisper; print('ok')"

# Confirm blogwatcher is available
docker exec openclaw-gateway blogwatcher --version

# Confirm mcporter is available
docker exec openclaw-gateway mcporter --version
```
