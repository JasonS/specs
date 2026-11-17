# Errors

Designed in the project thread "Design errors for every output", 2026-11-17. Builds on the per-method
errors Jason approved on 2026-11-16 and fills in the notation's Deferred "Errors" item, except unions
for custom method results, which stay deferred.

## Problem

No output says which errors a method can return. AIP sends a bare `Status`, REST a `default`
problem, JSON:API the HTTP errors of its own protocol, and GraphQL one shared `UserError` with no
code. A client can't tell "this book doesn't exist" from "this book is already archived" without
reading prose.

## Decision

The schema says which errors each method returns, as standard codes. Each output renders them in
its standard's own error format. Switching outputs never gains or loses an error.

### Codes

Google's canonical codes (google/rpc/code.proto), the ones a design can name. Auth codes
(`UNAUTHENTICATED`, `PERMISSION_DENIED`) come later.

| Code                  | Means                                                                |
| --------------------- | -------------------------------------------------------------------- |
| `INVALID_ARGUMENT`    | The request is wrong, whatever the state of the data                 |
| `NOT_FOUND`           | The resource, its owner, or a resource it names is gone              |
| `ALREADY_EXISTS`      | What the request would make already exists                           |
| `FAILED_PRECONDITION` | The data isn't in a state that allows this, such as an archived book |

### Standard methods: defaults, nothing to write

| Method   | Returns                                                                      |
| -------- | ---------------------------------------------------------------------------- |
| get      | `NOT_FOUND`                                                                  |
| batchGet | `NOT_FOUND` (AIP-231: the whole call fails when one is missing)              |
| list     | `INVALID_ARGUMENT`; `NOT_FOUND` when the resource has an `@owner`            |
| create   | `INVALID_ARGUMENT`; `NOT_FOUND` when it has an `@owner` or sends a reference |
| update   | `INVALID_ARGUMENT`, `NOT_FOUND`                                              |
| delete   | `NOT_FOUND`                                                                  |
| undelete | `NOT_FOUND`, `ALREADY_EXISTS` when it isn't deleted (AIP-164)                |

### Custom methods: the author opts in

```graphql
type Book @resource {
  bookId: ID!
  archive(bookId: ID!, reason: String): Book!
    @mutation
    @errors(codes: [NOT_FOUND, FAILED_PRECONDITION])
}
```

A custom method without `@errors` returns no listed errors. No default is added, because the author
knows what a custom method does and the engine doesn't (Jason, 2026-11-16).

The notation declares:

```graphql
enum ErrorCode {
  INVALID_ARGUMENT
  NOT_FOUND
  ALREADY_EXISTS
  FAILED_PRECONDITION
}
directive @errors(codes: [ErrorCode!]!) on FIELD_DEFINITION
```

`ErrorCode` joins the notation's taken names.

## What each output gets, for `archive`

**AIP over HTTP** (AIP-193). One response per status, from the shared table below, each in the
HTTP shape AIP-193 gives:

```json
{
  "error": {
    "code": 404,
    "message": "…",
    "status": "NOT_FOUND",
    "details": [
      {
        "@type": "type.googleapis.com/google.rpc.ErrorInfo",
        "reason": "…",
        "domain": "…"
      }
    ]
  }
}
```

The built-in `Status` schema becomes that shape, with `ErrorInfo` (AIP-193: every error carries
one) and `BadRequest` field violations for `INVALID_ARGUMENT` ("should use the standard detail
payloads"). Both names are reserved under AIP.

**gRPC.** Proto has no way to declare errors, so each rpc's comment lists its codes:

```proto
// Archives a book.
//
// Errors: NOT_FOUND, FAILED_PRECONDITION.
rpc ArchiveBook(ArchiveBookRequest) returns (Book)
```

**REST** (Zalando). Each operation lists its statuses as `application/problem+json`, except 404,
which stays with the `default` response (Zalando rule 151: document the errors specific to the
endpoint). The problem's `type` names the code, so two codes on one status stay apart (RFC 9457
3.1.1); the REST pull request picks the exact URI form from RFC 9457 and Zalando and shows it.

