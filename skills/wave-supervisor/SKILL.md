---
name: wave-supervisor
description: "Supervise a multi-agent wave build. Use when the user says /wave-supervisor, start a wave, supervise this build, run the wave workflow, shape the spec then dispatch workers, kick off a build wave, or wants parallel coding agents across Claude, ChatGPT, Codex, or other terminals. Pairs with wave-worker and wave-reviewer."
version: "2.0"
category: "agentic-systems"
tags: ["waves", "multi-agent", "orchestration", "coding"]
---

# Wave Supervisor

Shape a build with the human, split it into independent bundles, dispatch worker prompts, then review the finished wave. Do not write feature code. The value of this skill is clean decomposition and coordination, not implementation.

## Contract

**Produces:** `.wave/spec.md`, `.wave/context/*`, `.wave/bundles/*`, `.wave/waves/*`, worker prompts, and `.wave/cleanup/*`.
**Consumes:** User goals, repo context, current branch state, worker status files, reviewer verdicts, and the shared protocol.
**Does not produce:** Feature code, worker PRs, reviewer verdicts, or automatic merges unless the human explicitly delegates a merge.

## Start Here

1. Read `references/wave-protocol.md` from this skill folder.
2. Initialize `.wave/` if absent:
   - `mkdir -p .wave/{context,bundles,status,reviews,waves,cleanup}`
   - Copy `references/wave-protocol.md` to `.wave/PROTOCOL.md`.
3. Explain the terminal layout once if the human has not run a wave before:
   - one supervisor terminal
   - one worker terminal per active bundle
   - one reviewer terminal
   - the human pastes prompts, tests the integrated result, and controls merges

Use the tool equivalents available in the current agent runtime. In Claude Code, use skills and shell tools. In ChatGPT or Codex, use the same filesystem and git operations through the available terminal/tools.

## Phase 1: Shape

Do not jump straight to bundle writing. First make the work crisp enough for parallel execution.

Ask for the standard of success, constraints, explicit non-goals, risk areas, and how the human will test the result. Name tradeoffs while they are cheap to change. If the decision changes architecture, scope, data shape, or ownership boundaries, ask before proceeding.

Assemble a clean context folder. Let the human describe relevant areas in plain language, then search the repo and copy the useful files or excerpts into `.wave/context/`. Prefer a focused context set over asking every worker to rediscover the repo.

## Phase 2: Spec And Split

Write `.wave/spec.md` with:
- overall goal
- integration branch, defaulting to `main`
- PR mechanism, defaulting to `github`, with `local` as fallback
- success criteria and test plan
- explicit out-of-scope work

Split the work into bundles. Each bundle should be one PR a worker can complete independently. Bundles in the same wave must have disjoint `files_in_scope`. If two bundles need the same file, sequence them across waves with `depends_on`, or define a thin interface in one bundle for the other to consume later.

Write each bundle as `.wave/bundles/bundle-NN-<slug>.md` using the format in `.wave/PROTOCOL.md`. Fill every section. Implementation notes and acceptance criteria should be concrete enough that the worker does not need to re-derive the spec.

Write `.wave/waves/wave-N.md` as the human dashboard. Seed `.wave/status/<bundle-id>.worker.json` for each assigned bundle.

## Phase 3: Dispatch

Output ready-to-paste worker prompts, one per target terminal:

```text
--- paste into worker-A ---
/wave-worker bundle-01-<slug>
```

If the target agent does not support slash commands, use:

```text
Run the wave-worker skill for .wave/bundles/bundle-01-<slug>.md.
```

Tell the human to start the reviewer after workers are running.

## Phase 4: Review The Wave

When PRs or local branches are ready, review the wave as a system:

1. Read `.wave/spec.md`, `.wave/waves/wave-N.md`, all relevant status files, and reviewer verdicts.
2. Inspect PR diffs with `gh pr diff` for `github`, or `git diff <integration-branch>...<branch>` for `local`.
3. Check each bundle against its acceptance criteria and against cross-bundle coherence.
4. Write `.wave/cleanup/wave-N-cleanup.md` with one paste-ready cleanup prompt per worker that needs changes.
5. Update `.wave/waves/wave-N.md` with the current status summary.

After cleanup, the human tests and merges. Start the next wave only after the human asks.

## Standing Rules

- Do not edit files inside any bundle's `files_in_scope`.
- Keep `.wave/waves/wave-N.md` current.
- Use exploratory subagents only if the runtime supports them. If not, search and read files directly.
- Ask on forking decisions. Parallel execution only works when the spec is explicit.

## References

- `references/wave-protocol.md`: shared state, bundle format, status schema, review format, and PR mechanics.
