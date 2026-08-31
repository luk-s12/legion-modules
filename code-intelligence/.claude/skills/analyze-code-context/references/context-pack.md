# Context Pack — shape and rules

Report exactly this shape (as the structured summary you hand back to whoever asked you to explore
— not necessarily a file on disk, unless you are `code-context-auditor` writing its own report):

```yaml
contextPackVersion: "2"
provider:
  name: codegraph
  expectedVersion: "<pinnedCodegraphVersion from SKILL.md>"
  version: "<exact normalized output from `codegraph --version`, or \"unknown\" if unavailable>"
repository:
  root: "<normalized absolute path>"
  checkedRevision: "<git rev-parse HEAD, or \"unknown\">"
  workingTreeDirty: true|false          # from `git status --porcelain`
index:
  status: fresh|stale|unknown           # see references/codegraph-cli.md mapping
  indexedRevision: "<sha-or-unknown>"
  syncCompletedAt: "<timestamp-or-null>"
  checkedAt: "<timestamp of this query>"
query:
  task: "<the original task description you were given>"
  budgetApplied: true|false
  limits:
    maxFiles: 12
    maxSymbols: 20
    maxDepth: 4
    maxEstimatedTokens: 6000
onboarding:
  codegraphPresentInitially: true|false
  installRule: accepted|not_accepted|not_needed
  installationResult: success|failed|not_attempted
summary:
  entryPoints: []
  primarySymbols: []
  relatedFlows: []
files:
  inspectBeforeChange: []
  likelyToModify: []
  supportingContext: []
impact:
  direct: []
  transitive: []
  risk: low|medium|high|unknown
tests:
  direct: []
  related: []
  gaps: []
evidence: []
warnings: []
fallback:
  used: false
  reason: null
usage:
  graphQueries: 0
  graphCandidateFiles: 0
  syncAttempted: false
  syncSucceeded: false
  fallbackReason: null
```

## Rules

- Deduplicate and sort file paths and symbol names deterministically (alphabetical) — same input
  must produce the same order every time.
- Separate observed evidence (`evidence: []`, cite the exact `codegraph` command and field it came
  from) from your own recommendation (`summary`/`files`/`impact`) — never blend them.
- Mark heuristic or unverified relations explicitly in `warnings`, never silently upgrade a guess
  to a fact.
- Never include full file contents in this pack, even if `codegraph explore` handed you full
  source — recap only file paths, symbol names, and short (one-line) reasons.
- If the budget truncated results, set `query.budgetApplied: true` and note what was cut in
  `warnings` — never truncate silently.
- Prioritize by proximity to `query.task` when trimming to the budget, not by discovery order.
- `index.status` is never assumed `fresh` — only set it from the mapping in
  `references/codegraph-cli.md`.
- `provider.expectedVersion` is always the skill's pin. `provider.version` comes only from
  `codegraph --version`, never from `status --json`; keep it `unknown` when that command is absent
  or produces no usable output.
- A missing or mismatched CLI does not require a status call. Record the expected/observed version,
  set the precise fallback reason, and do not run `status` or another CodeGraph command.
- `fallback.used`/`fallback.reason` are always present, even when fallback never triggered
  (`false`/`null`).
- Never expose Legion's negotiation store. Record only whether the accepted
  `allow-codegraph-install` rule was supplied for this story.
- If the CLI is missing and the accepted rule is absent, use `installRule: not_accepted` and
  fallback reason `installation_not_authorized`; never claim why the rule is absent.
- `usage` counters are observed, not estimated: increment `graphQueries` once per structural
  command actually executed (`impact`/`callers`/`callees`/`affected`), set `graphCandidateFiles` to
  the count of distinct files that ended up in `files.likelyToModify`/`files.supportingContext`
  derived from graph evidence, and set `syncAttempted`/`syncSucceeded` only when this run actually
  attempted a `sync` (never infer them from freshness alone). `fallbackReason` mirrors
  `fallback.reason` when `fallback.used` is true, otherwise stays `null`. Do not report token counts,
  durations, or `Read` calls here — v2 cannot observe those reliably.
