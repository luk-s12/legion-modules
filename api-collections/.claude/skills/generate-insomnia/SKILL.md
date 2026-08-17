---
name: generate-insomnia
description: Transform the canonical OpenAPI document produced by api-collections into an Insomnia v4 export. Use only when the resolved formats include insomnia.
---

# Generate the Insomnia artifact

Read `<OUTPUT>/<base-repo-name>-openapi.yml`; do not inspect source code again.

Write `<OUTPUT>/<base-repo-name>-insomnia-collection.yml` as an Insomnia v4 export with
`_type: export` and a `resources` list. Include a base environment with `base_url`, request
groups based on OpenAPI tags/resources when available, and one request per path+method. Translate
parameters, bodies, and evidenced security schemes. Placeholder body values must come only from
referenced schema properties.

Use stable, unique resource identifiers and correct parent relationships. Re-read the YAML and
verify indentation, one request per OpenAPI operation, method/path mappings, no orphaned parent
IDs, and a defined environment variable for every variable reference.

