# Local patches vs upstream openclaw/openclaw

Track non-additive changes (anything outside `homelab/`).

## Active patches

### a2a-gateway-ingress: native A2A JSON-RPC ingress on the gateway

- **Files**: `src/gateway/server-a2a-ingress.ts` (additive — holds the whole
  implementation), plus three wiring points:
  `src/gateway/server-http.ts` (`isA2aPath` + the `a2a` stage),
  `src/gateway/server-runtime-state.ts` (optional `handleA2aRequest` param),
  `src/gateway/server-runtime-state-prepare.ts` (one call passing the handler
  into `createGatewayRuntimeState`).
- **Reason**: exposes `/.well-known/agent-card.json`, `/a2a` and
  `/a2a/v1/*` on the gateway HTTP server and threads an optional
  `handleA2aRequest` provider through runtime state, so the agent-bus
  and cross-agent handoffs can reach Puck natively without a sidecar.
- **Upstream PR**: not submitted (homelab-specific substrate).
- **Last applied**: 2026-07-26 against upstream@dfcd3b9e0b0.
- **2026-07-26 restructure**: upstream split `server.impl.ts` into
  `server-public.ts` + `server-start.ts` + `server-startup-*.ts`, leaving a
  thin re-export facade. The implementation used to live inline in
  `server.impl.ts`, so that split conflicted the entire file. It now lives in
  its own additive module and the patch is one wiring line, which should stop
  future upstream refactors of the startup path from conflicting. This also
  guards against the silent-drop failure mode: the handler was lost in an
  earlier sync (last present in cc0065d3b2) while the routing stayed behind,
  so inbound `POST /a2a/v1/message:send` 404'd and missions never reached a
  turn. The handler now resolves the gateway request context per request and
  answers 503 while the gateway is still starting, instead of capturing a
  local that only exists late in startup.

### homelab-config-validator: pre-push schema check for openclaw.json

- **Files**: `homelab/scripts/validate-config.mts` (additive, inside `homelab/`
  — listed here for discoverability, not because it conflicts).
- **Reason**: the gateway validates its config against a _strict_ zod schema and
  exits non-zero on an unknown key, so a key upstream renames or retires becomes
  a CrashLoopBackOff that only shows up after the image builds and rolls. Run it
  after every upstream merge and before pushing.
- **Run**: `./node_modules/.bin/tsx homelab/scripts/validate-config.mts`

## Config schema migrations (homelab/config/openclaw.json)

Not source patches, but the same upstream-drift hazard, and the reason the
validator above exists.

### 2026-07-26 (upstream@dfcd3b9e0b0)

The sync to this upstream crash-looped the gateway 7 times with
`Invalid config … Unrecognized keys`. Three changes were needed:

- `agents.defaults.compaction.reserveTokens` / `reserveTokensFloor` — **removed,
  no replacement.** These were the 2026-06-03 workaround for a projection
  reserve that defaulted to ~20K and starved prompt budget on chat's 24576
  window, wedging long a2a mission threads in a compact→truncate→retry loop.
  Upstream deleted that mechanism entirely (no `DEFAULT_PROJECTION_RESERVE_TOKENS`
  remains in the tree) and both keys are now on the retired-knob list that
  `openclaw doctor --fix` strips. The reserve is computed internally; if
  starvation returns, `compaction.keepRecentTokens` is the modern lever.
- `agents.defaults.memorySearch` → **root `memory.search`.** The owner moved to
  the root (per-agent overrides now live at `agents.entries.<id>.memory.search`;
  `agents.defaults` does not accept it at all). Three sub-blocks are gone with no
  replacement and were dropped rather than translated: `store.driver` (sqlite is
  the only driver), the whole `sync` block (watch / debounce / onSessionStart /
  onSearch are no longer configurable — indexing is driven internally), and
  `query.hybrid` (hybrid retrieval is always on; `query` now takes only
  `maxResults` and `minScore`).
- `agents.list[]` → `agents.entries{}`. The array form still loads via a doctor
  migration, so this one was not fatal; written canonically anyway so the seeded
  ConfigMap does not need a migration pass on every boot.

Separately, `memory.search.provider` was `"remote"` — not fatal, and not even a
schema error, because `provider` is a free-form string. But `"remote"` is not an
embedding-provider _id_, so it resolved to nothing and the gateway logged
"no loaded plugin registered a memory embedding provider that can serve
'remote'. Semantic memory recall will fall back to keyword/FTS-only search" —
embeddings had been silently off. The correct id is `openai-compatible` (the
core adapter, registered unconditionally rather than plugin-gated); it prefers
the explicit `remote` block over any `models.providers` entry, so LiteLLM stays
the embedding endpoint. **The validator cannot catch this class of bug** — grep
the gateway's first 30 log lines for warnings after any sync.

## Upstream sync runbook: persisted state migrations

Config is not the only thing that drifts. The state PVC outlives every image, so
an upstream release that bumps a database schema will refuse to boot:

