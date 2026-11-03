---
name: pr-qa
description: Hands-on QA of a PR on the local stack — a Haiku tester checks out the PR, runs CRUD and UI flows for real, screenshots each step, and checks the linked issue's acceptance criteria; passing criteria get ticked on the issue. Use when asked to QA/test a PR, check it against its issue's AC, or capture UI screenshots of a PR.
---

# PR QA (Smart Shelter)

You coordinate; a Haiku `tester` subagent does the hands-on run. Output is evidence: PASS/FAIL per check, CouchDB snippets, screenshots. Fixing code is out of scope — report it.

Pairs with `pr-code-review`: when both are asked, run the diff review while the tester runs, then fold QA results into the review artifact.

## 1. Gather the target

```bash
gh pr view <N> --json title,body,headRefName,headRefOid,files,commits
gh issue view <ISSUE> --json title,body   # issue from "Closes #…" in the PR body
```

Done when you have: head SHA, the linked issue's **Acceptance criteria** checklist, the CRUD surfaces and UI routes the diff touches, and any stacked PR it depends on (PR body says "ต่อจาก" / "merge #… ก่อน").

## 2. Dispatch the tester

`Agent` with `subagent_type: "tester"`, `model: "haiku"`, in the background. The brief must carry everything below — the tester knows nothing else.

**Setup**
- Worktree: reuse `.claude/worktrees/pr-<N>-qa` if it exists, else `git worktree add --detach .claude/worktrees/pr-<N>-qa <head-sha>` after `git fetch origin pull/<N>/head`. Copy `frontend/.env` from the main checkout; `pnpm install --frozen-lockfile` when `node_modules` is missing.
- Infra: `curl -s localhost:5984/` must return JSON; otherwise `docker compose up -d` from the repo root and wait. Leave containers running afterwards.
- Seed what the PR needs (`pnpm seed:master` etc.); run seeds twice when idempotency is part of the change.
- `pnpm dev` from the worktree in the background (5173, or `--port 5183` if taken); stop it at the end.

**Checks**
- Targeted vitest on the touched directories only.
- Playwright specs the PR touches or that cover its routes. For each failure, run the same spec on `develop` and report whether it fails **the same way** — "also fails on develop" only counts when the failing step and cause match.
- Manual flows via a standalone Playwright script in `<scratchpad>/pr<N>/`. Log in with test users from `e2e/helpers/couch.ts` (`SA_ROLES`, shelter roles) + `e2e/helpers/login.ts` `injectSession` — test users only. Per CRUD surface: list → create → update → delete/deactivate, verifying the CouchDB doc after each write (admin URL from `frontend/.env`). Plus every UI route the diff changes, for each role that sees it (e.g. SA editable vs back-office read-only).
- One full-page PNG per step in `<scratchpad>/pr<N>/shots/`, numbered and named for the step (`01-sa-list.png`, `02-create-modal.png`, …).
- Map each acceptance criterion to PASS/FAIL with evidence.

**Cleanup**: delete created test users, shelters, evacuees and test master items; leave the DB as seeded.

**Report**: PASS/FAIL per check with command tails and doc JSON; AC mapping; absolute screenshot paths; bugs/UX oddities.

Known gotchas to include in the brief:
- Master data lives in DB `registry`, doc `master_data:<type>` (no separate master DB).
- Health data lives in `medical:<evacuee_id>` (shelter DB), queried with `_find` on `type=medical` — not on the evacuee doc.
- Station 2 flow setup: copy from `e2e/helpers/onsite.ts` (`screenEvacuee`) and `e2e/onsite-stations-flow.test.ts`.
- Master items have no delete in the UI; "ปิดใช้" (deactivate) is the D of CRUD.

## 3. Verify, then tick

Read the report as a claim: open 2–3 screenshots yourself and confirm they show what the report says.

For each acceptance criterion with PASS evidence that is still `- [ ]` on the issue, tick it — the user's QA request is the approval:

```bash
gh issue view <ISSUE> --json body --jq .body > <scratchpad>/issue-<ISSUE>.md
# flip only the passing AC lines: "- [ ] <text>" → "- [x] <text>"
gh issue edit <ISSUE> --body-file <scratchpad>/issue-<ISSUE>.md
```

Tick only lines under **Acceptance criteria**; decision checklists ("ต้องตัดสินก่อนเริ่ม" etc.) stay as the owner left them. A FAIL or unverified criterion stays unticked and goes in the report with why.

## 4. Report to the user

- `SendUserFile` with the step screenshots (skip debug PNGs), `display: "render"`, a caption mapping numbers to steps.
- Chat: a table of check → result, the AC verdict per criterion (ticked / already ticked / left open), test failures with the develop comparison, and the state left behind (worktree head, containers running).
- When a `pr-code-review` artifact exists for this PR, put the QA summary into its body.

Done when every AC has a verdict, passing ones are ticked on the issue, and the user has the screenshots.
