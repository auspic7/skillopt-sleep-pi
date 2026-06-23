---
name: skillopt-sleep
description: "Offline self-evolution for pi. Use when the user wants their pi agent to improve from past usage, asks about a nightly/offline 'sleep' or 'dream' cycle, memory/skill consolidation, or says things like 'make my agent better the more I use it', 'review my past sessions', 'learn my preferences', 'consolidate what you learned', or 'run the sleep cycle'. Drives the SkillOpt-Sleep engine to harvest pi sessions -> mine recurring tasks -> replay offline -> consolidate validated SKILL.md/AGENTS.md behind a held-out gate."
---

# SkillOpt-Sleep: offline self-evolution for pi

Give the user's **pi** agent a **sleep cycle**. While the user is offline
(e.g. nightly), SkillOpt-Sleep reviews their real past pi sessions
(`~/.pi/agent/sessions/<slug>/*.jsonl`), re-runs recurring tasks on their own
API budget, and consolidates what it learns into **memory** (`AGENTS.md`) and
**skills** (`SKILL.md`) — but only keeps changes that pass a held-out
validation gate, and only after the user adopts them. No model-weight
training; zero inference-time overhead.

It synthesizes three ideas:
- **SkillOpt** — the skill/memory doc is trainable text; bounded add/delete/replace
  edits; accepted only through a held-out gate; rejected edits become negative feedback.
- **Claude Dreams** — offline consolidation that reads past sessions and rebuilds
  memory (dedup/merge/resolve); the input is never mutated; output is reviewed then adopted.
- **Agent sleep** — periodic offline replay turns episodes into durable skill.

## Prerequisites

The SkillOpt-Sleep engine is a small stdlib-only Python package. Clone it once and
point pi at it:

```bash
git clone https://github.com/microsoft/SkillOpt.git ~/src/SkillOpt
# add to your shell profile (~/.zshrc / ~/.bashrc):
export SKILLOPT_SLEEP_REPO="$HOME/src/SkillOpt"
pip install -e "$SKILLOPT_SLEEP_REPO"   # optional; the runner finds the package on its own
```

Requires Python >= 3.10. The engine is invoked through a single runner script,
so this skill needs **no pi extension and no hook** — pi just runs the skill.

## When to use this skill

Trigger when the user wants any of:
- "make my agent learn from how I use it" / "get better the more I use it"
- a nightly/scheduled or on-demand **offline self-improvement / dream / sleep** run
- to **review past pi sessions** and distill recurring tasks
- to **consolidate** feedback into `AGENTS.md` or a managed skill
- to **schedule** the cycle (cron) or **adopt** a staged proposal

## The cycle (six stages)

1. **Harvest** — read `~/.pi/agent/sessions/**/*.jsonl` (READ-ONLY) → session digests.
2. **Mine** — digests → `TaskRecord`s (recurring intents + outcome labels). pi's per-call
   `isError` on `toolResult` messages becomes a `neg:tool_error:<tool>` feedback signal —
   a real checkable outcome the gate can exploit.
3. **Replay** — re-run tasks offline under the *current* skill+memory → (hard, soft) scores.
4. **Consolidate** — reflect on failures → propose bounded edits → **gate** on a held-out
   slice; accept only if it strictly improves.
5. **Stage** — write proposed files + a diff + `report.md` into
   `<project>/.skillopt-sleep/staging/<date>/`. **Nothing live changes.**
6. **Adopt** — explicit (or opt-in auto): copy staged files over live ones, backing up first.

## How to drive it

```bash
RUNNER="$SKILLOPT_SLEEP_REPO/plugins/run-sleep.sh"

bash "$RUNNER" status   --source pi --project "$(pwd)"    # what's happened so far
bash "$RUNNER" dry-run  --source pi --project "$(pwd)"    # safe preview, no staging
bash "$RUNNER" run      --source pi --project "$(pwd)"    # full cycle, stages a proposal
bash "$RUNNER" adopt    --source pi --project "$(pwd)"    # apply staged proposal (with backup)
```

- `--source pi` selects pi session harvesting (native; added upstream).
- Default backend is `mock` (deterministic, **no API spend**) — good for trying the plumbing.
- `--backend pi` (recommended for pi users) replays via your local `pi -p`, using
  whatever model you've configured in pi (e.g. `zai/glm-5.2`) — no separate API
  key if pi is already authenticated. Keeps source and backend on the same agent.
- Add `--backend claude` or `--backend codex` to spend those vendors' budgets instead.
- Scope defaults to the invoked project; `--scope all` harvests every project.

### Scheduling (nightly)

```bash
# nightly at 03:17 (off the hour to avoid API spikes)
( crontab -l 2>/dev/null; echo "17 3 * * * \"$SKILLOPT_SLEEP_REPO/plugins/run-sleep.sh\" run --source pi --project \"$(pwd)\" --scope invoked --backend mock >> \"$(pwd)/.skillopt-sleep/cron.log\" 2>&1" ) | crontab -
```

For real lift, set `--backend anthropic` (or `claude`/`codex`) and ensure the
relevant API key is in the environment cron will see.

## Useful flags

| Flag | Default | Description |
|---|---|---|
| `--source pi` | (set above) | Harvest pi sessions |
| `--scope all\|invoked` | invoked | Harvest scope |
| `--backend mock\|pi\|claude\|codex\|copilot` | mock | Replay backend (mock = no API spend; pi = use pi's configured model) |
| `--model NAME` | backend default | Override replay model |
| `--lookback-hours N` | 72 | Harvest window (0 = full history) |
| `--max-tasks N` | 40 | Cap mined tasks |
| `--target-skill-path PATH` | auto | Explicit SKILL.md to evolve |
| `--auto-adopt` | off | Auto-adopt if gate passes |
| `--edit-budget N` | 4 | Max bounded edits per night |
| `--pi-home PATH` | ~/.pi | Override pi home for harvest |

Config keys live in `~/.skillopt-sleep/config.json` (`gate_mode`, `gate_metric`,
`evolve_memory`, `evolve_skill`, `preferences`, `recall_k`, `dream_rollouts`, ...).

## Memory consolidation targets

The sleep cycle can consolidate both:
- **SKILL.md** — a managed skill file (bounded edits: add/delete/replace)
- **AGENTS.md** — the project memory pi loads natively

Both are gated by the same held-out validation score. Set `evolve_memory: false` to
consolidate only skills, or `evolve_skill: false` for only memory.

## Hard rules

- **Never** hand-edit the user's `AGENTS.md` / `SKILL.md` as part of this skill.
  Only the `adopt` action changes live files, and it backs them up first.
- Harvest is read-only. `mock` replay has no side effects and spends nothing.
- Always show the user the **held-out baseline → candidate** score and the exact
  proposed edits before suggesting adoption. Evidence before adoption.
- If asked whether it really helps, run
  `python -m skillopt_sleep.experiments.run_experiment --persona researcher --assert-improves`
  — a deterministic demo (no API) that proves held-out lift and that the gate blocks
  harmful edits.
