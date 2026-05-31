# Wave Protocol

The shared contract for a wave run. Every agent (supervisor, workers, reviewer) obeys
this file. The supervisor copies it verbatim into `.wave/PROTOCOL.md` at the start of a
run. If it ever conflicts with a SKILL.md, this file wins.

## The cast

- **Supervisor** (1 terminal, `/wave-supervisor`): shapes the spec with the human, splits
  it into bundles, dispatches a wave, then reviews the whole wave and writes cleanup
  prompts. Never writes feature code.
- **Workers** (N terminals, `/wave-worker`, default 4): each owns one bundle per wave.
  Implements in an isolated git worktree, opens a PR, responds to review and cleanup.
- **Reviewer** (1 terminal, `/wave-reviewer`): polls open PRs for the wave and reviews
  each against its bundle's acceptance criteria. Never writes feature code.
- **Human** (Matt): shapes the spec with the supervisor, pastes prompts into worker
  terminals, tests after each wave, says go for the next wave.

## The working folder

All shared state lives in `.wave/` at the repo root. It is the only channel between
terminals; agents do not share memory. Treat the files as the source of truth.

```
.wave/
  PROTOCOL.md              # this contract, copied verbatim by the supervisor
  spec.md                  # overall spec, shaped collaboratively
  context/                 # copies of relevant files for clean context (read-only refs)
  bundles/
    bundle-01-<slug>.md    # one self-contained unit of work = one PR
    bundle-02-<slug>.md
  status/
    bundle-01.worker.json  # written ONLY by the worker who owns that bundle
  reviews/
    bundle-01.md           # written ONLY by the reviewer
  waves/
    wave-1.md              # which bundles are in wave N + a status summary
  cleanup/
    wave-1-cleanup.md      # supervisor's per-worker cleanup prompts after a wave
```

### Write-ownership (no two agents write the same file)

- A worker writes only `bundles/<its-bundle>` status via `status/<bundle>.worker.json`,
  plus its own worktree/branch. It never edits another bundle's files.
- The reviewer writes only `reviews/<bundle>.md`.
- The supervisor writes `spec.md`, `bundles/*`, `waves/*`, `cleanup/*`, and `context/*`.
- Everyone reads everything. Only one writer per file means no merge races.

## Bundle file format (`bundles/bundle-NN-<slug>.md`)

```
---
id: bundle-01
slug: short-slug
wave: 1
branch: wave1/bundle-01-short-slug
worker: unassigned        # supervisor sets to worker-A..D when dispatching
files_in_scope:           # globs this bundle is allowed to touch (keep disjoint!)
  - src/foo/**
depends_on: []            # bundle ids that must merge first
---

## Goal
One paragraph: what this bundle delivers and why.

## Context to load
- .wave/context/<file>  (and exact repo paths the worker should read first)

## Implementation notes
Concrete guidance, signatures, gotchas. Enough to execute without re-deriving the spec.

## Acceptance criteria
Checklist the reviewer scores against. "What good looks like."
- [ ] ...

## Out of scope
Explicitly what NOT to touch (protects other bundles' files_in_scope).

## Worker prompt
The exact text the human pastes into a worker terminal. Must name the bundle file path.
```

## Status file schema (`status/bundle-NN.worker.json`)

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

`state` transitions:
`assigned -> in_progress -> pr_open -> (changes_requested -> pr_open)* -> approved -> merged`
plus `blocked` from any state (worker sets `notes` with the blocker).

## Review file format (`reviews/bundle-NN.md`)

```
---
bundle: bundle-01
pr: 123
verdict: changes_requested   # or approve
reviewed_at: 2026-05-30T00:00:00Z
---

## Summary
...

## Required changes
- [ ] ...  (only blockers; the reviewer is strict but not pedantic)

## Nits (non-blocking)
- ...
```

## Wave file format (`waves/wave-N.md`)

Lists the bundle ids in this wave and a one-line status per bundle (the supervisor keeps
this current by reading the status + review files). This is the human's dashboard.

## PR + isolation mechanics

- Default mechanism is **real GitHub PRs**. Each worker runs in its own **git worktree**
  so workers never collide on the working tree:
  `git worktree add ../<repo>__<slug> -b <branch>`
- Worker pushes the branch and opens a PR with `gh pr create`, base = the wave's
  integration branch (default `main` unless the supervisor sets otherwise in `spec.md`).
- The PR title MUST start with the bundle id, e.g. `bundle-01: <summary>`, so the
  reviewer and supervisor can map PRs to bundles.
- The PR body MUST include a line `Bundle: bundle-01`.
- **Local-only fallback** (no GitHub): set `mechanism: local` in `spec.md`. Workers stay
  on local branches in worktrees; reviewer reads `git diff main...<branch>` instead of
  `gh`, and writes the same `reviews/*.md`. Everything else is identical.

## The wave loop

1. **Shape** (supervisor + human): collaborative. Define what good looks like before any
   code. Pull relevant files into `.wave/context/`.
2. **Spec + split** (supervisor): write `spec.md`, carve disjoint bundles, write
   `waves/wave-N.md`, assign a worker to each bundle, emit worker prompts.
3. **Dispatch** (human): paste each worker prompt into a worker terminal. Workers run in
   parallel, each opening one PR.
4. **Auto-review** (reviewer): polls open PRs, reviews each against its bundle's
   acceptance criteria, writes `reviews/*.md`, sets verdict.
5. **Wave review** (supervisor): once PRs are open and reviewed, reads the entire wave and
   writes `cleanup/wave-N-cleanup.md` with one cleanup prompt per worker that needs it.
6. **Cleanup** (human -> workers): paste cleanup prompts back into the worker terminals.
7. **Test** (human): merge approved PRs, test the integrated result.
8. **Next wave**: human tells the supervisor to start wave N+1.

## Rules that keep waves safe

- Bundles in the same wave must have **disjoint `files_in_scope`**. If two need the same
  file, sequence them across waves with `depends_on`, or give one a thin interface the
  other consumes.
- Workers never touch files outside their `files_in_scope`. If they must, they stop, set
  `state: blocked` with a note, and wait for the supervisor.
- No agent merges PRs except the human (or the supervisor only if the human has said so).
- Keep each bundle to roughly a half-day of work or less. Smaller bundles = cleaner PRs
  and fewer conflicts.
