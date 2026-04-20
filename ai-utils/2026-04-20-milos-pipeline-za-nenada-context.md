# Context: Miloš pipeline za Nenada (KEP-4880)

## Required skills

- `2026-04-20-milos-pipeline-za-nenada-skill.md` (next to this file) — Slack consulting draft technique.
- `/Users/ante/w/k51/AI-stuff/gh-actions-composite-action.md` — full technical dump on composite actions, Code Search API, rate limits, secrets/inputs rules, dependency version extraction regexes, Slack integration patterns, and testing harness for the kepler51 org. **Always read this before answering a kepler51 GH Actions question.**
- `~/w/o/ai-utils/docs/pipeline/kepler/deployment-strategies.md` — enumerates 93 repos, identifies 68 BE services (Strategy A), 7 shared libs without pipelines.
- `~/w/o/ai-utils/skills/kepler/deploy/onboard-spring-service.md` — standard workflow template.
- `~/w/o/ai-utils/docs/infra/ecr-repositories.md` — repo catalog.

## Required MCPs

None used. Optional `slack-workspace2` would enable direct posting; not needed because Ante is the middleman.

## Required tools

- `gh` CLI (authenticated) — for inspecting `kepler51/gh-actions` and `kepler51/k51-common-util-mem` branches.
- `git` — `fetch` and `checkout` on both repos.

## Files touched

Slack reply drafts under `~/w/o/ai-utils/tasks/kepler51/active/`:

- `kep-4880-dependency-report-tool.md` — plan doc (architecture, implementation steps).
- `kep-4880-slack-message.txt` — initial pitch.
- `kep-4880-slack-reply.txt` — branch handling + workflow_dispatch explanation.
- `kep-4880-slack-reply-2.txt` — existing Slack integration example + new channel proposal.
- `kep-4880-slack-reply-3.txt` — webhook setup.
- `kep-4880-slack-reply-4.txt` — what `GIT_PAT` is for.
- `kep-4880-slack-reply-5.txt` — `GIT_PAT` already exists.
- `kep-4880-slack-reply-6.txt` — manual test workflow + `gh workflow run`.
- `kep-4880-slack-reply-7.txt` — pass `github-token` via inputs + env `GH_TOKEN`.
- `kep-4880-slack-reply-8.txt` — composite actions cannot use `secrets.*` directly.
- `kep-4880-slack-reply-9.txt` — rate-limit fix: `sleep 3` between Code Search calls.
- `kep-4880-slack-reply-10.txt` — integration into main.yaml (first version: both branches).
- `kep-4880-slack-reply-11.txt` — clarified Nenad's ask: develop push → develop report, master push → main report. Split Gradle step by branch.
- `kep-4880-slack-reply-12.txt` — `workflow_dispatch` with inputs; buildjob skip; `always() && (success || skipped)` on dependency-report.
- `kep-4880-slack-reply-13.txt` — `paths-ignore: ['.github/**']` blocks workflow-only pushes.
- `kep-4880-slack-reply-14.txt` — `workflow_dispatch` trigger must be on default branch; use `--ref develop`.
- `kep-4880-slack-reply-15.txt` — Code Search 404 diagnosis attempt (scope/permission/org: qualifier) — WRONG path.
- `kep-4880-slack-reply-16.txt` — secrets/debug suggestion — WRONG path; real fix was browser re-run (transient GitHub flake).

Reference dump:
- `/Users/ante/w/k51/AI-stuff/gh-actions-composite-action.md` — reusable technical reference for future sessions.

## External services and credentials

- **GitHub org `kepler51`** — 93 repos. `GIT_PAT` (classic or fine-grained PAT) is an **org-level secret** already in use across all BE workflows. Do not create a new secret.
- **AWS CodeArtifact** — `java-internal` domain (owner 262361587560, us-east-1) stores `com.k51:common-util-mem`. Authenticated via `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`.
- **Slack** — two webhooks in play:
  - `SLACK_WEBHOOK_BACKEND_DEPLOY` (existing) → `#backend-deploy`, used by `ravsamhq/notify-slack-action@v2`.
  - `SLACK_WEBHOOK_LIBRARY_UPDATES` (created during this task) → `#dependency-reports`, added via the existing Slack app at https://api.slack.com/apps → Incoming Webhooks.
- **Jira** — Kepler51 Atlassian, task KEP-4880 ("BE - Github Library reporting tool") under KEP-1342 epic.

