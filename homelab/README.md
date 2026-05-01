# Puck — OpenClaw creative agent

This is **Leo's fork-equivalent of [openclaw/openclaw](https://github.com/openclaw/openclaw)**,
extended to run as `Puck` — the seventh-of-seven agents in [Leo's
homelab](https://github.com/l3ocifer/homelab), named for **Robin
Goodfellow** of *A Midsummer Night's Dream*. Puck's province:
poems, books, videos, music — the work that doesn't fit cleanly
under "infrastructure" or "business" or "audit".

> Note: GitHub limits forks of a single upstream repo to one per
> account, and `l3ocifer/frack-openclaw` already holds that fork
> relationship. This repo was created as a regular repo, with the
> upstream cloned in and `upstream` configured as an explicit remote.
> Functionally identical for sync purposes.

## Layout

```
puck-openclaw/                       ← repo root
├── (upstream openclaw source)
│   ├── src/, extensions/, ui/, ...
│   ├── package.json, pnpm-lock.yaml
│   └── ...
└── homelab/                          ← everything we add
    ├── Dockerfile                    ← node:24 + creative toolchain
    │                                   (ffmpeg, imagemagick, pandoc,
    │                                   yt-dlp, calibre)
    ├── k8s/                          ← kustomize tree
    ├── config/                       ← SOUL.md, TOOLS.md, openclaw.json
    ├── shared/                       ← submodule → l3ocifer/homelab
    ├── .github/workflows/
    ├── PATCHES.md, CHANGELOG.md, README.md
```

## Persona, in 30 seconds

Quick, playful, slightly mischievous. Drafts in the morning, revises
in the afternoon, ships at night. Internal-only — Frack publishes
anything that goes outward. Vimes can lock anything Puck makes
(privacy, libel, copyright); the lock is final until cleared.

See `config/SOUL.md` for the full persona.

## Required env vars

Provided by `puck-secrets` SealedSecret in `agents-shared` namespace.
See `config/openclaw.json` for the full reference.

## Build locally

```bash
git clone https://github.com/l3ocifer/puck-openclaw
cd puck-openclaw
git remote add upstream https://github.com/openclaw/openclaw.git
docker build -f homelab/Dockerfile \
  -t ghcr.io/l3ocifer/puck-openclaw:dev .
```

## Sync from upstream

```bash
git fetch upstream
git merge upstream/main          # or use the weekly auto-PR from CI
```

## License

- OpenClaw upstream: Apache-2.0 (see `../LICENSE`).
- Homelab additions in `homelab/`: same.
- Persona text in `config/SOUL.md` and creative outputs are Leo
  Paska's IP. Pieces written in Leo's voice are owned by Leo;
  pieces Puck makes for itself live in `puck-graph` and are not
  redistributed without Leo's say-so.
