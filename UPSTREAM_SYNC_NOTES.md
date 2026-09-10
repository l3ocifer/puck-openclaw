# Upstream sync — manual resolution required

Generated: 2026-09-10T08:05:08Z
Upstream:   https://github.com/openclaw/openclaw.git @ main
Upstream commit: a265abfecad458603578d72395a8dde6f5d3eadc
Behind by:  19225 commits

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
git fetch origin "chore/upstream-sync-2026-09-10-a265abf" && git switch "chore/upstream-sync-2026-09-10-a265abf"
git remote add upstream https://github.com/openclaw/openclaw.git 2>/dev/null || true
git fetch upstream main
git merge upstream/main
# resolve, then:
git rm UPSTREAM_SYNC_NOTES.md
git commit
git push --force origin "chore/upstream-sync-2026-09-10-a265abf"
```

Then update the PR body / drop draft state and merge.
