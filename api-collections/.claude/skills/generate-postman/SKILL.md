---
name: generate-postman
description: Transform the canonical OpenAPI document produced by api-collections into a Postman Collection v2.1 and matching environment. Use only when the resolved formats include postman.
---

# Generate Postman artifacts

Read `<OUTPUT>/<base-repo-name>-openapi.yml`; do not inspect source code again.

Write `<base-repo-name>-postman-collection.json` and
`<base-repo-name>-postman-environment.json` inside `OUTPUT`.

Use Postman Collection Format v2.1. Create one request per OpenAPI path+method, grouped by tag or
resource when available. Convert `{id}` path parameters to `:id` and declare URL variables.
Translate query/header parameters and security requirements. Build JSON body placeholders only
from referenced schema properties; never fabricate realistic-looking data.

The environment must contain `base_url` from the first OpenAPI server and placeholder variables
for evidenced security schemes. Requests must reference those variables consistently.

Re-read both files and verify valid JSON, the v2.1 schema URL, one request per OpenAPI operation,
valid method/path mappings, and that every referenced environment variable is defined.

