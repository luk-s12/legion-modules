# Authoring modules for Legion

This guide contains the technical reference that is unnecessary when discovering or using a
module. Return to the [README](../../README.md) for the quick introduction.

## Start from the template

Copy [`_meta/template/module.md`](../template/module.md) into a top-level folder:

```text
<module-name>/
├── module.md
├── .claude/
│   ├── agents/<agent-name>.md
│   ├── skills/<skill-name>/SKILL.md      # optional
│   └── rules/                            # optional
│       └── module-rules.md
├── scripts/                              # optional
├── references/                           # optional
├── assets/                               # optional
└── <other required files>
```

Each folder is one independent, installable module. `.claude/rules/` is the Claude Code
convention; `provides_rules` targets one concrete file inside it, although it may declare another
internal path. Dependency manifests, schemas, fixtures, and
any resources used by the agent are also allowed: there is no closed list of subfolders. Every
path declared by `agent_entrypoint`, `provides_skills`, or `provides_rules` must remain inside the
module. Legion does not load the repository's internal `AGENTS.md` when running the agent.

A repository may contain one module (`module.md` at its root) or a collection of sibling modules
(`<subfolder>/module.md`, one level deep). `/new-module` discovers every module in a collection
and skips folders whose names begin with `_`.

## Choose the type

- `gate`: verifies a story at a declared stage. It may reject when `blocking: true`, but never
  replaces `worktree-reviewer`.
- `generator`: reads a base repo or worktree and creates artifacts on demand. It does not enter
  the story cycle or return verdicts.
- `implementer`: writes code in a story worktree. It runs only when a subtask names
  `[implementer:<module-name>]`.

## The `module.md` contract

Every type requires:

- `type`: `gate`, `generator`, or `implementer`.
- `tools`: explicit capability list.
- `agent_entrypoint`: relative path to the agent.

### `gate` fields

- `valid_stages` and `default_stage`.
- `default_activation`: `opt-in` or `always`.
- `writes_to`: worktree subpath, or `""` when read-only.
- `blocking`.
- Optional: `max_rejection_rounds`, `max_concurrent`, `requires_local_config`,
  `provides_skills`, and `provides_rules`.

### `generator` fields

- `output`: `modules/output/<module-name>/<base-repo-name>/`.
- `scope`: `base-repo` or `worktree`.

Gate fields, `provides_skills`, and `provides_rules` do not apply to generators.

### `implementer` fields

No additional fields are required. It may use `provides_skills` and `provides_rules`. It cannot
declare `writes_to`, stage fields, `blocking`, `max_concurrent`, or `always` activation. Write
access applies only to the worktree of the story that requested it.

The template documents each field and incompatibility.

## Write the agent

The agent frontmatter must declare the same tools as `module.md`. Legion cross-checks both lists
during installation.

Keep the entrypoint focused on:

1. Received inputs.
2. Decision flow.
3. Read and write boundaries.
4. Completion criteria.

Move checklists, formats, and variants into skills. Tell the agent exactly when to read each one.
A conditional skill should not load in runs that do not need it.

Internal agent skills differ from `provides_skills`:

- Internal skill: guides the module's own agent.
- `provides_skills`: guides a story implementer when that story activates the module.

## `provides_skills` and `provides_rules`

Available to `gate` and `implementer`:

```yaml
provides_skills:
  - .claude/skills/oop-practices/SKILL.md
provides_rules: .claude/rules/module-rules.md
```

Every path must resolve inside the module.

`provides_rules` points to Markdown entries with stable `rule_id` headings:

```markdown
### no-mutable-public-fields

Public fields must not be mutable.

### max-function-length

Split a function when it mixes more than one responsibility.
```

Legion negotiates these rules the first time a project activates the module. Verdicts are stored
per project; updating or renegotiating a rule does not reopen finalized stories.

## Autoconfig

When the template still contains `<...>` placeholders, `/new-module` offers to complete them:

1. Resolve the type first.
2. Infer `tools` from the agent when possible.
3. Ask for remaining values in small batches.
4. Write the resolved manifest into the installed copy.
5. Show the result in the preview before registration.

A real author-supplied value is never replaced as though it were a placeholder.

## Local validation

Install a module or collection from a Legion instance:

```text
/new-module <repo-or-path>
```

In a collection with two or more modules, each folder receives its own preview and installs as
`<repo>_<subfolder>`. Processing is sequential and ends with a summary of registered, discarded,
and skipped modules. To install only one from a local checkout, target the folder containing its
`module.md` directly.

Review the preview and test the relevant flow:

- `gate`: activate it through `## Modules` and verify stage, `writes_to`, and blocking behavior.
- `generator`: run `/run-module <name>` against the base repo and a worktree; both sources must
  remain unchanged.
- `implementer`: use it in a subtask; it must pass design and review like every other agent.
- With `provides_rules`: confirm negotiation happens once per project.

## Trust model

`/new-module` validates shape and performs a best-effort risk scan; it cannot prove intent.

- `Bash` allows commands with the host process's reach.
- `writes_to` and `output` state the write contract; they do not create a sandbox by themselves.
- Legion never widens tools at launch.
- An upstream contract change requires a new preview.
- A module must never commit, push, or alter git state.

## Lifecycle

- `/module uninstall <name>`: deprecates or removes once active references are clear.
- `/module activate <name>`: reactivates a deprecated module.
- `/module renegotiate <name> [rule_id]`: reopens rules for the current project.
- `/run-module <name>`: runs a generator on demand.

Before publishing, verify that declared permissions describe the module's complete real scope,
not merely the minimum needed to pass the preview.
