# Upstream sync — manual resolution required

Generated: 2026-09-16T08:05:21Z
Upstream:   https://github.com/openclaw/openclaw.git @ main
Upstream commit: 3bfe6c63c19bf4b615c91a0957847c8bf2d93896
Behind by:  22892 commits

The automated 3-way merge on top of `origin/main` produced conflicts.
The merge was aborted before any conflict markers were committed, so
this branch currently contains only this notes file on top of
`origin/main` — that is by design.

## Conflicting paths

```
.github/CODEOWNERS
.github/workflows/openclaw-npm-release.yml
src/agents/agent-tools.cron-scope.test.ts
src/agents/model-ref-shared.test.ts
src/gateway/server-http.ts
src/gateway/server-runtime-state-prepare.ts
src/gateway/server-runtime-state.ts
```

## How to resolve

```bash
git fetch origin "chore/upstream-sync-2026-09-16-3bfe6c6" && git switch "chore/upstream-sync-2026-09-16-3bfe6c6"
git remote add upstream https://github.com/openclaw/openclaw.git 2>/dev/null || true
git fetch upstream main
git merge upstream/main
# resolve, then:
git rm UPSTREAM_SYNC_NOTES.md
git commit
git push --force origin "chore/upstream-sync-2026-09-16-3bfe6c6"
```

Then update the PR body / drop draft state and merge.