## Environment setup

- Repositories inspected locally (always `git fetch` first):
  - `/Users/ante/w/k51/k51-common-util-mem/` (library being published)
  - `/Users/ante/w/k51/gh-actions/` (shared actions; `check-dependents` branch is the WIP)

## Key decisions

- **Composite action over inline bash**: reusable across 7 shared libs (`common-util-mem`, `common-util`, `exception-util`, `audit-publisher-util`, `permissions-util`, `sensors-util`, `assets-util`) + future `hoard-java`.
- **GitHub Code Search API over per-repo scan**: 2 API calls + ~15 per-repo fetches = ~20 calls, vs ~200+ naive.
- **Dedicated Slack channel `#dependency-reports`**: separates from noisy `#backend-deploy`.
- **Action receives `branch` as input, runs one branch per call**: Nenad's requirement is trigger-branch-matches-report, not both every time. Push to develop → develop report; push to master → main report.
- **Gradle step split by branch** in buildjob: develop `clean build -x test` (no publish), master `clean build publish --info -x test`.
- **`workflow_dispatch` inputs** override `push`-computed values: branch choice, published-version string.
- **`if: github.event_name != 'workflow_dispatch'`** on buildjob to skip build+publish on manual invocation.
- **`if: always() && (needs.buildjob.result == 'success' || needs.buildjob.result == 'skipped')`** on dependency-report to run after either successful build or workflow_dispatch-skipped build.
- **Reuse existing `GIT_PAT` secret** — already scoped org-wide.
- **`paths-ignore: ['.github/**']`** kept in place (standard pattern for kepler51 workflows); means workflow-only changes need a companion touch of a non-workflow file to trigger, or use workflow_dispatch.

## Open questions

- **Bug in `check-dependents` action**: it checks `if [[ "$BRANCH" == "main" ]]`, but kepler51 uses `master` as the default. Caller must hardcode `branch: main` or use the mapping `github.ref_name == 'master' && 'main' || 'develop'`. Not yet fixed inside the action.
- **`workflow_dispatch` trigger on master branch**: as of the last reply, only develop branch has the updated workflow. Master needs the workflow with `workflow_dispatch` for manual dispatch to work (GitHub requires the trigger on the default branch).
- **Nested/multi-module Gradle projects** not handled: action only reads root `build.gradle`/`pom.xml`.
- **Develop branch fetch for pom.xml**: action loops over `POM_REPOS` (from Code Search default branch) for develop — if a repo has pom.xml only on develop, it's missed.
- **Reuse for other libraries** (`common-util`, `exception-util`, etc.): same pattern, not yet rolled out.

## Gotchas

- **Composite actions cannot use `${{ secrets.X }}`** inside `action.yaml`. Must receive via `inputs`.
- **Code Search API 404s are often transient.** Retry before diagnosing. A single failed run is not evidence of a config problem. Real debugging (scope, permissions, query formatting) only after multiple retries fail.
- **Code Search API rate limit** is far stricter than REST. `sleep 3` between back-to-back searches.
- **`github.ref_name` on master push returns `"master"`**, not `"main"`. Branch string comparisons must match the actual branch name.
- **`github.repository` returns `kepler51/k51-common-util-mem`**, not the Maven artifact id. Extract from `build.gradle` (`def artifactName = '...'`).
- **`gh` CLI reads `GH_TOKEN` env var automatically** — set in step `env:` block.
- **`paths-ignore: ['.github/**']`** means commits touching only workflow files don't trigger the workflow. Companion-touch a non-workflow file, or use `workflow_dispatch`.
- **`workflow_dispatch` trigger must exist on the default branch** for `gh workflow run` to dispatch — even if you pass `--ref otherbranch`. The execution uses `--ref`, but the dispatch capability is read from default branch.
- **`needs: buildjob` with no `if:` blocks on skipped dependencies.** Explicit `if: always() && (... == success || ... == skipped)` required to let dependency-report run after a skipped buildjob (workflow_dispatch case).
- **`paths-ignore` + workflow-only commit** = no trigger. Classic pitfall when developers push ONLY the workflow change to a new branch expecting it to run.
- **Slack mrkdwn vs CommonMark**: `*bold*` vs `**bold**`. Save drafts as `.txt`, not `.md`.
- **Re-fetch git state** before reading a colleague's repo. Working from stale clones in a long conversation leads to wrong advice.
