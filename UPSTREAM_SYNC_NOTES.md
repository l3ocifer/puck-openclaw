# Upstream sync — manual resolution required

Generated: 2026-09-29T08:04:05Z
Upstream:   https://github.com/openclaw/openclaw.git @ main
Upstream commit: cec7fe73dc4b573f1f35aa326e036a3c2b44b7b9
Behind by:  29889 commits

The automated 3-way merge on top of `origin/main` produced conflicts.
The merge was aborted before any conflict markers were committed, so
this branch currently contains only this notes file on top of
`origin/main` — that is by design.

## Conflicting paths

```
.github/CODEOWNERS
.github/workflows/openclaw-npm-release.yml
extensions/telegram/src/polling-session.test.ts
src/agents/agent-tools.cron-scope.test.ts
src/agents/embedded-agent-subscribe.handlers.tools.media.test.ts
src/gateway/server-http.ts
src/gateway/server-runtime-state-prepare.ts
src/gateway/server-runtime-state.ts
```

## How to resolve

```bash
git fetch origin "chore/upstream-sync-2026-09-29-cec7fe7" && git switch "chore/upstream-sync-2026-09-29-cec7fe7"
git remote add upstream https://github.com/openclaw/openclaw.git 2>/dev/null || true
git fetch upstream main
git merge upstream/main
# resolve, then:
git rm UPSTREAM_SYNC_NOTES.md
git commit
git push --force origin "chore/upstream-sync-2026-09-29-cec7fe7"
```

Then update the PR body / drop draft state and merge.
