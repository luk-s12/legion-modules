---
type: <gate|generator>                    # REQUIRED, both types. Pick one — delete this comment once decided. `implementer` is not supported yet, don't use it.

# ──────────────────────────────────────────────────────────────────────────
# type: gate ONLY — delete this whole block if you're building a generator
# ──────────────────────────────────────────────────────────────────────────
valid_stages:                             # REQUIRED. Stages this module technically supports (one or more). Today Legion only defines `post-finalized` (runs after a story's implementer reports FINALIZED, before the reviewer's verdict) — if your module needs a different stage, that's a Legion-side extension, not something you invent unilaterally in this field.
  - <stage-name>
default_stage: <stage-name>               # REQUIRED. Used when a story references this module in `## Modules` without an explicit `@ stage`. Must be one of the values listed in valid_stages above.
default_activation: <opt-in|always>       # REQUIRED. opt-in = only runs on stories that list it under `## Modules`. always = runs on every story without the story mentioning it.
writes_to: <path/inside/worktree/>        # REQUIRED KEY — leave the value empty ("") if this module never writes anything. If non-empty, it MUST be a path *inside* the story's worktree; Legion validates this before launch and again after (git status) — any write outside it is treated as an isolation incident, not a normal finding.
blocking: <true|false>                    # REQUIRED. true = a rejection from this module can send the story back for correction, same authority as the adversarial reviewer for that specific finding. false = findings are informational only, folded into the reviewer's ADVISORY section — this module can never itself stop a story from reaching `finalized`.
max_rejection_rounds: 3                   # OPTIONAL. Caps how many times this module alone can reject the same story before Legion escalates to the user. Legion will clamp this down (never up) to the installing project's own `max_correction_rounds` if yours is higher.
max_concurrent: 1                         # OPTIONAL. How many stories can have this module running at once. If omitted, Legion defaults it to 1 when writes_to is non-empty (a writing module needs serialization) or unlimited when writes_to is empty (a read-only check has nothing to race over).
requires_local_config: false              # OPTIONAL. Set true only if your agent reads a local secret/config file the installing project must provide by hand (e.g. an API key) — Legion will check that `modules/installed/<name>/.env.<base-repo>.local` exists (existence only, it never reads the content) and warn the user if it's missing.

# ──────────────────────────────────────────────────────────────────────────
# type: generator ONLY — delete this whole block if you're building a gate
# ──────────────────────────────────────────────────────────────────────────
output: modules/output/<module-name>/<base-repo-name>/   # REQUIRED. Where your artifacts land. Keep this literal path shape — the `<base-repo-name>` segment is filled in automatically per installing project, don't hardcode a real project name here.
scope: <base-repo|worktree>               # REQUIRED. What this module reads from by default when invoked with no extra argument. base-repo = the installing project's single clone (workspace/<base-repo>). worktree = requires the caller to pass `worktree:<Story-ID>` explicitly (see run-module's contract) — pick base-repo unless your module has no sensible default target.

# ──────────────────────────────────────────────────────────────────────────
# Always required, both types
# ──────────────────────────────────────────────────────────────────────────
tools:                                     # REQUIRED. The exact tool list your agent needs — Legion NEVER extends this at launch time, so anything missing here that your agent tries to use will simply fail at runtime.
  - Read
  - Grep
  - Glob
  # - Write   # add this if writes_to (gate) or output (generator) is non-empty — you cannot write without it.
  # - Edit    # only if you need to modify existing files rather than just create new ones.
  # - Bash    # ⚠ read this repo's README ("Trust model") before adding this. It means your agent can read anything the host process can reach, not just writes_to/output — a disclosed risk every installer will see flagged in their preview. Don't add it if Write alone gets the job done.
agent_entrypoint: .claude/agents/<agent-name>.md   # REQUIRED. Path, relative to this module's own folder, to the agent file Legion actually launches. Its frontmatter `tools:` must match the list above — Legion cross-checks the two and flags any mismatch.
---

# Module: <module-name>

<One or two sentences: what this module does, and what it explicitly does NOT do (scope
boundaries matter more here than a feature list — the person installing this needs to know the
edges before trusting it with a worktree or a base repo).>
