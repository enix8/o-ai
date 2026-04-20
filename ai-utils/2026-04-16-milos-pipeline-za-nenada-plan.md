# Plan: Miloš pipeline za Nenada (KEP-4880)

1. **Objective** — Help colleague Miloš design, implement, and debug a GitHub Action that scans all BE repos in `kepler51` org for outdated `common-util-mem` library versions and reports to Slack (Jira task KEP-4880). Deliverables are Slack-ready drafts since Miloš is implementing.

2. **Steps taken**
   - Read colleague's Slack message and Jira screenshot; inspected `k51-common-util-mem` build.gradle + existing main.yaml workflow.
   - Drafted initial approach: inline bash script in the workflow doing repo-by-repo scan.
   - After user prompt, ingested `~/w/o/ai-utils` context (`docs/pipeline/kepler/deployment-strategies.md`, `skills/kepler/deploy/onboard-spring-service.md`, `docs/infra/ecr-repositories.md`).
   - Revised approach to a **composite action** in the `kepler51/gh-actions` repo (reusable across 7 shared libraries).
   - Replaced per-repo scan of 93 repos with GitHub Code Search API (2 calls) + optional per-repo develop-branch fetches.
   - Produced Slack-friendly draft in `tasks/kepler51/active/kep-4880-slack-message.txt` (after iteration to fix mrkdwn formatting).
   - Answered follow-up questions via short reply drafts: `GIT_PAT`, webhook setup, Slack channel (`#dependency-reports`), manual test workflow, rate-limit mitigation (`sleep` between calls).
   - Reviewed Miloš's pushed code on `check-dependents` branch; identified bugs (action hardcodes `"main"` but callers pass `"master"`; develop/main are separate single-branch runs).
   - Dumped full technical context to `/Users/ante/w/k51/AI-stuff/gh-actions-composite-action.md` for future sessions.

3. **Deviations** — Pivot from inline bash to reusable composite action after reading ai-utils context (more repos than first assumed; `gh-actions` repo already established for reusables). Added Code Search API after noting naive scan of 93 repos would be wasteful. Added `sleep` calls only after Miloš hit a rate limit in his first test run.

4. **Outcome** — Consultative task, not code-writing: delivered 10 Slack reply drafts in `tasks/kepler51/active/` plus a reusable skill dump in `~/w/k51/AI-stuff/`. Miloš has a working composite action on `check-dependents` branch with a known integration-side bug (BRANCH matching) still to address. Integration into `main.yaml` is pending Miloš's next step.
