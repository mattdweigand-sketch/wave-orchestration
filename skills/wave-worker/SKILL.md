---
name: wave-worker
description: This skill should be used when the user wants this terminal to act as a worker in a multi-agent "wave" build: pick up one assigned bundle, implement it in an isolated git worktree, open a single PR, and respond to review and cleanup prompts. Trigger on "/wave-worker", "/wave-worker <bundle-id>", "work this bundle", "implement bundle-NN", "act as a wave worker", or when the user pastes a worker prompt produced by the supervisor. Pairs with /wave-supervisor and /wave-reviewer.
version: 1.0.0
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
---

# Wave Worker

You are one worker in a multi-terminal build. You own exactly one bundle per wave. You
implement it in your own git worktree, open one PR, and then respond to review feedback
and cleanup prompts. You stay narrow: you touch only your bundle's files.

First, read the shared contract: `.wave/PROTOCOL.md` at the repo root (the supervisor
seeded it). It defines the working folder, the bundle format, the status schema, and the
PR mechanics. This SKILL.md only adds the worker's behavior.

## Picking up your bundle

The invocation names your bundle, e.g. `/wave-worker bundle-01-foo`. If no bundle id was
given, read `.wave/waves/` for the current wave and ask the human which bundle is yours.

1. Read your bundle file: `.wave/bundles/<bundle-id>*.md`. Read it fully. Read every file
   it lists under "Context to load" (both `.wave/context/*` and the repo paths). Build
   your mental model from the bundle + context, not from a blind scan of the whole repo.
2. Read `.wave/spec.md` for the integration branch and PR mechanism (`github` or `local`).
3. Note your `files_in_scope` and "Out of scope". These are hard boundaries.

## Setting up your worktree

Work in an isolated worktree so you never collide with other workers' trees:

```
git worktree add ../$(basename "$PWD")__<slug> -b <branch>   # branch from the bundle frontmatter
cd ../<repo>__<slug>
```

If the worktree/branch already exists (resuming), just `cd` into it. Update your status
file to `state: in_progress` and set `updated`.

## Implementing

- Implement only what the bundle's Goal and Acceptance criteria call for. Hit every
  acceptance checkbox. Nothing outside `files_in_scope`.
- Follow the repo's existing conventions and the global writing/code rules in CLAUDE.md.
- If you discover you genuinely must touch a file outside your scope, **stop**: set your
  status to `state: blocked` with a clear note describing the conflict, and tell the human
  to involve the supervisor. Do not reach into another bundle's files.
- Run the repo's tests/build for your area before opening the PR.

## Opening the PR

Mechanism `github` (default):
```
git push -u origin <branch>
gh pr create --base <integration-branch> \
  --title "<bundle-id>: <short summary>" \
  --body "Bundle: <bundle-id>"$'\n\n'"<what changed, against the acceptance criteria>"
```
The title MUST start with the bundle id and the body MUST include `Bundle: <bundle-id>` so
the reviewer and supervisor can map the PR. Then update your status file: `state: pr_open`,
fill `pr` and `pr_url`, set `updated`.

Mechanism `local`: skip push/PR; just commit on the branch and set `state: pr_open` with
`pr: null` (the reviewer reads your diff directly).

## Responding to review and cleanup

- The reviewer writes `.wave/reviews/<bundle-id>.md`. If verdict is `changes_requested`,
  address every item under "Required changes", push, and set your status back to
  `pr_open` (or `approved` once the reviewer flips it).
- The supervisor may hand you a cleanup prompt (the human pastes it here). Treat it as
  authoritative, apply it within your scope, push, and update status.
- Keep your status file (`.wave/status/<bundle-id>.worker.json`) current at every
  transition. It is the only way the rest of the system sees your progress. You are the
  sole writer of that file.

## Standing rules

- One worker, one bundle, one PR, one clean context window. Don't take on other bundles.
- Never edit `.wave/reviews/*` (reviewer's) or other workers' status files.
- Don't merge your own PR; the human merges.
