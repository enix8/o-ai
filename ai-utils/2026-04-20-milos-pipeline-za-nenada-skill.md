---
name: slack-consulting-draft
description: Use when the user pastes a Slack message from a colleague asking for help on an infra/devops task and wants you to draft a reply they can copy-paste. The user is consultant, not implementer — they forward your drafts into Slack.
when_to_use: Colleague-facing async help where (1) the user forwards Slack messages verbatim, (2) the deliverable is a Slack reply, not code, and (3) the target codebase needs to be understood before answering. Typical giveaway phrases: "lik mi je poslo", "napisi odg", "mogu copy paste", "pisi u prvom licu njemu".
---

# Slack Consulting Drafts

When a user acts as a middleman between you and a third-party implementer over Slack, your job is to produce **copy-pasteable replies**, not to do the implementation yourself.

## Triggering signals

- User pastes or paraphrases a Slack message from a named third party ("milos salje…", "nenad pita…").
- User asks for a reply in first person: "pisi njemu", "u drugoj osobi".
- User corrects formatting toward Slack conventions ("nije slack friendly", "napravi za copy paste").

## Procedure

1. **Ingest context first, answer second.** Before drafting, look at whatever the user points at (their working directory, ai-utils skills/docs, the target repo). Do not answer from assumptions — the first iteration will almost always pivot after you read local context.

2. **Re-fetch git state before each reply if the colleague pushed.** When the colleague mentions they pushed code or shared a link, `git fetch` and re-read the file *before* drafting. Working from a stale clone leads to wrong advice. If you realize later your response was based on stale state, say so explicitly.

3. **Write each draft into its own file** under `tasks/<client>/active/<task-id>-slack-reply-N.txt` (incrementing `N`). Use `.txt`, not `.md`, because rendered markdown strips the mrkdwn characters when copied. Keep each file single-purpose: one question from the colleague → one reply file.

4. **Format for Slack mrkdwn**, not CommonMark:
   - `*bold*` (single asterisk), never `**bold**`.
   - `_italic_`, never `*italic*`.
   - No `#`/`##`/`###` headings — Slack ignores them. Use `*Section:*` instead.
   - Triple-backtick code blocks work. Inline backticks render inline code — avoid wrapping repo names, env var names, and artifact ids in backticks unless you actually want them monospaced (Miloš flagged this).
   - Bullet: `•` character (not `-` or `*`).
   - Numbered lists don't auto-format — use bullets or plain prose.

5. **Match the colleague's language register.** If they write in Serbian (ekavica), reply in Croatian (ijekavica) only if the user explicitly asks; otherwise mirror the colleague.

6. **Length is dictated by the question.** A "how do I trigger this manually" question gets 3–5 lines; an architectural pitch gets the full structure (problem → approach → example output → prerequisites → implementation plan). Do not default to long.

7. **When the colleague pushes code, read it before answering.** Clone or `git fetch` the branch they mention, read the action/workflow, and cite line-level issues (hardcoded values, branch-matching bugs, missing inputs) in the reply.

8. **Do not invent.** If the user says "imamo `GIT_PAT`", trust it. Don't re-derive or propose creating new secrets when existing ones cover the case. If you're not sure whether a secret/webhook/app exists, say so and ask rather than speculating.

9. **For transient API errors, suggest retry before deep debugging.** GitHub Code Search, GitHub Search, and similar endpoints flake regularly. If the colleague reports HTTP 404/5xx on first run, the first reply should be "try re-run first" — only dive into scopes/permissions/query malformation if the retry also fails. Over-diagnosing a transient flake burns credibility.

10. **Accept being wrong.** The user will push back ("jesi sig?", "koja je bila prva iteracija") — treat that as a prompt to re-check, not to defend. If you change your mind after reading new context, state that explicitly ("prva iteracija bila je X, nakon što sam pročitao Y sam pivotirao na Z"). If the user says "krivo si reko", accept the correction and ask what the real cause was so you can update skill/context.

## Anti-patterns

- Writing the Slack draft into a `.md` file. Rendered markdown loses Slack mrkdwn.
- Using `**bold**` (CommonMark) — Slack shows literal asterisks.
- Duplicating the full architecture pitch in every reply. Short follow-up questions deserve short replies.
- Narrating "let me check" in the Slack draft itself — the draft is the reply, not your process.
- Padding the reply with caveats the colleague didn't ask for.
- Deep-diagnosing a transient API error as a config problem. Retry first, debug second.
- Assuming the colleague didn't push since last reply. Re-fetch.

## File conventions in Ante's setup

- Drafts live at `~/w/o/ai-utils/tasks/<client>/active/<task-id>-slack-reply-N.txt`.
- A first-pass "pitch" file uses a descriptive suffix (`-slack-message.txt`, not `-slack-reply-1.txt`).
- A separate Markdown task doc (`<task-id>-<short-name>.md`) can capture the plan when asked, but is distinct from the Slack drafts.
- Reference dumps of reusable technical knowledge (not per-task Slack drafts) may live outside tasks/ — e.g., `~/w/k51/AI-stuff/<topic>.md`.
