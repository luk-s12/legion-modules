---
type: generator

output: modules/output/api-collections/<base-repo-name>/   # namespacing added automatically by Legion
scope: base-repo                          # default target when invoked with no extra argument (workspace/<base-repo>); worktree:<Story-ID> is also supported by Legion's run-module contract without any extra field here

tools:
  - Read
  - Grep
  - Glob
  - Write
agent_entrypoint: .claude/agents/api-collection-generator.md
---

# Module: api-collections

Reads a project's code (the installing project's base repo, or a specific story's worktree) and
produces a standalone, functional OpenAPI/Swagger document plus the user-selected collections
derived from it for Postman and/or Insomnia. Every generated artifact is namespaced and prefixed
with the project's name so exports from different projects never collide.

**Explicitly does not**: execute the project or hit it over the network to introspect it (static
analysis only — see the agent's "What this module does not do"), modify the analyzed project in
any way, or guess request/response fields it didn't actually find in the code. It only writes
inside its own `output` folder, except for Legion's standard execution report under
`modules/reports/api-collections/`.
