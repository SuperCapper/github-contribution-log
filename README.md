# github-contribution-log

# Contribution #1: Add debug logging of resolved config in train()

**Contribution Number:** 1  
**Student:** [David Jones]  
**Issue:** https://github.com/tinaudio/synth-setter/issues/33

**Status:** Phase I — In Progress

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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

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
