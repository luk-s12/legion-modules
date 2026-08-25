# Fallback — order and rules

Follow this order, stopping at the first step that gives you enough to proceed:

1. CodeGraph fresh and answered the query → use it.
2. CodeGraph `stale`/`unknown` but still answered → use it only as supporting evidence, with a
   `warnings` entry saying the
   index may not reflect the current worktree, and still read files directly (step 8 of the main
   skill applies regardless).
3. Targeted search by known symbol/file names (the ones already mentioned in the task, or found via
   `Grep`/`Glob` on obvious identifiers) — narrower than a full repository walk.
4. Conventional bounded exploration — the usual `grep`/directory-walk approach, but still scoped to
   what the task actually needs, not an open-ended tour of the repository.
5. If a material ambiguity remains after all of the above (you genuinely cannot tell which of two
   plausible components is the right one to touch), stop and report the ambiguity instead of
   guessing — this is a decision for whoever asked for the task, not something to resolve by
   picking one silently.

## Rules

- Always set `fallback.used` and `fallback.reason` in the Context Pack, even when fallback was
  never triggered (`false`/`null`).
- An empty or error result from CodeGraph is never evidence that a relation doesn't exist — it's
  evidence you didn't find one with this tool. Say so in `warnings`, don't claim the dependency is
  absent.
- CodeGraph being absent, uninitialized-but-installable, or answering nothing useful are three
  different situations — don't collapse them into one generic "fallback used" without recording
  which one happened in `fallback.reason`.
- Use stable reasons where applicable: `cli_missing`, `version_mismatch`,
  `installation_not_authorized`, `npm_unavailable`, `installation_failed`,
  `index_path_not_ignored`, `index_unknown`, `timeout`, `malformed_output`,
  `reference_integrity_mismatch`, or `insufficient_evidence`.
- `npm_unavailable`: the CLI was missing and installation was authorized, but the required
  `npm --version` preflight was missing, timed out, or exited unsuccessfully. Do not run the install
  command. This is distinct from `installation_failed`, which means npm passed preflight and the
  subsequent pinned install command failed.
- `version_mismatch`: a `codegraph` binary is present and runs, but its `--version` does not match
  `pinnedCodegraphVersion` in `SKILL.md`. Distinct from `cli_missing` (nothing runs at all) so the
  report doesn't imply the CLI is absent when it's actually just the wrong version. Never
  auto-install over a mismatched binary under the current missing-CLI-only authorization; report
  the expected and observed versions and use conventional discovery.
