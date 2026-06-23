# skillopt-sleep-pi

A [pi](https://github.com/earendil-works/pi-coding-agent) package that brings
[**SkillOpt-Sleep**](https://github.com/microsoft/SkillOpt) to the pi coding agent:
a nightly **sleep cycle** that reviews your real past pi sessions, replays your
recurring tasks, and consolidates what it learns into `AGENTS.md` / `SKILL.md` —
**but only keeps changes that pass a held-out validation gate**, and only after
you adopt them. No weight training, zero inference-time overhead.

This is the pi-native counterpart of SkillOpt-Sleep's Claude Code / Codex / Copilot
plugins. It uses pi's first-class skills system: **no extension, no hook, no daemon** —
just a skill that drives the upstream Python engine.

## Install

```bash
pi install git:github.com/auspic7/skillopt-sleep-pi
```

Then clone the SkillOpt engine once and point pi at it (add to `~/.zshrc`):

```bash
git clone https://github.com/microsoft/SkillOpt.git ~/src/SkillOpt
export SKILLOPT_SLEEP_REPO="$HOME/src/SkillOpt"
```

Requires Python >= 3.10.

## Use

In pi, just describe what you want — the skill auto-loads on intent:

> "run the sleep cycle on this project"
> "review my past pi sessions and consolidate what you learned"
> "make my agent better the more I use it"

Or invoke directly:

```bash
RUNNER="$SKILLOPT_SLEEP_REPO/plugins/run-sleep.sh"

bash "$RUNNER" status   --source pi --project "$(pwd)"   # what's happened so far
bash "$RUNNER" dry-run  --source pi --project "$(pwd)"   # safe preview, no staging
bash "$RUNNER" run      --source pi --project "$(pwd)"   # full cycle, stages a proposal
bash "$RUNNER" adopt    --source pi --project "$(pwd)"   # apply staged proposal (backup first)
```

- `--source pi` harvests your `~/.pi/agent/sessions/**/*.jsonl`.
- `--backend mock` (default) is free and deterministic — try the plumbing first.
- `--backend claude`/`codex` spends your real budget for genuine lift.

See the skill's [`SKILL.md`](./skills/skillopt-sleep/SKILL.md) for the full flag
reference, scheduling (nightly cron), and the hard safety rules.

## Why

pi transcripts expose a signal the other agent sources lack: **per-call `isError`**
on `toolResult` messages. SkillOpt-Sleep surfaces it as `neg:tool_error:<tool>`
feedback, which is exactly the kind of checkable success/failure signal the
validation gate thrives on — so pi sessions are unusually good raw material for
gated skill consolidation.

## How it works

```
harvest pi sessions → mine recurring tasks → replay offline
   → consolidate (reflect → bounded edit → GATE on held-out tasks)
   → stage proposal → (you) adopt
```

Three ideas synthesized (from the upstream SkillOpt-Sleep docs):

- **SkillOpt** — the skill/memory doc is trainable text; bounded add/delete/replace
  edits; accepted only through a held-out gate; rejected edits become negative feedback.
- **Claude Dreams** — offline consolidation that reads past sessions and rebuilds
  memory; input never mutated; output reviewed then adopted.
- **Agent sleep** — periodic offline replay turns episodes into durable skill.

## Requirements

- pi coding agent (this is a pi package).
- SkillOpt engine (one-time clone; MIT).
- Python >= 3.10.
- An API key for the chosen replay backend if you move past `--backend mock`.

## Status

Preview — tracks SkillOpt-Sleep's own preview status. The pi session source
(`--source pi`) is contributed upstream at [microsoft/SkillOpt#83](https://github.com/microsoft/SkillOpt/pull/83).

## License

MIT
