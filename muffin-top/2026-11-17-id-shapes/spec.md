# ID shapes

Approved by Jason on 2026-11-17, in the project chat.

## Problem

An `ID!` was always a bare string. Many APIs give their IDs a shape, such as a UUID or a prefixed
key like `sh_8f2k1m9x0q3z`, and a client can't tell that from the output. The validation
directives didn't apply to `ID`.

## Decision

`@format(as: UUID)` and `@pattern(regex:)` may go on a resource's `ID!` field. `@format` takes only
`UUID` there; `EMAIL` and `URI` stay String-only.

```graphql
type Shelf @resource {
  id: ID! @pattern(regex: "^sh_[a-z0-9]{12}$")
}

type Book @resource {
  id: ID! @format(as: UUID)
  shelf: Shelf! @owner
}
```

The shape is part of the ID, so it goes wherever the output sends that bare ID:

| Style           | Where the shape goes                                                                                                   |
| --------------- | ---------------------------------------------------------------------------------------------------------------------- |
| AIP HTTP        | Each path segment that names the ID, as `{book}` with `format: uuid`. `name` is a resource path, so it keeps no shape  |
| gRPC            | Nowhere: `name` is a resource path (AIP-122)                                                                           |
| REST            | Each `{book-id}` segment, the resource's `id`, and every reference to it, such as `shelf_id`                           |
| JSON:API        | Each `{bookId}` segment, the resource object's `id` member, its update's, and every identifier of it in a relationship |
| GraphQL (Relay) | Nowhere: a Relay ID is opaque and global (Global Object Identification), so `id` keeps no `@format` or `@pattern`      |

## Errors

A request that names an ID of another shape is wrong whatever the data holds, so it is
`INVALID_ARGUMENT`. Any method that names a shaped ID gets that code by default, added to the
codes it already has (see 2026-11-17-errors):

- a standard method that names the resource's own ID (get, batchGet, update, delete, undelete),
  when that resource's ID has a shape;
- any standard method, when an owner's ID in its path has a shape;
- a custom method on an item, in the same cases, even when its `@errors` doesn't list
  `INVALID_ARGUMENT`. Collection custom methods follow the owner rule only.

The schema decides the code, so every output gains it, including gRPC and GraphQL, where the shape
itself isn't written. Each output renders it as its errors already do.

## Rules

- `@pattern` applies to `String` and `ID`.
- `@format` applies to `String`, and to `ID` only with `as: UUID`.
- Anything else on an `ID` is the existing `core/field-directives` error, "This validation does not
  apply to the field's type." No new problem codes.

## Delivery

One pull request: the core rule and errors, every output, the notation and the corpus. Corpus
expectations are written by hand from each standard, never copied from engine output.

## Sources

- JSON Schema 2020-12 Validation, 7.3 (`format: uuid`) and 6.3.3 (`pattern`):
  <https://json-schema.org/draft/2020-12/json-schema-validation>
- AIP-122 Resource names: <https://google.aip.dev/122>
- google/rpc/code.proto, `INVALID_ARGUMENT`:
  <https://github.com/googleapis/googleapis/blob/master/google/rpc/code.proto>
- GraphQL Global Object Identification: <https://relay.dev/graphql/objectidentification.htm>
- JSON:API 1.1, identification: <https://jsonapi.org/format/#document-resource-object-identification>
