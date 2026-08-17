---
name: sync-api-collections
description: Validate and prepare an api-collections sync run, preserving the previous OpenAPI state for an endpoint and schema diff. Use only when api-collections is invoked with SYNC.
---

# Prepare a sync run

Require `<OUTPUT>/<base-repo-name>-openapi.yml`. If absent, stop and tell the caller to run
`/run-module api-collections` without `sync` first.

Search sibling project directories under `modules/output/api-collections/` for other
`*-openapi.yml` files. Update only the namespace matching the supplied `BASE_REPO` or `WORKTREE`.
List other namespaces as existing and untouched in the completion report; do not label them stale
without evidence. If the source cannot identify one unambiguous
namespace, stop and return the candidate names instead of choosing.

Read and retain the previous OpenAPI paths and schemas before overwriting. After generation,
compare old and new representations and report endpoints added, removed, and changed (method,
parameters, request, response, or security), plus schemas added, removed, and changed (properties,
required fields, or types).

Reuse saved format configuration unless explicit `FORMATS` changes it. `SYNC` alone never
causes a first-run format choice.
