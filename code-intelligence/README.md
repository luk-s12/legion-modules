# code-intelligence

A Legion `type: gate` module that uses [CodeGraph](https://github.com/colbymchenry/codegraph)
(`@colbymchenry/codegraph` on npm) for structural code discovery. This directory is self-contained
because Legion installs only the module's subfolder.

## Requirements for full use

- Install and register the module in Legion. Activate it with the exact installed name shown by the
  installation summary or `modules/registry.md`; do not assume it is `code-intelligence`. A
  collection install may register it as `legion-modules_code-intelligence`.
- Run it through Legion in Claude Code.
- Make Node.js and npm available on `PATH`. Automatic installation additionally requires npm
  registry/network access and permission to write to npm's global installation location.
- Either install exactly CodeGraph `1.5.0` beforehand or accept the negotiated
  `allow-codegraph-install` rule. A different installed version is not replaced automatically.
- Ensure `.codegraph/` is already ignored by the target repository before activation. Add this to
  the repository's `.gitignore` through the project's normal review process:

  ```gitignore
  .codegraph/
  ```

  Verify it from the repository root, probing a nonexistent child path with `--no-index` (a
  `.gitignore`/`info/exclude` containing only a CR-terminated blank line otherwise makes
  `check-ignore -q -- .codegraph/` itself falsely report the directory as ignored):

  ```text
  git check-ignore -q --no-index -- .codegraph/.legion-ignore-probe
  ```

- Activate the installed module in each story that should use it, replacing the placeholder with
  the exact registered name:

  ```markdown
  ## Modules
  - <installed-name>
  ```

When a prerequisite is absent, the module remains non-blocking and falls back to conventional,
bounded code discovery; CodeGraph-backed impact and affected-test evidence will be unavailable.
See [configuration](docs/configuration.md), [architecture](docs/architecture.md), and
[troubleshooting](docs/troubleshooting.md) for the detailed contracts and diagnostics.

## CodeGraph installation behavior

CodeGraph is **never installed silently**. This module publishes the `allow-codegraph-install` rule
through Legion's native `provides_rules` contract. The first time a project activates the module,
Legion negotiates the rule and persists the verdict for that project:

- if accepted and the CLI is missing, the skill first requires a successful `npm --version`
  preflight, then may install and verify only the pinned version;
- if rejected, it uses conventional fallback without installing or asking again on every run.

If a different CodeGraph version is already present, the module does not replace, upgrade, or
downgrade it automatically, even when the missing-CLI installation rule was accepted. It reports
`version_mismatch` and uses conventional discovery. The project owner must resolve the global
installation manually, or explicitly expand and renegotiate authorization before a future run.

If npm is unavailable, times out, or fails its version preflight, the module records
`npm_unavailable` and does not attempt installation. `installation_failed` is reserved for a pinned
install attempted after a successful npm preflight that then fails.

Revisit the decision with the exact registered name established above:
`/module renegotiate <installed-name> allow-codegraph-install`. To install it manually:

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

## `REPORT` write permission

`writes_to: ""` is correct: this module never writes inside the story `WORKTREE`
(`code-context-auditor.md` states it explicitly — never write to `WORKTREE`). `Write` is still
declared in `tools` because the auditor writes its advisory finding to the opaque `REPORT` locator
Legion supplies, which lives outside the worktree by design. `/new-module`'s generic
`writes_to`/`tools` consistency check does not distinguish this case — it flags any non-empty
`Write` against an empty `writes_to` as a risk item — so installing this module will still show that
finding in the preview. This section explains why the finding does not indicate a real
inconsistency; it does not remove or silence it.

## Privacy

CodeGraph runs locally; this module never sends source code to an external service. CodeGraph
collects anonymous usage telemetry without code, paths, or names unless it is disabled with
`codegraph telemetry off`. The module does not change telemetry settings; that decision belongs to
the installing project.

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
