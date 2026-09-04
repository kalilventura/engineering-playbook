---
name: documenting-api-endpoints
description: Use when creating, reviewing, or extending an API controller/endpoint handler in any language or framework — ensures the route, request shape, response shape, and every status code are documented as part of the endpoint, not left implicit in the code.
---

# Documenting API Endpoints

## Overview

The documentation of an endpoint IS the contract; the handler code is just
one implementation of it. Write/update the doc block for route, request,
response, and status codes as a required part of the change — not an
afterthought, not skippable because "the code is self-explanatory."

Framework-agnostic: applies whether you're annotating with OpenAPI/Swaggo
(Go), swagger-jsdoc/NestJS decorators (JS/TS), drf-spectacular/FastAPI
(Python), springdoc (Java), or plain markdown next to a handler that has no
doc-generation tooling at all.

## When to Use

- Creating a new API controller/endpoint handler.
- Reviewing a handler for an undocumented status code, param, or response shape.
- Extending an existing endpoint with a new param, response field, or error case.
- Reviewing the mapping between domain errors and HTTP status codes (see [[clean-architecture]]).

## The four required elements

Every endpoint doc must state, explicitly:

1. **Route** — method + path + version (e.g. `POST /v1/documents`)
2. **Request** — every param (path/query/body/header/form), each marked
   required or optional, with its type and any format constraint (uuid,
   date, enum values)
3. **Response** — the exact shape returned per status, including nested
   objects and pagination envelopes; not just "returns the object"
4. **Status codes** — every code the endpoint can actually return, and the
   condition that produces each one — not just the happy path

Missing any of the four means the doc is incomplete, regardless of how good
the code is.

## Status code discipline

Map each status to a real, distinct condition in the handler — don't reuse
one code for two different failures, don't invent one for a condition that
can't happen.

| Code | Use when |
|---|---|
| 200 | Successful read/action returning a body |
| 201 | Resource created — document the created shape |
| 204 | Successful action, no body (delete, etc.) |
| 400 | Malformed input (bad JSON, invalid format, missing required field) |
| 401 | Missing/invalid auth credentials |
| 403 | Authenticated but not permitted (map domain `ErrPermissionDenied`) |
| 404 | Referenced resource doesn't exist (map domain `ErrNotFound`) |
| 409 | Conflict with current state (duplicate, version mismatch) |
| 422 | Well-formed input, fails business validation |
| 500 | Unexpected/internal failure — last resort, not a catch-all for known cases |

If the handler has a `switch`/`errors.Is` chain mapping domain errors to
HTTP codes, every branch in that chain needs a matching `@Failure` /
`@Response` entry — the doc and the error-mapping code must stay in sync.

## Example (Go + swaggo, from a real handler)

```go
// Execute deletes a document from the database using its unique UUID.
//
// @Summary      Delete a Document by ID
// @Description  Deletes a document from the database using its unique UUID.
// @Tags         documents
// @Param        id   path      string  true  "The unique identifier (UUID) of the document" format(uuid)
// @Success      204
// @Failure      400  {object}  error "Invalid document id format"
// @Failure      403  {object}  error "Requester lacks permission to delete this document"
// @Failure      404  {object}  error "The document with the specified ID was not found"
// @Failure      500  {object}  error "Internal Server Error"
// @Router       /v1/documents/{id} [delete]
func (c *DeleteDocumentController) Execute(ctx echo.Context) error {
    // ...
    err := c.deleteDocument.Execute(ctx.Request().Context(), input)
    if err != nil {
        switch {
        case errors.Is(err, deletedocument.ErrPermissionDenied):
            return echo.NewHTTPError(http.StatusForbidden, err.Error())
        case errors.Is(err, deletedocument.ErrDocumentNotFound):
            return echo.NewHTTPError(http.StatusNotFound, err.Error())
        default:
            return echo.NewHTTPError(http.StatusInternalServerError, err.Error())
        }
    }
    return ctx.NoContent(http.StatusNoContent)
}
```

Port the same shape to other stacks: NestJS `@ApiOperation`/`@ApiResponse`,
FastAPI `responses={...}` + Pydantic models, Spring `@Operation`/`@ApiResponses`.
The annotation syntax changes; the four required elements don't.

## Quick Reference

| Situation | Do |
|---|---|
| New endpoint | Document route, request, response, and every status code before calling it done |
| Handler has a `switch`/`errors.Is` error-mapping chain | Every branch needs a matching `@Failure`/`@Response` entry |
| A field is required by validation | Mark it required in the doc, not just optional by omission |
| Only 200/201 are documented | Incomplete — document every code the handler can actually return |
| Response is a typed error DTO | Document that shape, not the language's built-in error type |

## Checklist for a new/changed endpoint

- [ ] Route documented (method, path, version)
- [ ] Every request param documented with required/optional + type/format
- [ ] Success response documented with its exact shape (not "returns data")
- [ ] Every status code the handler can return is documented, with the
      condition that triggers it
- [ ] Error-mapping code (`switch`/`errors.Is`/equivalent) and documented
      `@Failure`/`@Response` entries match 1:1
- [ ] No handler-only status code trap: a code the code can return but the
      doc omits, or vice versa

## Common Mistakes

- **Happy-path-only docs**: only 200/201 documented, error codes omitted —
  reviewers and API consumers can't tell what failure modes exist.
- **Generic response type**: `{object} error` for every failure instead of
  the actual shape returned — if the body is a typed error DTO, document
  that type, not the language's built-in error.
- **Undocumented required-ness**: a body field is required by validation
  but the doc doesn't mark it `true`/required.
- **Doc/code drift**: a new domain error added to the `switch` without a
  matching `@Failure` line, or a `@Failure` line for a code the code can no
  longer produce.
