# github-contribution-log

# Contribution #1: Add debug logging of resolved config in train()

**Contribution Number:** 1  
**Student:** [David Jones]  
**Issue:** https://github.com/tinaudio/synth-setter/issues/33

**Status:** Phase III — Complete

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

- **Working branch:** https://github.com/SuperCapper/synth-setter/tree/fix-issue-33
- **My findings:** The gap is confirmed at lines 121–123 of
  `src/synth_setter/cli/train.py`. Both prerequisites for the fix (`OmegaConf`
  imported at line 12, `log = RankedLogger(...)` defined at line 40) are
  already in scope — no new imports are needed.

**Step 3 answers:**

- **What I expected to happen:** Running `python -m synth_setter.cli.train log_cli_level=DEBUG`
  should print the fully-resolved Hydra config (all `${...}` interpolations
  expanded, all overrides applied) to the log immediately after seed setup and
  before any object is instantiated.
- **What actually happened:** No config output appears at any log level. The
  gap between seed setup (line 121) and the first `log.info` (line 123) is
  empty — there is no `log.debug` call.
- **Files involved:** `src/synth_setter/cli/train.py` (only file that needs to
  change). Relevant test file: `tests/test_train.py` (where new assertions will
  be added).
- **Where to look:** `train()` function, lines 119–123 — the block between
  `L.seed_everything` and the first `log.info`.

---

## Solution Approach

### Step 4 — Four Questions

**1. Root cause?**
There is no `log.debug` call in `train()` that dumps the resolved config.
The function composes a Hydra `DictConfig` from multiple YAML files plus
command-line overrides, but never logs the final merged result. The
responsible gap is in `src/synth_setter/cli/train.py`, `train()` function,
between lines 121 and 123 — after `L.seed_everything` and before the first
`log.info`.

**2. Proposed fix?**
I will add one `log.debug` call in `train()`, immediately after the seed
setup block and before any object is instantiated, that emits the fully
resolved Hydra config as YAML:
```python
log.debug("Resolved Hydra config:\n%s", OmegaConf.to_yaml(cfg, resolve=True))
```
Both `OmegaConf` and `log` are already imported — no new dependencies.

**3. Files to modify:**
- `src/synth_setter/cli/train.py` — insert the one-line debug call
- `tests/test_train.py` — add assertions that the debug call fires with the correct content

**4. How will I verify it works?**
- Write a unit test that patches `log.debug`, calls `train()` with a minimal
  mock config, and asserts the call was made with a string containing the
  YAML representation of the config.
- Run `make test-fast` (CPU-only fast tests) to confirm no regressions.
- Run `make format` to pass ruff, pydoclint, and gitlint pre-commit checks.

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

**Implement:** https://github.com/SuperCapper/synth-setter/tree/fix-issue-33 — commits added here as work progresses.

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

- [x] `test_train_debug_logs_resolved_cfg_before_datamodule` — patches both `OmegaConf.to_yaml` and `log.debug` in the train module; asserts `to_yaml` is called with `resolve=True` and that `log.debug` receives "Resolved Hydra config" as its format string and the YAML string as its second argument. Added to `tests/test_train.py`.

### Integration Tests

