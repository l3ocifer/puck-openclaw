# Upstream sync — manual resolution required

Generated: 2026-09-26T08:05:06Z
Upstream:   https://github.com/openclaw/openclaw.git @ main
Upstream commit: 5e4b43cf86c76b188631ed420032dc221eadd794
Behind by:  28273 commits

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
git fetch origin "chore/upstream-sync-2026-09-26-5e4b43c" && git switch "chore/upstream-sync-2026-09-26-5e4b43c"
git remote add upstream https://github.com/openclaw/openclaw.git 2>/dev/null || true
git fetch upstream main
git merge upstream/main
# resolve, then:
git rm UPSTREAM_SYNC_NOTES.md
git commit
git push --force origin "chore/upstream-sync-2026-09-26-5e4b43c"
```

Then update the PR body / drop draft state and merge.
