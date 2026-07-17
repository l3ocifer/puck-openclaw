# Local patches vs upstream openclaw/openclaw

Track non-additive changes (anything outside `homelab/`).

## Active patches

### a2a-gateway-ingress: native A2A JSON-RPC ingress on the gateway

- **Files**: `src/gateway/server-http.ts`, `src/gateway/server-runtime-state.ts`, `src/gateway/server.impl.ts`
- **Reason**: exposes `/.well-known/agent-card.json`, `/a2a` and
  `/a2a/v1/*` on the gateway HTTP server and threads an optional
  `handleA2aRequest` provider through runtime state, so the agent-bus
  and cross-agent handoffs can reach Puck natively without a sidecar.
- **Upstream PR**: not submitted (homelab-specific substrate).
- **Last applied**: 2026-07-17 against upstream@db3213264a6
  (`merge: upstream openclaw @ db3213264a6 + reapply a2a-gateway-ingress`).

Puck's own work — finished pieces, drafts, methodology notes — lives
in `puck-graph/pages/`, not in source code.