**JSON:API.** Each operation's error objects take `code` from its codes, as an enum (JSON:API 1.1,
Error Objects: "an application-specific error code"). `INVALID_ARGUMENT` is 422 on a request body
and 400 on query parameters (JSON:API 1.1, Creating Resources; Query Parameters). A body field's
error points at it with `source.pointer`.

**GraphQL** (Saleor, GitLab). Each mutation's payload has a typed error list with a code enum:

```graphql
type ArchiveBookPayload {
  book: Book
  errors: [ArchiveBookError!]!
}
type ArchiveBookError {
  field: [String!]
  message: String!
  code: ArchiveBookErrorCode!
}
enum ArchiveBookErrorCode {
  NOT_FOUND
  FAILED_PRECONDITION
}
```

Saleor names the list `errors`, each with `field`, `message` and a `code` enum (`AccountError`,
`AccountErrorCode`); GitLab names it `errors` too, as plain strings. Shopify's `userErrors` is the
same idea under another name, used by Shopify alone. A union of error types inside the payload (Hot Chocolate's
mutation conventions) was considered; no well-known public API was found using it, and Jason wants
idiomatic Relay (2026-11-17). It could come later as an option of the GraphQL output, once outputs
take options; nothing here should rule that out. Every mutation, standard or custom, gets its own `<Mutation>Error` and `<Mutation>ErrorCode`; the
shared `UserError` and the `userErrors` field go. A mutation with no codes has no `errors`.

Queries keep their results. A missing item is `null`, as GraphQL does. A custom `@query` with
`@errors` reports a code in the response's `errors` entry as `extensions.code` (GraphQL spec 7.1.2),
and its description lists the codes.

## HTTP statuses: one shared table

From google/rpc/code.proto's HTTP mapping, in the package the HTTP outputs already share
(`@muffin-top/openapi`). Core never knows HTTP. Each output overrides only what its standard says.

| Code                  | Status | Overrides                       |
| --------------------- | ------ | ------------------------------- |
| `INVALID_ARGUMENT`    | 400    | JSON:API: 422 on a request body |
| `NOT_FOUND`           | 404    |                                 |
| `ALREADY_EXISTS`      | 409    |                                 |
| `FAILED_PRECONDITION` | 400    |                                 |

## Rules

New rule `core/errors`:

| Severity | Message                                                      |
| -------- | ------------------------------------------------------------ |
| error    | @errors goes on a custom method, marked @query or @mutation. |
| error    | This code is already listed.                                 |
| error    | List at least one code, or leave @errors out.                |

A code that isn't an `ErrorCode` is `core/directive-uses` ("This value is not allowed for the
argument."), as today.

Reserved names change: AIP adds `ErrorInfo` and `BadRequest`; GraphQL drops `UserError` and adds
`<Mutation>Error` and `<Mutation>ErrorCode` for each mutation it makes.

## Engine

- Core exports `type ErrorCode` and gives each method its codes: `standardMethodOf` returns
  `errors`, decided from the resource as above; `CustomMethod` gains `errors`, as written.
- Each output's method type carries `errors: readonly ErrorCode[]` and renders them. No output
  decides which errors a method has.
- The status table lives in `@muffin-top/openapi`, which names no output; outputs pass their
  overrides.

## Delivery

Each its own pull request and review. Corpus expectations are written by hand from each standard,
never copied from engine output.

1. Notation and core: `ErrorCode`, `@errors`, `core/errors`, codes on every method; notation.md.
   Outputs render as before.
2. The shared status table, AIP over HTTP and gRPC.
3. REST.
4. JSON:API.
5. GraphQL. Jason sees the generated `archiveBook` before it merges.
