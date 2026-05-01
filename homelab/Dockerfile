# puck-openclaw/homelab/Dockerfile
#
# Builds the OpenClaw image for **Puck**, Leo's creative agent —
# province: poems, books, videos, music. The build context is the
# repository root (this fork of openclaw/openclaw); upstream TS source
# is at the repo root, plus any patches we've layered on top.
#
# Mirrors upstream's build approach (pnpm + bun) and adds creative-
# work tooling: ffmpeg, ImageMagick, pandoc, calibre.
#
# Build (from repo root):
#   docker build -f homelab/Dockerfile \
#     -t ghcr.io/l3ocifer/puck-openclaw:latest \
#     -t ghcr.io/l3ocifer/puck-openclaw:$(git rev-parse --short HEAD) .

# syntax=docker/dockerfile:1.7

# ─── Stage 1: build ──────────────────────────────────────────────────
FROM node:24-bookworm AS build

RUN curl --retry 5 --retry-all-errors --retry-delay 2 -fsSL https://bun.sh/install | bash
ENV PATH="/root/.bun/bin:${PATH}"
RUN corepack enable

WORKDIR /app

# pnpm peeks at patches/, scripts/, ui/package.json, and openclaw.mjs at
# install time (preinstall/postinstall hooks + patchedDependencies).
# COPY them in BEFORE pnpm install or the install fails with ENOENT.
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml* tsconfig*.json* ./
COPY .npmrc .npmrc
COPY openclaw.mjs ./
COPY ui/package.json ./ui/package.json
COPY patches ./patches
COPY scripts ./scripts

RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    NODE_OPTIONS=--max-old-space-size=2048 \
    pnpm install --frozen-lockfile --prod=false

COPY . .

RUN pnpm build:docker || pnpm build || true
RUN pnpm ui:build || true

# ─── Stage 2: runtime ────────────────────────────────────────────────
FROM node:24-bookworm-slim

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      ca-certificates curl wget tini \
      jq ripgrep git \
      postgresql-client \
      python3 python3-pip \
      gnupg lsb-release \
      # Creative tooling Puck uses for media work
      ffmpeg imagemagick \
      pandoc \
      fonts-dejavu fonts-liberation fonts-noto \
      libsndfile1 \
 && rm -rf /var/lib/apt/lists/*

ARG KUBECTL_VERSION=v1.34.0
RUN curl -fsSL "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" \
      -o /usr/local/bin/kubectl \
 && chmod +x /usr/local/bin/kubectl

# 1Password CLI for own credentials
RUN curl -sSfO https://downloads.1password.com/linux/keys/1password.asc \
 && gpg --dearmor 1password.asc \
 && mv 1password.asc.gpg /usr/share/keyrings/1password-archive-keyring.gpg \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/1password-archive-keyring.gpg] https://downloads.1password.com/linux/debian/$(dpkg --print-architecture) stable main" \
      > /etc/apt/sources.list.d/1password.list \
 && apt-get update \
 && apt-get install -y --no-install-recommends 1password-cli \
 && rm -rf /var/lib/apt/lists/* 1password.asc

# yt-dlp for source-material research (read-only — never publishes)
RUN curl -fsSL https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp \
      -o /usr/local/bin/yt-dlp \
 && chmod +x /usr/local/bin/yt-dlp

# gh CLI (manuscript repos, video repos)
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
      -o /usr/share/keyrings/githubcli-archive-keyring.gpg \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
      > /etc/apt/sources.list.d/github-cli.list \
 && apt-get update \
 && apt-get install -y --no-install-recommends gh \
 && rm -rf /var/lib/apt/lists/*

COPY --from=build /app/dist          /opt/openclaw/dist
COPY --from=build /app/node_modules  /opt/openclaw/node_modules
COPY --from=build /app/package.json  /opt/openclaw/package.json
COPY --from=build /app/openclaw.mjs  /opt/openclaw/openclaw.mjs
COPY --from=build /app/skills        /opt/openclaw/skills
RUN ln -sf /opt/openclaw/openclaw.mjs /usr/local/bin/openclaw \
 && chmod 755 /opt/openclaw/openclaw.mjs

WORKDIR /opt/openclaw
RUN mkdir -p /root/.openclaw/workspace /root/.openclaw/skills /data

EXPOSE 18789

HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD curl -fsS http://localhost:18789/healthz || exit 1

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "/opt/openclaw/openclaw.mjs", "gateway", "--bind", "lan", "--port", "18789", "--allow-unconfigured"]
