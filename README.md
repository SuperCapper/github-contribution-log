# github-contribution-log

# Contribution #1: Add debug logging of resolved config in train()

**Contribution Number:** 1  
**Student:** [David Jones]  
**Issue:** https://github.com/tinaudio/synth-setter/issues/33

**Status:** Phase II — In Progress

---

## Why I Chose This Issue

I chose this issue for several reasons that align with my current skillset and learning goals in this course. First, the issue is a perfect entry point for my first open source contribution. It's small, self-contained, and doesn't require deep domain knowledge of audio synthesis or machine learning. The problem is well-defined: add a single debug logging call in src/train.py. This lets me focus on learning the contribution workflow—cloning the repo, setting up the development environment, running tests, and submitting a PR—without getting overwhelmed by complex code changes.

Second, the issue introduces me to tools and patterns I'll encounter throughout the course. The project uses Hydra for configuration management, which is a common framework in ML research projects. Learning how to use OmegaConf.to_yaml() to dump a resolved config is a small but practical skill: it's exactly the kind of debugging technique I'll need when troubleshooting my own models in the future. According to Hydra's documentation, printing the composed config can be extremely helpful for debugging, as it allows me to see all runtime-resolved values.

Third, the project is fascinating. synth-setter tackles synthesizer parameter prediction ("synth inversion")-given an audio recording, it predicts the parameters needed to reproduce that sound. As somenone interested in AI for creative applications, I'm excited to contribute to a project at the intersection of machine learning and music technology.

Left a comment on the issue introducing myself and confirming I’d like to work on it. 

---

## Understanding the Issue

### Problem Description

When debugging test failures or unexpected training behavior in synth-setter, developers currently can't see the exact resolved Hydra configuration without manually adding temporary print statements and re-running their code. This makes it harder to verify that configuration overrides and defaults are being applied correctly, especially when dealing with Hydra's composition system where configs can be merged from multiple files.

### Expected Behavior

When a developer runs the training script with debug logging enabled (via log_cli_level=DEBUG or -v flags), the full resolved Hydra configuration should be printed to the logs early in the train() function—specifically right after seed setup and before any model instantiation begins. In normal runs (without debug logging), the output should remain quiet.

### Current Behavior

Currently, there is no debug logging of the resolved config. Developers who need to inspect the config must manually insert print(OmegaConf.to_yaml(cfg)) or use a debugger, which adds friction to the debugging process and can lead to inconsistent debugging practices across the team.

### Affected Components

src/train.py – The main training script that contains the train() function. The logging call should be inserted around line 61, after seed setup and before model instantiation begins.

Hydra/OmegaConf – The configuration management system used by the project. The change uses OmegaConf.to_yaml(cfg, resolve=True) to dump the resolved configuration in YAML format.

Logging system – The change uses log.debug(), which will only output when debug logging is enabled, keeping normal runs quiet.

---

## Reproduction Process

### Environment Setup

The project targets Linux/macOS only (README §Prerequisites). On Windows the
required path is the VS Code Dev Container (`.devcontainer/cpu/devcontainer.json`),
which runs `linux/amd64` inside Docker.

**Challenges faced and fixes applied:**

| Challenge | Fix |
|---|---|
| First `Reopen in Container` attempt failed with Wayland socket mount error | Disabled `dev.containers.mountWaylandSocket` in VS Code settings |
| Base image `tinaudio/synth-setter:devcontainer-tools` (18.9 GB) not cached — VS Code timed out | Pulled image manually with `docker pull` first, then reopened |
| `initialize.sh` failed: `$'\r': command not found` (exit 1) | Shell scripts cloned with Windows CRLF endings; converted all `.sh` files to LF with PowerShell and set `core.autocrlf=false` in repo git config |
| `postCreateCommand` failed: `/venv/main/bin/python3: bad interpreter: Permission denied` (exit 126) | Non-blocking for Phase II — pre-commit hooks not installed, but container started successfully |

Container confirmed running: `docker ps` shows `vsc-synth-setter-...` with status `Up`.

### Steps to Reproduce

