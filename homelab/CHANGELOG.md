# Changelog

Puck-OpenClaw releases.

## Unreleased

### Added

- Initial homelab/ overlay scaffolding (mirrors frack-openclaw layout
  — same upstream framework, different role).
- Dockerfile builds OpenClaw from local fork + adds creative
  toolchain: `ffmpeg`, `imagemagick`, `pandoc`, `yt-dlp`, `calibre`,
  fonts (DejaVu, Liberation, Noto). No Stripe/BlueBubbles/etc —
  Puck is internal-only.
- k8s manifests for `agents-shared` namespace + floating
  (soft-prefer thebeast for RAM-heavy creative workloads) +
  Longhorn-backed (longhorn-single state @ **50 GiB** for media
  artifacts, longhorn-rwx graphs, all 7 sibling graphs mounted).
- Cluster ClusterRole `puck-cluster-readonly`: read-only across the
  cluster. No mutating verbs anywhere.
- config/SOUL.md (Robin-Goodfellow persona, creative work, internal-
  only, Vimes-lockable).
- config/TOOLS.md (creative toolchain, RO Postgres + cluster,
  workspace + portfolio layout, methodology).
- config/openclaw.json (LiteLLM-routed with diverse model aliases —
  chat/code/long/frontier/embed/tts/transcribe/image; allowlisted
  exec; A2A peers for all 6 siblings; 2 cron tasks).
- GitHub Actions: build.yml, upstream-sync.yml (Sun 03:30 UTC, last
  in the seven-agent stagger), shared-docs-bump.yml.
- Submodule of l3ocifer/homelab at homelab/shared/.

### New repo

- Created fresh as `l3ocifer/puck-openclaw` (cannot be a GitHub fork
  because `l3ocifer/frack-openclaw` already forks the same upstream;
  instead, we cloned upstream and rewired remotes — see README.md
  "Sync from upstream" for the workflow).
