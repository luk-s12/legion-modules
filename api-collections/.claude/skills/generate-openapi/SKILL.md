---
name: generate-openapi
description: Discover HTTP endpoints and models through static source analysis and produce the canonical self-contained OpenAPI document for the api-collections module. Use on every api-collections generation or sync run.
---

# Generate OpenAPI

## Resolve project identity and stack

Derive `<base-repo-name>` first from the final segment of `OUTPUT`, then cross-check it against
the source/project folder.
Use `STACK` when supplied; otherwise inspect the dependency manifest (`package.json`,
`pom.xml`, `build.gradle`, `pyproject.toml`, `requirements.txt`, `go.mod`, `Gemfile`, or
the project equivalent). Use the project manifest version when present, otherwise `0.0.0`.

## Discover the API

Prefer an existing OpenAPI/Swagger YAML or JSON specification. Also recognize generators such as
`springdoc-openapi`, `drf-yasg`, `drf-spectacular`, `@nestjs/swagger`, `tsoa`,
`flask-smorest`, `apispec`, and FastAPI. Normalize and complete that source instead of
rebuilding it.

Otherwise inspect routes and connected models using the applicable heuristic:

| Stack | Evidence to inspect |
|---|---|
| Express | `router`/`app` calls for GET, POST, PUT, PATCH, DELETE, OPTIONS, and HEAD; route files |
| NestJS | `@Controller`, HTTP decorators, DTOs and `class-validator` decorators |
| Spring | `@RestController`, mapping annotations, DTO/record classes |
| Django REST | `urls.py`, matching views and serializers |
| FastAPI | app/router HTTP decorators and referenced Pydantic models |

For an uncovered stack, do not guess. For each real endpoint collect method, complete path,
parameters, request type, response types/statuses, security, tags, and source evidence. Resolve
concrete model fields. Use `type: object` plus an explanatory description only when a field type
cannot be determined; never create unseen fields.

Apply `ONLY`, when supplied, to the discovered endpoint set by tag/path. Retain every schema
transitively referenced by the filtered endpoints.

## Write the canonical document

Write `<OUTPUT>/<base-repo-name>-openapi.yml` as OpenAPI 3.0.3 with project title/version, a
description stating that static analysis generated it, a detected server or
`http://localhost:3000`, every discovered operation with a response, shared concrete models in
`components.schemas`, and only evidenced security schemes.

If no endpoint is found, still write a valid document with `paths: {}`; state the coverage
failure in `info.description` and the completion report. Do not present it as complete.

## Validate by re-reading

Verify indentation, top-level structure, unique path+method pairs, a response for every operation,
and a defined target for every local `$ref`. The file must be self-contained.
