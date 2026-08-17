<p align="center">
  <img src="_meta/assets/legion-modules-l-wordmark.png" alt="LEGION MODULES" width="360">
</p>

<p align="center"><strong>English</strong> · <a href="README.es.md">Español</a></p>

# Legion Modules — official module collection for Legion

<img src="_meta/assets/legion-modules-l-mark.png" alt="L" width="18" align="absmiddle"> **EGION MODULES**
is a **monorepo of modules for [Legion](https://github.com/luk-s12/legion)**, the dynamic
multi-agent orchestration system. A **module** is a self-contained Claude Code project — one
agent, optionally with skills — that plugs into a Legion orchestration to verify a User Story
(`type: gate`), produce a standalone artifact from a base repo (`type: generator`), or write code
directly in a story's worktree as its author (`type: implementer`), without Legion ever having to
own or edit that code. Legion treats every module the same way it treats the destination project
itself: it clones it, reads it, and never touches it by hand.

Each module lives in its **own top-level folder** in this repo (e.g. `dummy-e2e/`,
`api-collections/`) — independent from the others, individually installable. There is no shared
runtime between modules: installing one never requires installing another.

## Table of contents

- [What a module is (and isn't)](#what-a-module-is-and-isnt)
- [Repository layout](#repository-layout)
- [Module types](#module-types)
- [The `module.md` manifest](#the-modulemd-manifest)
- [Skills and rules a module brings](#skills-and-rules-a-module-brings-provides_skillsprovides_rules)
- [`type: implementer`](#type-implementer--a-module-that-writes-code)
- [Creating a new module](#creating-a-new-module)
- [Autoconfig: installing straight from the template](#autoconfig-installing-straight-from-the-template)
- [Testing a module against a real Legion instance](#testing-a-module-against-a-real-legion-instance)
- [Trust model](#trust-model)
- [Lifecycle once installed](#lifecycle-once-installed)

## What a module is (and isn't)

A module is **not** a fork of Legion and does not modify `CLAUDE.md`, the orchestrator, or any
other worktree. It is an independent Claude Code project that Legion's `/new-module` command
clones into `modules/installed/<name>/` inside a Legion instance, validates against a manifest
contract, and from then on launches like any other subagent — with the same isolation rules as a
`worktree-agent`:

- A `type: gate` module only writes inside the worktree of the story that triggered it (or a
  declared subpath of it).
- A `type: generator` module only writes inside its declared `output` folder — never inside the
  base repo or a worktree it reads from.
- A `type: implementer` module has real `Write`/`Edit` over the **entire** worktree of the story
  that names it — the widest grant any module gets, and only ever because a story explicitly
  asked for it (`## Subtasks: [implementer:<name>]`), never picked automatically or activated by
  default across every story.
- None of the three ever talks to another worktree, another module, or writes directly to
  `.orchestrator/signals/`/`.orchestrator/announcements/` — findings go into
  `modules/reports/`, and only Legion's orchestrator decides whether something gets escalated.

## Repository layout

```
legion-modules/
├── README.md                        ← you are here
├── README.es.md                     ← Spanish version
├── _meta/                           # everything that isn't an installable module — branding, scaffolding
│   ├── assets/                       # logos/wordmarks used by the READMEs
│   └── template/                     # module.md skeleton to copy when starting a new module
└── <module-name>/                   # one folder per module, independently installable
    ├── module.md                     # manifest — see below (_meta/template/ currently only ships this file)
    ├── .claude/                      # agents/, optionally skills/ — see "Creating a new module" (exact shape may evolve)
    │   └── ...
    └── ...                            # any other files the agent's own logic needs
```

A module folder is exactly what ends up cloned into a Legion instance's
`modules/installed/<name>/` — nothing above the module's own folder in this repo is part of what
gets installed.

## Module types

Defined and enforced by Legion's own `modules/README.md` — this repo builds modules *against*
that contract, it doesn't define it independently:

| Type | Runs when | Can reject a story? | Writes to |
|---|---|---|---|
| `gate` | At a story's `valid_stages` (e.g. `post-finalized`), opt-in via a story's `## Modules` or `default_activation: always` | Yes, if `blocking: true` (same authority as the adversarial reviewer for that finding — never a replacement for it) | `writes_to`, a subpath of the story's worktree |
| `generator` | On demand, via `/run-module <name>` — outside the story cycle entirely | No — never produces a verdict | `output`, namespaced automatically per destination project |
| `implementer` | Only when a story names it explicitly (`## Subtasks: [implementer:<name>]`) — never auto-selected, `default_activation: always` is rejected outright for this `type` | No verdict of its own — its code goes through the same design gate and `worktree-reviewer` as any implementer | The **entire** worktree of the story that named it — no `writes_to` to narrow it |

A `gate` module never becomes the sole gate to `finalized` — Legion's `worktree-reviewer` always
has the final word, even when a `gate` module is `blocking: true`.

## The `module.md` manifest

Frontmatter YAML, `snake_case`, one comment per field — same convention Legion itself uses for
`.orchestrator/config.md`. Required fields depend on `type`:

- **Always**:
  - `type` — `gate` | `generator` | `implementer`.
  - `tools` — YAML list of declared tools (`Read`, `Grep`, `Glob`, `Bash`, `Write`, `Edit`;
    anything else gets flagged as an unknown risk in `/new-module`'s preview).
  - `agent_entrypoint` — path, relative to the module's own folder, to the agent Legion launches
    (e.g. `.claude/agents/my-agent.md`).
- **`type: gate`**, additionally:
  - `valid_stages` — list of supported stages; today Legion only has `post-finalized`
    implemented.
  - `default_stage` — one of the stages listed in `valid_stages`.
  - `default_activation` — `opt-in` (needs `## Modules` in the story) | `always`.
  - `writes_to` — path inside the worktree, or `""` (empty string) if it never writes. The key
    must always exist, even when its value is empty.
  - `blocking` — `true` | `false`.
  - Optional: `max_rejection_rounds` (integer), `max_concurrent` (integer),
    `requires_local_config` (`true` | `false`), `provides_skills` (list of paths),
    `provides_rules` (path) — see [Skills and rules a module brings](#skills-and-rules-a-module-brings-provides_skillsprovides_rules).
- **`type: generator`**, additionally:
  - `output` — path shaped as `modules/output/<module-name>/<base-repo-name>/` (the
    `<base-repo-name>` segment is filled in automatically by Legion).
  - `scope` — `base-repo` | `worktree`.
  - None of the `gate`-only fields apply — this includes `provides_skills`/`provides_rules`: a
    `generator` never enters the story cycle, so there's no consumer for either field.
    `/new-module` rejects them outright on this `type`.
- **`type: implementer`**, no additional required fields; optionally `provides_skills`/
  `provides_rules` (same format as `gate`). `writes_to`, `blocking`, `valid_stages`/
  `default_stage`, `max_rejection_rounds`, `max_concurrent` do **not** apply — `/new-module`
  rejects any of them being present. `default_activation` can only be `opt-in` on this `type`
  (`always` is rejected outright) — see [`type: implementer`](#type-implementer--a-module-that-writes-code).

### `type: gate` example

```yaml
---
type: gate                                # gate | generator | implementer
valid_stages:                             # stages this module technically supports (one or more)
  - post-finalized
default_stage: post-finalized             # used when the story doesn't specify a stage
default_activation: opt-in                # opt-in (needs ## Modules in the story) | always
tools:                                     # Legion NEVER extends this list when launching
  - Bash
  - Read
  - Grep
  - Glob
  - Write
agent_entrypoint: .claude/agents/e2e-runner.md
writes_to: e2e/                           # path INSIDE the worktree only (empty if it doesn't touch it)
max_rejection_rounds: 3                   # never looser than max_correction_rounds in the destination's config.md
max_concurrent: 1                         # default, if omitted: 1 if writes_to is non-empty, unlimited if empty
requires_local_config: true               # if true, /new-module checks for .env.<base-repo>.local (existence only)
blocking: true                            # true = can REJECTED like the reviewer; false = goes to reviewer's ADVISORY
provides_skills:                          # optional — paths inside the module's own clone, SKILL.md format
  - skills/oop-practices/SKILL.md
provides_rules: rules.md                  # optional — path inside the module's own clone, itemized with rule_id
---

# Module: e2e-runner
```

## Skills and rules a module brings (`provides_skills`/`provides_rules`)

A `type: gate` or `type: implementer` module can, besides its main function, contribute content
that the implementing agent reads directly — **only for the story that references the module**,
never fused into the installing project's own conventions, never applied to a story that doesn't
ask for it.

- **`provides_skills`** — procedural guidance (a checklist, a How-to), in the exact same format
  as `.claude/skills/<name>/SKILL.md`. Read wholesale by the implementing agent, same authority
  as Legion's own `QUALITY_GUIDE`.
- **`provides_rules`** — punctual, rule-shaped statements that CAN contradict a real rule of the
  installing project, so Legion negotiates them explicitly with the user the first time a story
  of that project references the module — **once per project**, never at `/new-module` install
  time (a module in this repo is a reusable library across many different projects over time, so
  install time is too early to know which project's rules it'll be checked against). What's
  accepted ends up embedded as real text in that story's design document, never merged into the
  installing project's own core rules.

Each path in `provides_skills` and the `provides_rules` path itself must resolve to a real file
inside the module's own folder — `/new-module` rejects the manifest outright if one doesn't.

### `rules.md` format (what `provides_rules` points to)

One entry per rule, each with a **stable `rule_id`** you define (Legion never invents one) plus
its statement:

```md
### no-mutable-public-fields
Public fields must never be mutable — expose a getter and a dedicated setter/mutator method
instead of a bare public field.

### max-function-length
A function longer than 40 lines must be split — extract named helpers for each logical step.
```

`rule_id` must be unique within the file — `/new-module` rejects a `rules.md` with duplicates or
an entry missing a `rule_id`.

### `type: generator` example

```yaml
---
type: generator
output: modules/output/<module-name>/<base-repo-name>/   # namespacing added automatically by Legion
scope: base-repo                                          # base-repo | worktree — what it runs against by default
tools:
  - Read
  - Grep
  - Glob
  - Write
agent_entrypoint: .claude/agents/api-collection-generator.md
---

# Module: api-collections
```

### `type: implementer` example

```yaml
---
type: implementer
tools:
  - Read
  - Grep
  - Glob
  - Write
  - Edit
agent_entrypoint: .claude/agents/code-quality-writer.md
provides_skills:                          # optional — same format as gate's
  - skills/oop-practices/SKILL.md
provides_rules: rules.md                  # optional — same format as gate's
---

# Module: code-quality-pro
```

## `type: implementer` — a module that writes code

Unlike `gate` (verifies, can reject) and `generator` (produces a standalone artifact), a
`type: implementer` module's own agent **writes code directly in a story's worktree**, as its
author — the same role a native Legion implementer plays, but sourced from your clone.

- **No `writes_to`** — there's no smaller zone to offer than the whole worktree, so this `type`
  doesn't declare one. Legion verifies isolation the other way around: a hard `git status`
  check against the base repo and any other *idle* worktree (any diff there is an incident), and
  a best-effort cross-reference against a sibling story's own events for any *actively running*
  worktree — a plain diff can't tell "the module escaped its worktree" apart from "a parallel
  story simply progressed," so that case is never auto-flagged, only logged for manual triage.
- **Never auto-selected** — only runs because a story explicitly names it:
  `## Subtasks: 1. [implementer:<module-name>] <description>`. `default_activation: always` is
  rejected at install time for this `type` on purpose: a third-party module with real write
  access never replaces the native implementer silently across every story.
- **Same design gate, no shortcuts** — your agent submits its own design proposal for its subtask
  and goes through the same barrier as any other implementer.
- **Reviewer, unchanged** — `worktree-reviewer` audits your code exactly like any other, against
  the approved design (which already has any `provides_rules` conflict pre-resolved).

## Creating a new module

**Start from [`_meta/template/`](_meta/template/)** — for now it ships only
[`_meta/template/module.md`](_meta/template/module.md): a fully commented manifest skeleton with both
`gate` and `generator` blocks inline, marked to delete whichever doesn't apply. Copy that file
into your module's folder and edit it — you don't need to reconstruct the manifest shape from the
reference below. The rest of a module's folder (`.claude/agents/`, optionally `.claude/skills/`)
you write from scratch, following the layout above.

```
mkdir my-module
cp _meta/template/module.md my-module/module.md
```

1. **Pick the type** — `gate` if the module should be able to verify (and possibly reject) a
   story; `generator` if it only reads code and produces a standalone artifact, with no verdict;
   `implementer` if it should write code directly in a story's worktree, as its author (see
   [`type: implementer`](#type-implementer--a-module-that-writes-code) — real `Write`/`Edit`
   access over a whole worktree, only ever run when a story explicitly names it).
2. **Create the folder** `<module-name>/` at the root of this repo (or copy `_meta/template/module.md`
   into it as shown above).
3. **Write `module.md`** following the format above — every field required for the declared
   `type` (see [The `module.md` manifest](#the-modulemd-manifest)). Left-over `<...>`
   placeholders from the template don't have to be resolved by hand: `/new-module` recognizes
   them and offers to fill them in interactively instead of rejecting the manifest outright — see
   [Autoconfig: installing straight from the template](#autoconfig-installing-straight-from-the-template).
4. **Write the agent** at the path declared in `agent_entrypoint` (`.claude/agents/<name>.md`),
   with a frontmatter `tools:` list that **matches** `module.md`'s `tools:` — `/new-module`
   cross-checks the two and flags any tool the agent actually uses but the manifest didn't
   declare. Keep the agent read-only where possible: for a `gate` module, only touch
   `writes_to`; for a `generator`, only touch `output`. Never assume access to `STORY`,
   `BRANCH`, or `EVENTS` unless Legion's prompt for that module type actually passes them (a
   `generator` invoked without `worktree:` gets neither).
5. **Add skills, if the agent needs a checklist or reference guide** to apply consistently
   (`.claude/skills/<name>/SKILL.md`) — same pattern Legion itself uses for
   `security-guide`/`data-guide` alongside its own specialist agents. Optional, not every module
   needs one.
6. **Keep the risk profile as low as the job allows.** `tools` outside
   `Read, Grep, Glob, Bash, Write, Edit` gets flagged as unknown by `/new-module`'s scan; `Bash`
   in particular means the module can read anything the host process can reach, regardless of
   `writes_to`/`output` — a disclosed, accepted risk, not something declared narrowly in
   `writes_to` can technically prevent. Don't reach for `Bash` (or an external CLI it would
   shell out to) if the agent can build the output itself with `Write`.
7. **Test it locally** against a real Legion instance before treating it as done — see next
   section.

## Autoconfig: installing straight from the template

Not every dev picking up `_meta/template/` knows Legion's manifest contract well enough to fill in
every field by hand on the first try — that's expected, not a blocker. If you point `/new-module`
at a module whose `module.md` still has unresolved `<...>` placeholders in required fields (or
`type` itself is still `<gate|generator>`), Legion doesn't reject it outright as an incomplete
manifest. It runs an **autoconfig assist** instead:

1. If `type` is still unresolved, it asks you first — nothing else can be inferred without it.
2. If `agent_entrypoint` already points at a real agent file, it reads that agent's own
   frontmatter `tools:` and pre-fills the manifest's `tools:` from it, so you're not typing the
   same list twice.
3. It asks whatever's left via `AskUserQuestion`, in small batches, with a sensible default
   pre-selected wherever one exists (`default_activation: opt-in`, `blocking: true`,
   `requires_local_config: false`, and so on) — confirming is usually enough, free text is only
   needed for values Legion truly can't guess (`writes_to`, `valid_stages`, `output`).
4. It writes the resolved values back into its own freshly cloned copy at
   `modules/installed/<name>/module.md` and continues the normal install flow from there — the
   risk preview you see afterward notes that the manifest was completed via autoconfig, so you
   know some of what you're approving wasn't authored as-is by the module.

This only ever fills in what's still a template placeholder — a field you deliberately left with
a real, non-placeholder value is never touched or second-guessed. And it's still not an exemption
from the contract: whatever you skip in the assist still needs a real value before the manifest
passes validation, same as any other incomplete `module.md`.

## Testing a module against a real Legion instance

A module only proves itself by actually going through Legion's install and run path — reading
the manifest back to yourself isn't a test.

1. In a Legion instance (the orchestrator repo, not this one), run `/new-module <path-to-this-repo>/<module-name>` pointing at your module's local folder. This runs the full validation
   Legion applies to any module: required-fields check, `tools` classification, the
   manifest-vs-agent cross-check, a best-effort network scan, a dependency-vulnerability scan if
   a package manifest is present — then shows you the full preview before registering anything.
2. Accept the preview (`modules/pending/<module-name>.md` in that Legion instance) to register
   it in `modules/registry.md` as `installed`.
3. **For `type: gate`**: run `/legion` on a story that lists your module under `## Modules`
   (`opt-in`) or set `default_activation: always` and run any story. Confirm it launches at the
   `stage` you declared, writes only inside its `writes_to` subpath, and that a deliberate
   rejection (`blocking: true`) actually stops the story from reaching `finalized` until fixed.
4. **For `type: generator`**: run `/run-module <module-name>` against the base repo, then again
   with `worktree:<Story-ID>` against a `finalized` story. Confirm the artifact lands under
   `modules/output/<module-name>/<base-repo-name>/` and that `workspace/<base-repo>` (or the
   worktree) comes out of the run with no unexpected changes (`git status --porcelain` before and
   after).
5. **For `type: implementer`**: write a story with `## Subtasks: 1. [implementer:<module-name>]
   <description>` and run `/legion` on it. Confirm your agent's `DESIGN_PROPOSED` goes through
   the same gate as any implementer, that it only ever touches its own assigned worktree, and
   that `worktree-reviewer` audits its code like any other story's.
6. If you declared `provides_rules`, confirm the first story that references the module against a
   given project triggers the one-time negotiation (`AskUserQuestion`), and that a second story of
   the same project doesn't ask again.
7. Once it behaves as intended, it's ready to be proposed for inclusion in this repo.

## Trust model

`/new-module` validates **form, not intent** — the same posture this repo's own modules are
built under. A clean risk preview is not a guarantee of safety, and no field in `module.md` is a
technical sandbox:

- `tools: [Bash]` means arbitrary command execution scoped to the host process, not to
  `writes_to`/`output`.
- Network and dependency scans are pattern-matching / manifest-detection, not data-flow analysis
  — they exist to put real information in front of whoever installs the module, not to gatekeep
  automatically.
- A module never widens its own `tools` at runtime — the installing Legion instance never
  extends what was validated at install time.

Building a module for this repo means writing it so that its declared `tools`/`writes_to`/
`output` are the *true* full extent of what it needs — not the minimum that passes review.
`type: implementer` is the widest grant of the three (`Write`/`Edit` over a whole worktree, no
`writes_to` to narrow it) — hold it to the same standard: the design gate and
`worktree-reviewer` are the real defense, same as for any implementer, native or not.

## Lifecycle once installed

Once a module from this repo is installed into a Legion instance, its lifecycle is managed
entirely by that instance, not by this repo:

- `/module uninstall <name>` / `/module activate <name>` — deprecate, reactivate, or remove it.
- `/module renegotiate <name> [rule_id]` — reopens a `provides_rules` accept/reject verdict for
  the current project, without waiting for a version change.
- A version/contract check runs automatically before every launch (`git fetch` on the installed
  clone); if `tools`/`writes_to`/`output`/`type` changed upstream (i.e. in this repo), the
  installing instance re-runs the full risk-preview flow before letting the updated code run
  again. If `rules.md` changed, only the `rule_id`s that are new or reworded get re-negotiated per
  project — the rest keep their existing verdict.

That means a change here — tightening `tools`, adding a field, fixing a bug in the agent — is
picked up safely by every Legion instance that has the module installed, without silently
widening what it's allowed to do.
