# wave-orchestration

A multi-agent "wave" build kit for Claude Code: one supervisor, one spec reviewer, and N
workers across separate terminals. The supervisor shapes a spec with you and splits it
into modular **bundles**; each worker cuts one PR in its own git worktree; the reviewer
auto-reviews PRs against each bundle's acceptance criteria; the supervisor reviews the
whole wave and hands back cleanup prompts. Then you test and start the next wave.

## What's here

```
skills/
  wave-supervisor/      # /wave-supervisor — shapes spec, splits bundles, dispatches, reviews the wave
    SKILL.md
    assets/PROTOCOL.md  # the shared contract all agents obey (seeded into .wave/ at run time)
  wave-worker/          # /wave-worker — owns one bundle, one worktree, one PR
    SKILL.md
  wave-reviewer/        # /wave-reviewer — polls PRs, auto-reviews against acceptance criteria
    SKILL.md
```

## Install on another device

Skills live as local folders at `~/.claude/skills/<name>/`. After cloning this repo,
copy (or symlink) the three skill folders into place:

```bash
git clone https://github.com/mattdweigand-sketch/wave-orchestration.git
cd wave-orchestration

# copy
cp -R skills/wave-supervisor skills/wave-worker skills/wave-reviewer ~/.claude/skills/

# or symlink (so `git pull` updates the live skills)
ln -s "$PWD/skills/wave-supervisor" ~/.claude/skills/wave-supervisor
ln -s "$PWD/skills/wave-worker"     ~/.claude/skills/wave-worker
ln -s "$PWD/skills/wave-reviewer"   ~/.claude/skills/wave-reviewer
```

The folder name is the slash command, so this gives you `/wave-supervisor`,
`/wave-worker`, and `/wave-reviewer`.

## Running a wave

1. Terminal 1: `/wave-supervisor` — talk through what to build; it writes the spec and bundles.
2. Terminals 2-5: `/wave-worker` — paste the per-bundle prompts the supervisor hands you.
3. Terminal 6: `/wave-reviewer` — it polls open PRs and reviews each one.
4. The supervisor reviews the wave and gives you cleanup prompts. Paste them, test, repeat.

All coordination happens through a `.wave/` folder in the target repo. See
`skills/wave-supervisor/assets/PROTOCOL.md` for the full contract.