- [x] `test_train_fast_dev_run_tiny_model_tiny_data` (existing) — confirmed still passes after the fix, exercising the fallback path (resolve fails on test fixture's `hydra.*` internal keys, falls back to `resolve=False` gracefully).

### Manual Testing

Ran the full fast test suite (`2022 passed, 5 skipped`) with only two pre-existing failures unrelated to this change (`test_post_create_performance` — known devcontainer permission issue; `test_pinned_baselines_resolve` — network/W&B artifact unavailable). No regressions introduced.

---

## Implementation Notes

### Phase III Progress

Implemented the one-line fix from the Phase II plan. The core change in `src/synth_setter/cli/train.py` adds a `log.debug` call immediately after `L.seed_everything` and before the first `log.info`, exactly as planned. The implementation uses a try/except fallback to `resolve=False` because the test fixtures compose the Hydra config with `return_hydra_config=True`, which includes internal Hydra keys (`hydra.sweep.subdir`, `hydra.job.num`) set to `???` — these cannot be resolved outside of a real Hydra job. In production the `cfg` passed to `train()` never contains these internal keys, so the try path always succeeds.

**Challenges:** The primary challenge was the test fixture. Three iterations were needed: (1) pre-computing `OmegaConf.to_yaml(cfg_train)` before `train()` failed because the fixture has unresolvable Hydra-internal interpolations; (2) adding `run_name = "test"` to the fixture resolved one key but not `hydra.job.num`; (3) the final approach mocks `OmegaConf.to_yaml` in the train module, asserting it is called with `resolve=True` without actually executing resolution on the test fixture — this is correct unit test isolation.

### Code Changes

- **Files modified:** `src/synth_setter/cli/train.py`, `tests/test_train.py`
- **Key commits:** [86c27ca](https://github.com/SuperCapper/synth-setter/commit/86c27ca) — `internal-feat(train): log resolved Hydra cfg at DEBUG`
- **Branch:** https://github.com/SuperCapper/synth-setter/tree/fix-issue-33
- **Approach decisions:** Used try/except with `# noqa: BLE001` (matching the existing pattern in `_log_model_artifact`) so a resolution failure never aborts training. Mocked `OmegaConf.to_yaml` in the test rather than trying to satisfy all of the Hydra fixture's internal interpolation requirements.

---

## Pull Request

**PR Link:** https://github.com/tinaudio/synth-setter/pull/1755

**PR Description:**

**What does this PR do?** Adds a single `log.debug` call in `train()` that emits the fully-resolved Hydra config immediately after seed setup and before any object is instantiated. Uses `OmegaConf.to_yaml(cfg, resolve=True)` so all `${...}` interpolations are expanded. Falls back to `resolve=False` if resolution fails (e.g. unresolvable internal Hydra keys in test fixtures). The call is gated by `log.debug`, so normal runs without debug logging enabled remain completely silent.

**Why was this PR needed?** Issue #33 identified that developers debugging training failures had no way to inspect the final composed Hydra config without inserting a manual `print()` statement. Running with `log_cli_level=DEBUG` should be sufficient.

**What are the relevant issue numbers?** Closes #33

**Does this PR meet the acceptance criteria?**
- [x] Tests added for new/changed behavior
- [x] All tests passing (`make test-fast`: 2022 passed, 5 skipped; 2 pre-existing failures unrelated to this change)
- [x] Follows project style guide (conventional commits, `# noqa: BLE001` pattern, `RankedLogger`)
- [x] No breaking changes introduced

**Maintainer Feedback:**
- *(awaiting review)*

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

The most concrete skill gained was understanding how to isolate a unit test from a complex fixture. The `cfg_train` fixture in this project includes `return_hydra_config=True`, which injects internal Hydra keys (`hydra.job.num`, `hydra.sweep.subdir`) set to `???`. Calling `OmegaConf.to_yaml(cfg, resolve=True)` on that fixture raises an exception — not because the production code is wrong, but because the test environment is missing runtime state that Hydra would normally supply. The fix was to mock `OmegaConf.to_yaml` in the train module rather than trying to pre-resolve the fixture, which is the right unit-test isolation boundary: test that the function is called with the right arguments, not that OmegaConf itself works correctly.

I also learned how to navigate a large, well-structured ML codebase. Reading `_log_model_artifact` to see how they handle wandb exceptions (the `# noqa: BLE001` pattern) told me exactly how to handle the fallback in the fix. The existing code was the best documentation.

### Challenges Overcome

The biggest challenge was the test fixture. Three iterations were needed before the test was correct:
1. Pre-computing `OmegaConf.to_yaml(cfg_train)` before calling `train()` failed because the fixture has unresolvable Hydra-internal interpolations.
2. Adding `run_name = "test"` to the fixture resolved one key but not `hydra.job.num`.
3. The final approach — mocking `OmegaConf.to_yaml` at the module level — was the right answer. It tests the contract (the function is called with `resolve=True` and the result is logged) without trying to exercise OmegaConf's resolution engine on a fixture it can't fully populate.

The rebase also required manual conflict resolution: the upstream `train.py` had grown significantly (adding `_derive_checkpoint_uri`, `_upload_best_checkpoint`, `_has_wandb_logger`) since the branch was originally cut. The patch had to be re-applied by hand to the new version of the file.

### What I'd Do Differently Next Time

Rebase onto upstream more frequently during development rather than once at submission time. The conflict arose because the branch drifted far from upstream while I was working. A daily `git fetch upstream && git rebase upstream/main` would have caught the divergence when the upstream changes were smaller and easier to reconcile.

I'd also check the test fixture's actual structure before writing the test — reading `conftest.py` first would have revealed the `return_hydra_config=True` flag immediately and saved two failed test iterations.

---

## Resources Used

- [Hydra documentation: Structured Configs](https://hydra.cc/docs/tutorials/structured_config/intro/) — for understanding `OmegaConf.to_yaml` and `resolve=True`
- [OmegaConf API reference](https://omegaconf.readthedocs.io/en/2.3_branch/usage.html#omegaconf-to-yaml) — confirmed `resolve=True` is the correct kwarg
- [tinaudio/synth-setter issue #33](https://github.com/tinaudio/synth-setter/issues/33) — issue thread where I introduced myself and confirmed the approach
- [GitHub docs: Creating a pull request from a fork](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request-from-a-fork)
