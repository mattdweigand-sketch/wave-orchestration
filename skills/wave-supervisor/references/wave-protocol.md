# Wave Protocol

The shared contract for a wave run. Every agent reads the copy at `.wave/PROTOCOL.md`.
The supervisor seeds that file from `wave-supervisor/references/wave-protocol.md` at the
start of a run. If this protocol conflicts with a role skill, this protocol wins.

## Cast

- **Supervisor:** shapes the spec with the human, splits it into bundles, dispatches a
  wave, reviews the whole wave, and writes cleanup prompts. Does not write feature code.
- **Workers:** each own one bundle per wave. Each worker implements in an isolated
  worktree or branch, opens one PR or local review branch, and responds to review.
- **Reviewer:** reviews each PR or local branch against its bundle's acceptance criteria.
  Does not write feature code.
- **Human:** shapes the spec, starts terminals or agent sessions, pastes prompts, tests
  the integrated result, and controls merges.

## Working Folder

All shared state lives in `.wave/` at the repo root. Agents do not share hidden memory;
these files are the coordination channel.

```text
.wave/
  PROTOCOL.md
  spec.md
  context/
  bundles/
    bundle-01-<slug>.md
  status/
    bundle-01.worker.json
  reviews/
    bundle-01.md
  waves/
    wave-1.md
  cleanup/
    wave-1-cleanup.md
```

## Write Ownership

- Supervisor writes `spec.md`, `context/*`, `bundles/*`, `waves/*`, and `cleanup/*`.
- Each worker writes only its own `status/<bundle-id>.worker.json` and implementation
  branch or worktree.
- Reviewer writes only `reviews/<bundle-id>.md`.
- Everyone reads everything.

One writer per coordination file avoids merge races and conflicting instructions.

## Bundle Format

```markdown
---
id: bundle-01
slug: short-slug
wave: 1
branch: wave1/bundle-01-short-slug
worker: worker-A
files_in_scope:
  - src/foo/**
depends_on: []
---

## Goal
One paragraph explaining what this bundle delivers and why.

## Context to load
- .wave/context/<file>
- exact/repo/path.ts

## Implementation notes
Concrete guidance, signatures, gotchas, and sequencing.

## Acceptance criteria
- [ ] Reviewer-verifiable outcome.

## Out of scope
Explicit work and files this bundle must not touch.

## Worker prompt
Paste-ready prompt that names this bundle file.
```

## Status Schema

```json
{
  "bundle": "bundle-01",
  "worker": "worker-A",
  "branch": "wave1/bundle-01-short-slug",
  "state": "assigned",
  "pr": null,
  "pr_url": null,
  "updated": "2026-05-30T00:00:00Z",
  "notes": ""
}
```

States: `assigned -> in_progress -> pr_open -> (changes_requested -> pr_open)* ->
approved -> merged`, plus `blocked` from any state.

## Review Format

```markdown
---
bundle: bundle-01
pr: 123
verdict: changes_requested
reviewed_at: 2026-05-30T00:00:00Z
---

## Summary
Short result.

## Required changes
- [ ] Blockers only.

## Nits
- Optional non-blocking comments.
```

## Wave File

`waves/wave-N.md` lists the bundle ids in that wave and one status line per bundle. The
supervisor keeps it current so the human has a dashboard.

## PR And Isolation Mechanics

Default mechanism is `github`. Each worker uses an isolated git worktree:

```bash
git worktree add ../<repo>__<slug> -b <branch>
```

Workers push branches and open PRs against the integration branch. PR titles start with
the bundle id, for example `bundle-01: add auth retry handling`. PR bodies include
`Bundle: bundle-01`.

For `local`, workers commit on local branches. The reviewer reads
`git diff <integration-branch>...<branch>` and writes the same review files.

## Wave Loop

1. Shape the spec with the human.
2. Copy relevant context into `.wave/context/`.
3. Write `.wave/spec.md`.
4. Split independent bundles with disjoint `files_in_scope`.
5. Dispatch worker prompts.
6. Workers implement and open PRs or local branches.
7. Reviewer writes verdict files.
8. Supervisor reviews the wave as a system and writes cleanup prompts.
9. Human tests, merges, and asks for the next wave.

## Safety Rules

- Bundles in the same wave have disjoint `files_in_scope`.
- If a worker needs a file outside scope, it stops and marks the bundle `blocked`.
- No agent merges unless the human explicitly delegates that action.
- Keep bundles roughly half a day of work or smaller.
