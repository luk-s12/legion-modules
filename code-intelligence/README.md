# code-intelligence

A Legion `type: gate` module that uses [CodeGraph](https://github.com/colbymchenry/codegraph)
(`@colbymchenry/codegraph` on npm) for structural code discovery. This directory is self-contained
because Legion installs only the module's subfolder.

## Prerequisite

CodeGraph is **never installed silently**. This module publishes the `allow-codegraph-install` rule
through Legion's native `provides_rules` contract. The first time a project activates the module,
Legion negotiates the rule and persists the verdict for that project:

- if accepted and the CLI is missing, the skill may install and verify only the pinned version;
- if rejected, it uses conventional fallback without installing or asking again on every run.

Revisit the decision with
`/module renegotiate code-intelligence allow-codegraph-install`. To install it manually:

```text
npm install -g @colbymchenry/codegraph@1.5.0
```

Rollback:

```text
npm uninstall --global @colbymchenry/codegraph
```

Version pinned by this MVP: **1.5.0**. Official source:
`https://github.com/colbymchenry/codegraph`, published on npm by the same author (`colbymchenry`).

## What it does

- **During implementation** (`provides_skills`): the `worktree-agent` for a story that activates
  this module (`## Modules` in the story) receives the `analyze-code-context` skill. It decides when
  CodeGraph is useful, applies a context budget, and always requires files to be read directly
  before editing; it never relies solely on CodeGraph output.
- **After `FINALIZED`** (`agent_entrypoint`, `post-finalized`): `code-context-auditor` compares the
  story's actual diff with the impact CodeGraph would have identified and writes a non-blocking
  **advisory** report (`blocking: false`) to the exact `REPORT` locator supplied by Legion. It never
  rejects a story in this MVP.

## Permissions and `Bash` risk

This module declares `Bash` to invoke the CodeGraph CLI with separate arguments. It never builds a
command by concatenating provider output. Like any module with `Bash`, the process may technically
read anything accessible to it, not only the files the module intends to use. This is a disclosed
risk, not a boundary enforced by a technical sandbox, and follows the same trust model as other
modules in this repository.

## Privacy

CodeGraph runs locally; this module never sends source code to an external service. CodeGraph
collects anonymous usage telemetry without code, paths, or names unless it is disabled with
`codegraph telemetry off`. The module does not change telemetry settings; that decision belongs to
the installing project.

## Tested platforms

Only **Windows** has been tested, using synthetic TypeScript and basic Java repositories.
Linux/macOS and a real Spring Boot repository remain unverified and are not supported by this MVP.

## Known MVP limitations

- `codegraph explore` has no `--json` output and may return complete files without respecting the
  context budget, so it is disabled. The skill uses bounded search and structured queries instead.
- The CodeGraph index (`.codegraph/`) always lives inside the analyzed repository/worktree; there is
  no external cache. The skill runs `init`/`sync` only when `.codegraph/` is already ignored.
  Otherwise, it falls back. It never changes ignore rules.
- Negotiation has no temporary "Not now" choice: acceptance or rejection is persisted per project.
  Use `/module renegotiate` to change the decision later; no Legion core modification is required.
- Like Legion itself, this module runs only within Claude Code. It depends on the environment's
  `Agent`, `SendMessage`, and Skills conventions and is not portable to another assistant without
  rewriting its launch mechanism.
