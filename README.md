# pr-qa

A [Claude Code](https://claude.com/claude-code) skill for hands-on QA of a pull request on your local stack.

A `tester` subagent checks out the PR in a worktree, starts the app, runs CRUD and UI flows for real, takes a screenshot at each step, and maps the results to the linked issue's acceptance criteria (AC). Criteria that pass are ticked on the issue. It reports problems and does not fix code.

## What it does

1. **Gathers the target.** Reads the PR and its linked issue, then reads the repo's own docs (`CLAUDE.md`, `README`, scripts, compose files) to learn how to run the project locally.
2. **Dispatches a tester.** Sets up a worktree at the PR's head SHA, starts services and the dev server, seeds data, then runs:
   - unit tests for the touched directories
   - e2e specs covering the PR, with any failure compared against the base branch
   - manual browser flows: list → create → update → delete for each CRUD surface, with the stored record checked after each write
3. **Verifies.** Checks a few screenshots itself before trusting the report.
4. **Ticks the issue.** Flips `- [ ]` to `- [x]` for passing criteria under the *Acceptance criteria* heading only.
5. **Reports.** Sends the screenshots and a table of check → result, with an AC verdict for each criterion.

## Requirements

- Claude Code
- [`gh`](https://cli.github.com/) CLI, authenticated for the target repo
- `git`
- A project that can run locally (dev server, optional seed data and test users)
- A `tester` subagent. If you don't have one, the skill falls back to the cheapest capable agent.

## Install

Clone into your personal skills folder:

```bash
git clone https://github.com/Fishcanwalk/pr-qa ~/.claude/skills/pr-qa
```

To use it in a single project instead, clone into `<project>/.claude/skills/pr-qa`.

## Usage

Ask Claude in natural language, or invoke the skill directly:

```
/pr-qa 123
QA PR 123 against its issue's acceptance criteria
test this PR and screenshot the UI changes
```

## Making QA work better in your repo

The skill figures out how to run your project from its docs. The clearer those docs are, the better the run goes. Worth documenting in `CLAUDE.md` or `README`:

- install, dev-server, seed and test commands, plus the dev port
- backing services (database, queue, …) and a health check for each
- test users and fixtures to log in with (never real credentials)
- project quirks, such as where data lives or flows that need special setup

## Notes

- Fixing code is out of scope. Bugs go in the report.
- Only test users and seeded data are used. Records the run creates are deleted afterwards.
- Services started for the run are left running, and the dev server is stopped.
- It writes to the issue (ticks checkboxes), so only run it on PRs and issues you are allowed to edit.
