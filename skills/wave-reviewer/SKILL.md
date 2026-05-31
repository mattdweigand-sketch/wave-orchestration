---
name: wave-reviewer
description: "Review PRs or local branches in a multi-agent wave build against each bundle's acceptance criteria. Use when the user says /wave-reviewer, review the wave PRs, auto-review the wave, poll GitHub and review, act as the wave spec reviewer, or when the supervisor tells the human to start the reviewer. Works across Claude, ChatGPT, Codex, or other agent terminals. Pairs with wave-supervisor and wave-worker."
version: "2.0"
category: "agentic-systems"
tags: ["waves", "review", "pull-request", "acceptance-criteria"]
---

# Wave Reviewer

Review each bundle's PR or local branch against its acceptance criteria. Write a verdict file the worker and supervisor can act on. Do not write feature code and do not merge.

## Contract

**Produces:** `.wave/reviews/<bundle-id>.md` files and optional PR comments.
**Consumes:** `.wave/PROTOCOL.md`, `.wave/spec.md`, `.wave/waves/wave-N.md`, bundle files, worker status files, and PR or local diffs.
**Does not produce:** Feature edits, worker status updates, supervisor cleanup prompts, approvals that merge code, or merges.

## Start Here

1. Read `.wave/PROTOCOL.md`.
2. Read `.wave/spec.md` for `mechanism` and `integration_branch`.
3. Read the current `.wave/waves/wave-N.md`.
4. For each listed bundle, read its bundle file and status file.

The bundle's acceptance criteria are the rubric. Review against the agreed spec, not personal preference.

## Poll The Wave

Run a poll pass, wait for new PRs or updated commits, then poll again until every current-wave bundle has a verdict and the human stops the reviewer.

For `github`:

```bash
gh pr list --state open --json number,title,headRefName,updatedAt
```

Map PRs by the `bundle-NN` title prefix and `Bundle:` body line. For new or updated PRs, inspect:

```bash
gh pr diff <number>
```

For `local`, read branches from `.wave/status/*.worker.json` and inspect:

```bash
git diff <integration-branch>...<branch>
```

Do not busy-spin. Sleep between poll passes or rerun when the human or supervisor pings you.

## Review One Bundle

1. Read the bundle's Goal, Implementation notes, Acceptance criteria, `files_in_scope`, and Out of scope.
2. Check the diff against every acceptance checkbox.
3. Verify the diff stayed inside `files_in_scope`.
4. Look for correctness issues, missing criteria, obvious regressions, and seam problems with sibling bundles.
5. Separate blockers from nits. Do not invent new requirements.

Write `.wave/reviews/<bundle-id>.md` using the protocol format with:
- `verdict: approve` or `verdict: changes_requested`
- short summary
- required changes checklist for blockers only
- optional nits

For `github`, also post a PR comment with the same actionable summary:

```bash
gh pr review <number> --comment --body "<summary + required changes>"
```

Use `--comment`, not `--approve`; the human owns merge decisions.

## Standing Rules

- Review against the bundle contract every time.
- Never edit feature code or worker status files.
- You are the sole writer of `.wave/reviews/*`.
- Keep reviews tight enough that a worker can act without re-reading the whole PR.
