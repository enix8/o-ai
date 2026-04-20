# Context: Miloš pipeline za Nenada (KEP-4880)

## Required skills

- `2026-04-16-milos-pipeline-za-nenada-skill.md` (next to this file) — Slack consulting draft technique.
- `/Users/ante/w/k51/AI-stuff/gh-actions-composite-action.md` — full technical dump on composite actions, Code Search API, rate limits, secrets/inputs rules, dependency version extraction regexes, Slack integration patterns, and testing harness for the kepler51 org. **Always read this before answering a kepler51 GH Actions question.**
- `~/w/o/ai-utils/docs/pipeline/kepler/deployment-strategies.md` — enumerates 93 repos, identifies 68 BE services (Strategy A), 7 shared libs without pipelines. Needed to understand which repos are candidate dependents.
- `~/w/o/ai-utils/skills/kepler/deploy/onboard-spring-service.md` — standard workflow template (Slack step, secrets, branch conventions).
- `~/w/o/ai-utils/docs/infra/ecr-repositories.md` — repo catalog.

## Required MCPs

None used in this session. Optional: `slack-workspace2` would enable direct posting; not needed because Ante was the middleman.

## Required tools

- `gh` CLI (authenticated) — for inspecting `kepler51/gh-actions` and `kepler51/k51-common-util-mem` branches.
- `git` — used to `fetch` and `checkout check-dependents` branch in `gh-actions`.
- No build tools needed; this was advisory work.

## Files touched

- `~/w/o/ai-utils/tasks/kepler51/active/kep-4880-dependency-report-tool.md` — plan doc (architecture, implementation steps, Code Search + develop branch strategy).
- `~/w/o/ai-utils/tasks/kepler51/active/kep-4880-slack-message.txt` — initial Slack pitch (`*bold*`, bullets, code block).
- `~/w/o/ai-utils/tasks/kepler51/active/kep-4880-slack-reply.txt` — reply #1 (branch handling + workflow_dispatch explanation).
- `~/w/o/ai-utils/tasks/kepler51/active/kep-4880-slack-reply-2.txt` — reply #2 (existing Slack integration example + new channel proposal).
- `~/w/o/ai-utils/tasks/kepler51/active/kep-4880-slack-reply-3.txt` — reply #3 (webhook setup steps).
- `~/w/o/ai-utils/tasks/kepler51/active/kep-4880-slack-reply-4.txt` — reply #4 (what `GIT_PAT` is for).
- `~/w/o/ai-utils/tasks/kepler51/active/kep-4880-slack-reply-5.txt` — reply #5 (`GIT_PAT` already exists as org secret).
- `~/w/o/ai-utils/tasks/kepler51/active/kep-4880-slack-reply-6.txt` — reply #6 (manual test workflow + `gh workflow run`).
- `~/w/o/ai-utils/tasks/kepler51/active/kep-4880-slack-reply-7.txt` — reply #7 (pass `github-token` via `inputs` + env `GH_TOKEN`).
- `~/w/o/ai-utils/tasks/kepler51/active/kep-4880-slack-reply-8.txt` — reply #8 (composite actions cannot use `secrets.*` directly).
- `~/w/o/ai-utils/tasks/kepler51/active/kep-4880-slack-reply-9.txt` — reply #9 (rate-limit fix: `sleep 3` between Code Search calls, `sleep 1` in loops).
- `~/w/o/ai-utils/tasks/kepler51/active/kep-4880-slack-reply-10.txt` — reply #10 (integration into main.yaml: `needs: buildjob`, artifact-id extraction, calling action twice for develop + main).
- `/Users/ante/w/k51/AI-stuff/gh-actions-composite-action.md` — new reusable skill/reference dump for future sessions.

## External services and credentials

- **GitHub org `kepler51`** — 93 repos. `GIT_PAT` (classic or fine-grained PAT) is an **org-level secret** used across all BE workflows (e.g., for `kepler51/gh-actions@update-argo-app`). Do not create a new secret.
- **AWS CodeArtifact** — `java-internal` domain (owner 262361587560, us-east-1) stores `com.k51:common-util-mem`. CI authenticates via `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` secrets.
- **Slack** — two webhooks in play:
  - `SLACK_WEBHOOK_BACKEND_DEPLOY` (existing) → `#backend-deploy` channel, used by `ravsamhq/notify-slack-action@v2` in all BE workflows.
  - `SLACK_WEBHOOK_LIBRARY_UPDATES` (**to be created**) → `#dependency-reports` channel (Miloš already created the channel). Webhook should be added via the existing Slack app at https://api.slack.com/apps → Incoming Webhooks → "Add New Webhook to Workspace".
