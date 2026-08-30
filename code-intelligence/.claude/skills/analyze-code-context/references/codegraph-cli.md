# CodeGraph CLI — confirmed shape (`@colbymchenry/codegraph@1.5.0`, Windows)

This is the self-contained operational contract used by the installed module.

## Commands used by this module

| Command | Has `--json`/`-j` | Path argument | Notes |
|---|---|---|---|
| `status [-j] [path]` | yes | positional or `cwd` | Safe on an uninitialized dir: `exit 0`, `{"initialized": false, ...}` |
| `init [path]` | no | positional only | Idempotent-ish: re-running rebuilds, use only when `initialized: false` |
| `sync [path]` | no | positional only | Only when `status` reports pending changes |
| `impact <symbol> [-j] [-p path] [-d depth]` | yes | `-p`, never positional alongside `<symbol>` | `exit 1` + stderr if not initialized |
| `callers <symbol> [-j] [-p path]` | yes | same rule | |
| `callees <symbol> [-j] [-p path]` | yes | same rule | |
| `affected [files...] [-j] [-p path]` | yes | same rule | returns `affectedTests`, a list of test files, never a "ran" claim |
| `explore <query...> [-p path] [--max-files N]` | **no** | `-p` | **disabled by this MVP**: returns unbounded full source |

**Never pass the repository path as a second positional argument next to a symbol** — e.g.
`codegraph impact createUser /some/path` errors with `too many arguments for 'impact'`. Always use
`-p <path>` explicitly, or run with `cwd` already set to the repository.

## Freshness mapping (from `status --json`)

```
pendingChanges: { added, modified, removed }   # all zero + worktreeMismatch null + index.state "complete" → fresh
worktreeMismatch: null | <details>              # non-null → unknown (index built against a different checkout)
index.state: "complete" | other                 # anything else → unknown
initialized: false                              # → unknown, go index first
```

Do not use `git log -1` timestamps to guess freshness — `status --json`'s own fields are the
authoritative, provider-native signal (confirmed more reliable in the Fase 0B spike than a
git-based heuristic).

## Why `explore` is disabled

It returns Markdown followed by full source blocks and has no byte/token limit. `--max-files` does
not make the output fit `maxEstimatedTokens`, because even one file may be larger than the budget.
Do not invoke it. Find an anchor with bounded `Grep`/`Glob`, then use structured graph commands.

## Absence / errors

- CLI not installed: the `status`/`impact`/etc. commands simply fail to execute (command not
  found). Treat any such failure as "CodeGraph absent" and go to `onboarding.md`.
- Uninitialized directory: `status --json` still returns `exit 0` with `initialized: false` — use
  this, not a try/catch around `impact`, to detect "needs `init`" vs. "CLI absent".
- Unknown symbol: `impact`/`callers`/`callees` print `ℹ Symbol "..." not found` (not always inside
  the `--json` payload) — treat as "no relation found", never as "symbol doesn't exist in the
  codebase" (it may exist under a different exported name, alias, or in a language CodeGraph
  didn't index).

## Index location

`codegraph init <path>` always writes to `<path>/.codegraph/` — there is no supported external
cache location. Before every `init` or `sync`, require
`git -C <path> check-ignore -q --no-index -- .codegraph/.legion-ignore-probe` to succeed. Probing a
nonexistent child path under `--no-index` avoids a confirmed false positive: a `.gitignore` or
common `info/exclude` whose only content is a blank line with a CR line ending (CRLF or CR-only —
a plain LF blank line does not trigger it) makes `check-ignore -q -- .codegraph/` report `exit 0`
even though `.codegraph/` is not actually ignored. Otherwise do not mutate the index and use
fallback. The module never edits ignore rules itself.
