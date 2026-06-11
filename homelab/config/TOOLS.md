# Puck — tools and environment

## Runtime

- **Framework**: OpenClaw (openclaw/openclaw fork at `l3ocifer/puck-openclaw`)
- **Image**: `ghcr.io/l3ocifer/puck-openclaw:latest`
- **Namespace**: `agents-shared`
- **Schedule**: floats. Soft-prefer thebeast (RAM-heavy — long
  manuscripts in context, full video stems in memory).
- **State PVC**: `puck-state` (longhorn-single, **50 GiB** —
  manuscript drafts, audio renders, video stems, image variants;
  cold artifacts archived weekly to MinIO)
- **Graph PVCs mounted**:
  - `puck-graph` RW — own portfolio, in-progress drafts, canon
    notes, methodology, locked pieces
  - `leo-graph` (restricted-write paths only —
    `pages/world/creative-projects.md`,
    `pages/agent-contributions/puck/`)
  - All six siblings' graphs RO — for cross-agent context

## Models

Puck calls models via LiteLLM (`http://litellm.litellm.svc:4000/v1`):

| Alias        | Use                                                            |
| ------------ | -------------------------------------------------------------- |
| `chat`       | default conversational + drafting                              |
| `code`       | scripting/automation in skills                                 |
| `long`       | book-length context (full manuscript review)                   |
| `frontier`   | careful prose work — limited budget                            |
| `embed`      | embeddings for portfolio search                                |
| `tts`        | text-to-speech (LiteLLM-routed; coqui or openai-tts behind it) |
| `transcribe` | whisper for audio/video transcripts                            |
| `image`      | image gen (sdxl-vllm if unparked, else openai/dalle)           |

Puck is the most model-diverse agent. Different work needs different
tools.

## Communication channels

| Channel                       | Use                                                                     |
| ----------------------------- | ----------------------------------------------------------------------- |
| Matrix `@puck:leopaska.xyz`   | morning briefing, drafts to Leo, peer messages                          |
| Telegram bot (shared)         | quick "look at this" sends to Leo's phone (suppressed in quiet hours)   |
| ntfy `ntfy.leopaska.xyz/puck` | finished-piece notifications                                            |
| A2A — peer to all 6 siblings  | Frack pulls ad copy, Sancho asks for poems, Vetinari requests metaphors |
| HTTP API `:18789`             | exposes `/portfolio/<id>`, `/draft/<id>`, `/render/<job>`               |

Puck does NOT have iMessage, Stripe, BlueBubbles, or Home Assistant
access. Internal-facing — Frack publishes anything that goes outward.

## Creative tooling (in image)

- `ffmpeg` — video assembly, audio mixing, format conversion
- `imagemagick` — image manipulation, batch processing
- `pandoc` — Markdown ↔ DOCX/EPUB/PDF/LaTeX
- `yt-dlp` — source material research (RO; never publishes)
- `calibre` (binary; install at runtime via `apt`) — ebook packaging
- DejaVu / Liberation / Noto fonts — wide font coverage

## Postgres

| Database                                                                                                   | Access                | Purpose                                           |
| ---------------------------------------------------------------------------------------------------------- | --------------------- | ------------------------------------------------- |
| `openclaw_puck` (owner: `puck`)                                                                            | RW                    | own session DB, portfolio metadata, draft history |
| `ironclaw_frick`, `openclaw_frack`, `hermes_sancho`, `openfang_vetinari`, `hermes_quirm`, `ironclaw_vimes` | RO via `puck_ro` role | cross-agent context                               |

## Kubernetes access

ServiceAccount `puck-ops` in `agents-shared`. Cluster-wide
**read-only** — Puck observes the cluster but never modifies it.

## Workspace layout (in `puck-state`)

```
/workspace/
├── media/
│   ├── source/                      ← RO Leo-supplied source material
│   ├── output/                      ← RW renders, drafts
│   └── archive-staging/             ← weekly archive batch → MinIO
├── manuscripts/
│   └── <project-id>/                ← per-project workspace
│       ├── chapters/
│       ├── notes/
│       └── canon/
└── tracks/
    └── <project-id>/
        ├── stems/
        ├── master/
        └── notes/
```

## Graph layout (in `puck-graph`)

```
puck-graph/
├── pages/
│   ├── portfolio/                   ← finished pieces, canonical index
│   │   └── INDEX.md
│   ├── in-progress/                 ← drafts I'm currently working
│   ├── canon/                       ← multi-session continuity notes
│   │   └── <project-id>.md
│   ├── methodology/                 ← writing/composing/editing notes
│   ├── locked/                      ← Vimes-flagged DO-NOT-PUBLISH
│   └── briefings/                   ← morning briefings to Leo (sent + drafts)
└── journals/                        ← daily activity log
```

## Skills (planned, in `puck-graph/pages/skills/`)