- **Jira** — Kepler51 Atlassian, task KEP-4880 ("BE - Github Library reporting tool") under KEP-1342 epic. Board: K51 Scrum Board.

## Environment setup

- No env vars needed on Ante's side. Consulting-only session.
- For Miloš's local testing: `gh auth login` against kepler51 with a PAT that has org-wide `repo` read.
- Repositories inspected locally:
  - `/Users/ante/w/k51/k51-common-util-mem/` (library being published)
  - `/Users/ante/w/k51/gh-actions/` (shared actions; `check-dependents` branch is the WIP)

## Key decisions

- **Composite action over inline bash**: reusable across 7 shared libs (`common-util-mem`, `common-util`, `exception-util`, `audit-publisher-util`, `permissions-util`, `sensors-util`, `assets-util`) and future `hoard-java`. Rejected inlining bash directly into `k51-common-util-mem/.github/workflows/main.yaml` because copy-paste into every library would diverge.
- **GitHub Code Search API over per-repo scan**: 2 API calls (default branch) + ~15 per-repo fetches (develop) = ~20 calls, vs ~200+ calls for naive scan of all 93 repos. Rejected listing all repos and iterating because rate budget would be wasted.
- **Dedicated Slack channel `#dependency-reports`**: separates from noisy `#backend-deploy`. Miloš confirmed creating the channel.
- **Action receives `branch` as input, runs one branch per call**: Miloš chose to make develop and main completely separate code paths inside the action. To satisfy Nenad's "both develop and master" requirement, the calling workflow invokes the action twice (once with `branch: main`, once with `branch: develop`) in the same job. Considered having the action do both internally — rejected to keep the Slack messages separable.
- **Use existing `GIT_PAT` secret**: no new token needed. Already scoped org-wide and already used by `update-argo-app`.
- **`workflow_dispatch`**: added to workflows for manual test triggering via `gh workflow run`. Two lines of YAML.
- **Test via dedicated `test-dependency-report.yaml`**: avoids triggering the real `buildjob` (which publishes to CodeArtifact) during testing. Delete after integration is verified.

## Open questions

- **Bug in `check-dependents` action**: it checks `if [[ "$BRANCH" == "main" ]]`, but `k51-common-util-mem` uses `master` as its default branch. If the calling workflow passes `${{ github.ref_name }}`, the main branch code path never executes. Fix options (not yet applied): (a) caller hardcodes `branch: main`, (b) caller normalizes `master → main`, (c) action accepts both. Miloš has not yet integrated into `main.yaml`, so this is pending.
- **Integration into `k51-common-util-mem/.github/workflows/main.yaml` is not done** — only the test workflow (`test-dependency-report.yaml`) has been added so far (commit 4b3f3594). Reply draft #10 covers the needed changes (new `dependency-report` job with `needs: buildjob`, artifact-id extraction, two `uses: kepler51/gh-actions@check-dependents` steps for develop and main).
- **Nested/multi-module Gradle projects** not handled: action only reads root `build.gradle`/`pom.xml`. If any dependent has the dep only in a submodule, it's invisible. Accepted for v1.
- **Develop branch fetch for pom.xml**: action loops over `POM_REPOS` for develop but Code Search only indexes default branch — if a repo has pom.xml only on develop, it won't be in `POM_REPOS` and won't be scanned.

## Gotchas

- **Composite actions cannot use `${{ secrets.X }}`** inside `action.yaml`. Secrets must be passed as inputs from the calling workflow. Miloš hit this (`Unrecognized named-value: 'secrets'`) on his first push. Fix: define `github-token` and `slack-webhook-url` as `inputs`, reference via `${{ inputs.* }}`, set `GH_TOKEN` env from input.
- **Code Search API rate limit** is far stricter than the REST API. Exceeded on the second search call back-to-back during Miloš's first run. `sleep 3` between Code Search calls is enough.
- **`github.ref_name` on master push returns `"master"`**, not `"main"`. Action string comparisons must match actual branch name.
- **`github.repository` returns `kepler51/k51-common-util-mem`**, which is NOT the Maven artifact id (`common-util-mem`). Extract the artifact id from `build.gradle` (`def artifactName = '...'`) instead.
- **`gh` CLI reads `GH_TOKEN` env var automatically** — no manual `export` needed, just set it in the step's `env:` block.
- **`paths-ignore: ['.github/**']` in the existing workflow** means pushing workflow changes to master does NOT trigger `buildjob`. Useful for iterating on the workflow without republishing the library.
- **The k51-common-util-mem repo's default branch is `master`** (not `main`). Other repos in the org may vary.
- **Slack mrkdwn vs CommonMark**: `*bold*` (Slack) vs `**bold**` (CommonMark). Saving drafts as `.md` files strips formatting on copy. Use `.txt`.
