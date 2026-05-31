---
name: wave-worker
description: "Implement one assigned bundle in a multi-agent wave build. Use when the user says /wave-worker, /wave-worker <bundle-id>, work this bundle, implement bundle-NN, act as a wave worker, or pastes a worker prompt from the supervisor. Works across Claude, ChatGPT, Codex, or other agent terminals. Pairs with wave-supervisor and wave-reviewer."
version: "2.0"
category: "agentic-systems"
tags: ["waves", "worker", "worktree", "pull-request"]
---

# Wave Worker

Own exactly one bundle. Implement it in an isolated git worktree, open one PR or local branch, and respond to review. Stay narrow so parallel work does not collide.

## Contract

**Produces:** One implementation branch or PR, updates to `.wave/status/<bundle-id>.worker.json`, and responses to reviewer or supervisor cleanup prompts.
**Consumes:** `.wave/PROTOCOL.md`, `.wave/spec.md`, one bundle file, listed context files, repo guidance files, and review files for your bundle.
**Does not produce:** Other bundles, supervisor files, reviewer verdicts, merges, or broad refactors outside the assigned scope.

## Pick Up The Bundle

The invocation should name a bundle, for example `/wave-worker bundle-01-auth-flow`. If no bundle id is provided, read `.wave/waves/` for the current wave and ask which bundle is yours.

1. Read `.wave/PROTOCOL.md`.
2. Read `.wave/spec.md` for the integration branch and PR mechanism.
3. Read `.wave/bundles/<bundle-id>*.md` fully.
4. Read every path listed under the bundle's `Context to load`.
5. Note `files_in_scope`, `Out of scope`, and acceptance criteria before editing.

Follow local repo instructions from whichever file exists for this runtime: `AGENTS.md`, `CLAUDE.md`, `.cursor/rules`, or equivalent. The bundle remains the scope contract.

## Set Up The Worktree

Use the branch from the bundle frontmatter.

```bash
git worktree add ../$(basename "$PWD")__<slug> -b <branch>
cd ../<repo>__<slug>
```

If the worktree or branch already exists, resume there. Update `.wave/status/<bundle-id>.worker.json` to `state: in_progress` with a fresh `updated` timestamp.

## Implement

Implement only the bundle goal and acceptance criteria. Touch only `files_in_scope`.

If you discover a required edit outside scope, stop. Set your status to `blocked`, explain the conflict in `notes`, and ask the human to involve the supervisor. Do not silently expand the bundle.

Run the relevant tests, type checks, build, or focused smoke checks before opening the PR.

## Open The PR Or Local Branch

For `github`:

```bash
git push -u origin <branch>
gh pr create --base <integration-branch> \
  --title "<bundle-id>: <short summary>" \
  --body "Bundle: <bundle-id>"$'\n\n'"<what changed, mapped to acceptance criteria>"
```

The PR title starts with the bundle id. The body includes `Bundle: <bundle-id>`.

For `local`, commit on the branch and skip push/PR. The reviewer will read the local diff.

Update your status to `pr_open`, fill `pr` and `pr_url` when available, and set `updated`.

## Respond To Review

Read `.wave/reviews/<bundle-id>.md` when it appears.

If the verdict is `changes_requested`, address every required change within scope, rerun checks, push or commit, and set status back to `pr_open`.

If the supervisor gives a cleanup prompt, treat it as authoritative for this bundle. Apply it within scope and update status.

## Standing Rules

- One worker, one bundle, one branch or PR.
- Never edit `.wave/reviews/*`, `.wave/waves/*`, `.wave/spec.md`, or another worker's status file.
- Do not merge your own PR.
- Keep your status file current. It is the coordination channel.
