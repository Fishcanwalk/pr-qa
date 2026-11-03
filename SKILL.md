---
name: pr-qa
description: Hands-on QA of a PR on the local stack — a tester subagent checks out the PR, runs CRUD and UI flows for real, screenshots each step, and checks the linked issue's acceptance criteria; passing criteria get ticked on the issue. Use when asked to QA/test a PR, check it against its issue's AC, or capture UI screenshots of a PR.
---

# PR QA

You coordinate; a cheap, fast `tester` subagent does the hands-on run. Output is evidence: PASS/FAIL per check, data snippets, screenshots. Fixing code is out of scope — report it.

Pairs with a code-review skill: when both are asked, run the diff review while the tester runs, then fold QA results into the review artifact.

Requires: `gh` CLI authenticated for the repo, `git`, and a way to run the app locally (see step 1).

## 1. Gather the target

```bash
gh pr view <N> --json title,body,headRefName,headRefOid,files,commits
gh issue view <ISSUE> --json title,body   # issue from "Closes #…" / "Fixes #…" in the PR body
```

Also learn how this project runs locally — read `CLAUDE.md`, `AGENTS.md`, `README`, `CONTRIBUTING`, `package.json` scripts / `Makefile` / `docker-compose*.yml`. Capture:

- install, dev-server, seed/migrate, and test commands, plus the dev port
- backing services the stack needs (DB, queue, …) and how to check they are up
- test users / fixtures / seed data the project provides for QA (never real credentials)
- how tests are organised (unit vs e2e) and where e2e helpers live
- project-specific gotchas (where data lives, UI quirks, flows that need special setup)

Done when you have: head SHA, the linked issue's **Acceptance criteria** checklist, the CRUD surfaces and UI routes the diff touches, any stacked PR it depends on (PR body says "depends on", "merge #… first"), and the run recipe above.

## 2. Dispatch the tester

`Agent` with `subagent_type: "tester"` (or the cheapest capable agent available), in the background. The brief must carry everything below — the tester knows nothing else.

**Setup**
- Worktree: reuse `.claude/worktrees/pr-<N>-qa` if it exists, else `git fetch origin pull/<N>/head` then `git worktree add --detach .claude/worktrees/pr-<N>-qa <head-sha>`. Copy untracked env files (`.env*`) from the main checkout; install dependencies when missing, using the project's package manager and lockfile.
- Infra: check backing services are reachable with the project's own health check; otherwise start them the documented way and wait. Leave them running afterwards.
- Seed what the PR needs; run seeds twice when idempotency is part of the change.
- Start the dev server from the worktree in the background (alternate port if the default is taken); stop it at the end.

**Checks**
- Targeted unit tests on the touched directories only.
- E2E specs the PR touches or that cover its routes. For each failure, run the same spec on the base branch and report whether it fails **the same way** — "also fails on base" only counts when the failing step and cause match.
- Manual flows via a standalone browser-automation script (Playwright if the project uses it) in `<scratchpad>/pr<N>/`. Log in with the project's test users / helpers only. Per CRUD surface: list → create → update → delete/deactivate, verifying the stored record after each write. Plus every UI route the diff changes, for each role that sees it (e.g. admin editable vs read-only).
- One full-page PNG per step in `<scratchpad>/pr<N>/shots/`, numbered and named for the step (`01-admin-list.png`, `02-create-modal.png`, …).
- Map each acceptance criterion to PASS/FAIL with evidence.

**Cleanup**: delete test users and records the run created; leave the data as seeded.

**Report**: PASS/FAIL per check with command tails and record JSON; AC mapping; absolute screenshot paths; bugs/UX oddities.

Include the project gotchas from step 1 in the brief verbatim.

## 3. Verify, then tick

Read the report as a claim: open 2–3 screenshots yourself and confirm they show what the report says.

For each acceptance criterion with PASS evidence that is still `- [ ]` on the issue, tick it — the user's QA request is the approval:

```bash
gh issue view <ISSUE> --json body --jq .body > <scratchpad>/issue-<ISSUE>.md
# flip only the passing AC lines: "- [ ] <text>" → "- [x] <text>"
gh issue edit <ISSUE> --body-file <scratchpad>/issue-<ISSUE>.md
```

Tick only lines under the **Acceptance criteria** heading; other checklists (decisions to make first, definition-of-done, etc.) stay as the owner left them. A FAIL or unverified criterion stays unticked and goes in the report with why. If the PR links no issue or the issue has no AC, skip ticking and report against the PR description instead.

## 4. Report to the user

- Send the step screenshots (skip debug PNGs) so the user can see them, with a caption mapping numbers to steps.
- Chat: a table of check → result, the AC verdict per criterion (ticked / already ticked / left open), test failures with the base-branch comparison, and the state left behind (worktree head, services running).
- When a code-review artifact exists for this PR, put the QA summary into its body.

Done when every AC has a verdict, passing ones are ticked on the issue, and the user has the screenshots.