```
Gateway failed to start: OpenClaw agent database
/root/.openclaw/agents/main/agent/openclaw-agent.sqlite uses schema version 11;
run openclaw doctor --fix to migrate persisted media before using it.
```

There is no scoped migration flag — `openclaw doctor --fix` is the only path,
it takes ~2 minutes, and it rewrites the runtime config copy (stripping JSON5
comments, which is harmless: the ConfigMap in this repo is the source of truth).
It is deliberately NOT wired into an init container, because it would add two
minutes and a config rewrite to every pod restart in exchange for a fault that
only occurs on an upstream schema bump.

Because the crash-looping pod cannot be exec'd into, run it against a paused
copy of the pod, which mounts the same PVC on the same node:

```sh
POD=$(kubectl get pods -n agents-shared -l app.kubernetes.io/name=puck -o name | head -1 | cut -d/ -f2)
kubectl debug -n agents-shared "$POD" --copy-to=puck-migrate --container=openclaw -- sh -c 'sleep 3600'
kubectl exec -n agents-shared puck-migrate -c openclaw -- node /opt/openclaw/openclaw.mjs doctor --fix
kubectl delete pod -n agents-shared puck-migrate
kubectl delete pod -n agents-shared "$POD"   # fresh pod re-seeds and starts clean
```

### State migrations belong in an initContainer, not in your hands

The state PVC outlives the image, and OpenClaw versions what it persists. So an
upstream sync that bumps a state schema leaves the agent running but degraded,
telling you to run `openclaw doctor --fix`. Doing that turns out to be awkward
in exactly the wrong way:

- Run it inside the live pod and doctor declines the session-store work —
  _"Gateway or another SQLite maintenance command owns the state directory"_ —
  after happily applying the harmless parts, so it looks like it worked.
- Stop the gateway first by scaling the Deployment to 0 and ArgoCD's `selfHeal`
  scales it straight back up. Patching `syncPolicy.automated` off does not help
  either: the Application itself is managed by the root app-of-apps, which
  restores the spec within seconds.

The fix is a `doctor-fix` initContainer that runs upstream's migrator before the
gateway starts, when the state directory is quiet. It is declarative, idempotent,
survives selfHeal, and puts the migration log somewhere findable.

It ends in `|| true` deliberately. Doctor mixes migrations with advisory findings
(no command owner configured, missing main transcript) and exits non-zero for
those too; blocking boot on advice would be worse than the problem. A migration
that genuinely fails still surfaces, because the gateway then refuses to start
and the initContainer log names the cause.

Its image is the same `:homelab` tag as the gateway so ArgoCD Image Updater pins
both to one digest — the migrator must never be a different version than the
runtime it is migrating for.

Two migrations have needed this so far: the media/session schema bump (schema
version 11) and the legacy `exec-approvals.json` file.

### The re-seed trap: a ConfigMap change that never reaches the gateway

The seed init container compares the ConfigMap's sha256 against a marker file at
`/root/.openclaw/runtime-config/.configmap-source.sha256`. It never compares the
ConfigMap against the _runtime copy itself_. Normally that is fine and desirable
— it is what lets doctor's rewrite survive a restart. It fails when the gateway
rewrites the runtime copy in the same boot in which it was seeded, which it does
whenever it normalises the config (you can see it happen: it preserves the file
it overwrote as `openclaw.json.clobbered.<timestamp>` and the previous accepted
one as `openclaw.json.last-good`). After that the marker matches the ConfigMap
while the file on disk does not, and _every subsequent restart is a no-op_ — the
init container logs `runtime copy preserved (configmap unchanged: <hash>)` and
the stale copy is loaded forever.

This is how the `memory.search.provider` fix appeared not to apply: the
ConfigMap was correct, ArgoCD reported Synced, the pod was restarted twice, and
the gateway still logged the `"remote"` warning, because it was reading a copy
that had been clobbered back to the pre-fix content.

Clear the marker to force a re-seed. Do this after any ConfigMap change that a
plain restart did not pick up:

```sh
POD=$(kubectl get pods -n agents-shared -l app.kubernetes.io/name=puck -o name | head -1 | cut -d/ -f2)
kubectl exec -n agents-shared "$POD" -c openclaw -- rm -f /root/.openclaw/runtime-config/.configmap-source.sha256
kubectl delete pod -n agents-shared "$POD"
```

Then confirm the value actually landed rather than trusting the restart:

```sh
POD=$(kubectl get pods -n agents-shared -l app.kubernetes.io/name=puck -o name | head -1 | cut -d/ -f2)
kubectl logs -n agents-shared "$POD" -c seed-workspace | grep openclaw.json   # want "seeding", not "preserved"
kubectl exec -n agents-shared "$POD" -c openclaw -- grep -n openai-compatible /root/.openclaw/runtime-config/openclaw.json
```

A freshly seeded copy still has its JSON5 comments; a doctor-rewritten one does
not, which is a quick way to tell which of the two the gateway is running.

Puck's own work — finished pieces, drafts, methodology notes — lives
in `puck-graph/pages/`, not in source code.