The issue is a **missing feature** (absent code path), not a runtime crash.
Reproduction means confirming the debug call is absent and showing what the
output gap looks like.

1. Open `src/synth_setter/cli/train.py` in the devcontainer.
2. Locate the `train()` function (line 110). Observe the sequence after seed
   setup:
   ```python
   if cfg.get("seed"):
       L.seed_everything(cfg.seed, workers=True)   # line 121

   log.info(f"Instantiating datamodule <{cfg.datamodule._target_}>")  # line 123
   ```
3. **Observed:** There is no `log.debug(...)` call between lines 121 and 123.
   Running `python -m synth_setter.cli.train log_cli_level=DEBUG` produces no
   resolved-config output in the logs — the developer has no way to see the
   fully-composed Hydra config without inserting a manual `print()`.

### Reproduction Evidence

- **My findings:** The gap is confirmed at lines 121–123 of
  `src/synth_setter/cli/train.py`. Both prerequisites for the fix (`OmegaConf`
  imported at line 12, `log = RankedLogger(...)` defined at line 40) are
  already in scope — no new imports are needed.

---

## Solution Approach

### Analysis

The `train()` function composes a Hydra `DictConfig` that may merge defaults
from multiple YAML files plus command-line overrides. Developers currently
cannot inspect the final resolved values (interpolations expanded, overrides
applied) without adding a temporary `print()`. The fix is to emit that
snapshot at `DEBUG` level so it is visible when debug logging is enabled and
completely silent in normal runs.

### Proposed Solution

Insert one `log.debug` call in `train()` immediately after the seed setup and
before any object instantiation begins. Use `OmegaConf.to_yaml(cfg,
resolve=True)` so that all `${...}` interpolations are expanded in the output.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `train()` in `src/synth_setter/cli/train.py` never emits the
resolved Hydra config at any log level. A developer running with debug logging
enabled sees no config snapshot, making it hard to verify that overrides and
defaults composed correctly.

**Match:** The file already uses `log.info(...)` for every major step. The
`log` object is a `RankedLogger` (rank-zero-only stdlib `logging` wrapper).
`OmegaConf.to_yaml(cfg, resolve=True)` is the standard Hydra idiom for
dumping a resolved config. Both are already imported and in scope.

**Plan:**
1. In `src/synth_setter/cli/train.py`, after line 121 (`L.seed_everything`)
   and before line 123 (the first `log.info`), insert:
   ```python
   log.debug("Resolved Hydra config:\n%s", OmegaConf.to_yaml(cfg, resolve=True))
   ```
2. Add a unit test in `tests/` that patches `log.debug`, calls `train()` with
   a minimal mock config, and asserts `log.debug` was called with a string
   containing the YAML representation of the config.
3. Run `make test-fast` to confirm no regressions.
4. Run `make format` to pass pre-commit (ruff, pydoclint, gitlint).

**Implement:** Branch `feat/debug-log-resolved-config` — link to commits added
here as work progresses.

**Review:**
- [ ] Single-line change only — no scope creep
- [ ] `resolve=True` used so interpolations are expanded
- [ ] `log.debug` (not `print` or `log.info`) — silent in normal runs
- [ ] Conventional commit title: `internal-feat(train): log resolved Hydra cfg at DEBUG`
- [ ] No `Co-Authored-By` trailer (blocked by repo pre-commit hook)
- [ ] `make format` passes (ruff, pydoclint, gitlint)

**Evaluate:** Run training with `log_cli_level=DEBUG` (or `pytest -s` with the
new test). The resolved YAML config should appear in the log output immediately
after seed setup and before the first `Instantiating datamodule` line.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: `test_train_logs_resolved_cfg_at_debug_when_debug_enabled` — patch `log.debug`, call `train()` with a minimal mock config, assert `log.debug` was called with a string containing `OmegaConf.to_yaml` output
- [ ] Test case 2: `test_train_does_not_log_cfg_at_info` — assert `log.info` was NOT called with the config YAML (config dump must be debug-only)
- [ ] Test case 3: `test_train_cfg_debug_log_fires_after_seed_before_datamodule` — assert the debug call order relative to `seed_everything` and `hydra.utils.instantiate`

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