- `draft.sh` — scaffold a new piece (poem/essay/chapter/track/video)
- `revise.sh` — apply LLM-assisted revision pass on a draft
- `render-video.sh` — assemble video from shot list + script + b-roll
- `render-track.sh` — ffmpeg pipeline for audio mastering
- `archive-cold.sh` — weekly batch to MinIO, with provenance log
- `portfolio-index.py` — regenerate `portfolio/INDEX.md` from filenames
- `tts.py` — wrapper for LiteLLM tts (audiobook, narration)
- `transcribe.py` — wrapper for whisper (interview prep, source material)

## OpenClaw config

Configured in `homelab/config/openclaw.json`. Puck's enabled toolsets:

- `read`, `write`, `edit` (full RW within `/workspace/` and `puck-graph/`)
- `exec` (sandboxed — ffmpeg, pandoc, imagemagick, yt-dlp, etc.)
- `web_search`, `browser` (research, source-material discovery)
- `memory` (own + cross-graph search)

Puck does NOT have `kubectl write`, BlueBubbles, Stripe, or Home
Assistant tools.

## Web Search

Use the internal self-hosted search/extract service when native
`web_search` is unavailable or has no provider:

```bash
curl -s "$AGENT_TOOL_SERVICE_URL/search?q=public+domain+poetry+archives&limit=5"
```

Extract page text after picking a result:

```bash
curl -s -X POST "$AGENT_TOOL_SERVICE_URL/extract" \
  -H 'content-type: application/json' \
  -d '{"url":"https://example.com","max_chars":12000}'
```

`SEARXNG_URL` points at the raw no-key SearXNG backend for skills that
call `/search?format=json` directly.

## Required env vars

Provided by `puck-secrets` SealedSecret in `agents-shared`:

| Var                                            | Use                                                                                   |
| ---------------------------------------------- | ------------------------------------------------------------------------------------- |
| `LITELLM_API_KEY`                              | virtual key tagged `agent:puck`                                                       |
| `DATABASE_URL`                                 | `postgres://puck@homelab-pg-rw...`                                                    |
| `PUCK_RO_PASSWORD`                             | psql for sibling DB introspection                                                     |
| `MATRIX_HOMESERVER` + `MATRIX_ACCESS_TOKEN`    | `@puck:leopaska.xyz`                                                                  |
| `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID_PUCK` | shared bot, dedicated chat                                                            |
| `NTFY_TOKEN`                                   | finished-piece notifications                                                          |
| `OFP_SHARED_SECRET`                            | A2A mutual auth                                                                       |
| `BW_CLIENTID` + `BW_CLIENTSECRET`              | Vaultwarden API-key login (own credentials only) for ad-hoc lookups via `bw get item` |
| `MINIO_ACCESS_KEY` + `MINIO_SECRET_KEY`        | archive bucket access                                                                 |
| `HEALTHCHECKS_UUID`                            | per-agent UUID for hc-ping.com heartbeats                                             |

## Source control & GitOps (fleet convention)

- **Forgejo — `https://git.leopaska.xyz` — is the source of truth** for
  every repo: homelab, all agent repos, business apps. Clone/push via
  `origin` (`git@git-ssh.leopaska.xyz` SSH or HTTPS).
- **GitHub (`l3ocifer/*`) is a push-mirror backup only.** Never push,
  open issues, or open PRs on GitHub — mirroring from Forgejo is
  automatic and one-way.
- **All deploys are GitOps via ArgoCD** (`argocd.leopaska.xyz`):
  commit → push to Forgejo `main` (or PR) → CI builds the image →
  ArgoCD (+ Image Updater) rolls it. Never `kubectl apply` desired
  state by hand; self-heal reverts live edits. Manual
  `rollout restart` is fine when config in git already changed.
- **Issue intake:** Forgejo issues/comments are webhooked through
  agent-bus to the routed agent's inbox (`pages/inbox/`) with a
  `task_id: forgejo-<repo>-<n>`. Routing: `agent:<name>` label →
  per-repo route → repo-name prefix → vetinari (triage default).
- **Acting on issues:** use the Forgejo API with `$FORGEJO_TOKEN`
  (in this agent's k8s Secret, scopes `write:issue,write:repository`):

  ```bash
  # comment your result
  curl -s -X POST -H "Authorization: token $FORGEJO_TOKEN" \
    -H 'Content-Type: application/json' -d '{"body":"<result>"}' \
    https://git.leopaska.xyz/api/v1/repos/<owner>/<repo>/issues/<n>/comments
  # close when resolved
  curl -s -X PATCH -H "Authorization: token $FORGEJO_TOKEN" \
    -H 'Content-Type: application/json' -d '{"state":"closed"}' \
    https://git.leopaska.xyz/api/v1/repos/<owner>/<repo>/issues/<n>
  ```

- **File new work as Forgejo issues** (not GitHub, not ad-hoc notes)
  so it routes through the same intake to the right agent.
