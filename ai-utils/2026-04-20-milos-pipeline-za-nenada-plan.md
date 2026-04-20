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
   - Dumped full technical context to `/Users/ante/w/k51/AI-stuff/gh-actions-composite-action.md`.
   - Reviewed Miloš's `test-dependency-report.yaml` and advised on integration into `main.yaml` (reply #10): separate `dependency-report` job with `needs: buildjob`, artifact-id extraction, branch mapping `master → main` inline.
   - Clarified Nenad's actual requirement (reply #11): develop push → develop report only, master push → main report only (not both on every run); reworked to split Gradle step in buildjob (develop: build only, master: build+publish).
   - Added `workflow_dispatch` support (reply #12): inputs for branch + published-version, conditional buildjob skip (`if: github.event_name != 'workflow_dispatch'`), `dependency-report` with `if: always() && (needs.buildjob.result == 'success' || needs.buildjob.result == 'skipped')`.
   - Diagnosed non-triggering push (reply #13): `paths-ignore: ['.github/**']` excludes commits that only touch the workflow file.
   - Diagnosed `gh workflow run` failure (reply #14): `workflow_dispatch` trigger must exist on default branch (master); advised committing workflow to master with `--ref develop` for testing.
   - Diagnosed HTTP 404 from Code Search API (replies #15, #16): suggested scope/permission checks and `${#GH_TOKEN}` length debug — **turned out to be transient GitHub API flake, fixed by re-run in browser**.

3. **Deviations** —
   - Pivoted from inline bash to reusable composite action after reading ai-utils context.
   - Added Code Search API after noting naive scan of 93 repos would be wasteful.
   - Added `sleep` calls only after Miloš hit a rate limit.
   - Changed from "always report both branches" to "report matches trigger branch" after Miloš clarified Nenad's real ask.
   - Over-engineered 404 diagnosis — real fix was simple retry. Should default to retry first for Code Search errors.

4. **Outcome** — Consultative task: delivered 16 Slack reply drafts + a reusable skill dump. Miloš has working composite action on `check-dependents` branch and integrated it into `main.yaml` on develop. Test workflow run succeeded on retry. `workflow_dispatch` trigger needs to be committed to master branch for manual runs to work. Integration into other shared libraries is future work.
