---
type: gate
legion_contract: 2

valid_stages:
  - post-finalized
default_stage: post-finalized
default_activation: opt-in
writes_to: ""
blocking: false
requires_local_config: false

provides_skills:
  - .claude/skills/analyze-code-context/SKILL.md
provides_rules: .claude/rules/module-rules.md

tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Write

agent_entrypoint: .claude/agents/code-context-auditor.md
---

# Module: code-intelligence

Gives a `worktree-agent` (or specialist) access to
[CodeGraph](https://github.com/colbymchenry/codegraph) for structural discovery during story
implementation through the read-before-edit `analyze-code-context` skill. At `post-finalized`, it
performs a non-blocking (`blocking: false`) audit comparing the story's actual impact with what
CodeGraph would have identified.

**Explicitly out of scope:** replacing an LSP, running tests, modifying story code, installing
CodeGraph before Legion obtains user consent, or blocking a story in this MVP. Any finding is folded
into the `worktree-reviewer`'s `ADVISORY`. The auditor writes nothing inside the worktree
(`writes_to: ""`) and uses the exact `REPORT` locator supplied by Legion without reconstructing
paths.

See `README.md` for installation consent through `provides_rules` and known limitations, including
Windows-only validation and disabled `codegraph explore`.
