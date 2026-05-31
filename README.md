# wave-orchestration

Agent-agnostic wave build kit for parallel coding work. One supervisor shapes the spec
and splits it into bundles. Workers each implement one bundle in an isolated worktree.
A reviewer checks each PR or local branch against the bundle's acceptance criteria. The
human tests, merges, and starts the next wave.

The skills are written for Claude, ChatGPT/Codex, or any agent that can read files and
run git commands. Claude-specific setup lives only in `CLAUDE.md` as a light pointer.

## What's Here

```text
skills/
  wave-supervisor/
    SKILL.md
    references/wave-protocol.md
  wave-worker/
    SKILL.md
  wave-reviewer/
    SKILL.md
evals/
  evals.json
CLAUDE.md
```

## Install

Install the three folders wherever your agent runtime loads local skills.

Claude Code:

```bash
mkdir -p ~/.claude/skills
cp -R skills/wave-supervisor skills/wave-worker skills/wave-reviewer ~/.claude/skills/
```

Codex or ChatGPT-style local skills:

```bash
mkdir -p ~/.codex/skills
cp -R skills/wave-supervisor skills/wave-worker skills/wave-reviewer ~/.codex/skills/
```

Symlink instead of copying if you want local edits to update the installed skills.

## Run A Wave

1. Start `wave-supervisor` in the target repo.
2. Shape the spec with the supervisor.
3. Open one worker terminal or session per active bundle and run `wave-worker` with the
   bundle id the supervisor gives you.
4. Start `wave-reviewer` so reviews are written as PRs or local branches appear.
5. Test and merge after cleanup, then ask the supervisor for the next wave.

All coordination happens through `.wave/` in the target repo. The shared contract is
seeded from `skills/wave-supervisor/references/wave-protocol.md` into `.wave/PROTOCOL.md`.
