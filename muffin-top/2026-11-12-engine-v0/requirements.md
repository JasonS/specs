# Engine v0

## Problem and objective

The graphqall notation ([docs/notation.md](https://github.com/JasonS/graphQall/blob/main/docs/notation.md))
defines how GraphQL SDL describes resources and how each API style renders them. Nothing implements
it yet. Engine v0 is a TypeScript library that turns SDL written in the notation into a
Google-style resource-oriented HTTP API: the design as data, plus an OpenAPI file. A CLI and a
playground will later wrap this library.

The notation defines the rules; these requirements state which of them v0 must implement and how
v0 proves it.

## Requirements

1. The library takes SDL text and a target, and returns the resource graph, the design for the
   target, the output files and the problems, as described in the notation's "Engine inputs and
   outputs".
2. The library runs in a browser and in Node. It uses no filesystem, network or Node-only API.
3. v0 supports one target, the Google-style HTTP API: the AIP design, written as one OpenAPI 3.1
   file. The OpenAPI file follows Google's own HTTP and JSON mapping for AIP APIs (proto3 JSON and
   `google.api.http`). For example, a `Date` field is `{ "year": 2026, "month": 11, "day": 12 }`,
   and a book's path is `/v1/publishers/{publisher}/books/{book}`.
4. A caller can add a new output format for an existing design style without changing the
   library's source, and select it as a target.
   - Acceptance: code outside the library adds a stub format after the AIP design, selects it, and
     receives the stub's output for that design.
5. v0 implements every rule in the notation that applies to the Google HTTP target: types,
   resources, identity, timestamps, ownership, plurals, operations, fields, create and update
   inputs, validation, comments, references, custom methods, paging, filtering, sorting, other
   directives, the AIP style conventions, deviations, and problems.
6. `@subset`, `@expandable` and `@number` have no effect on the Google HTTP target. Each produces a
   warning and changes nothing in the OpenAPI file. The notation's validity rules for them, such as
   a `@subset` that leaves out the `ID!` field, are still errors under every target.
7. A directive declared in the SDL that the notation doesn't define is kept, with its arguments, in
   the resource graph, so a later GraphQL format can write it out. The Google HTTP target ignores
   it with a warning. An undeclared directive is an error.
8. v0 defines one problem code for each error and warning the notation describes. The list of
   codes is added to the notation's Problems section, with the owner's approval.
9. Every rule, directive, argument and problem code has at least one test. Each expected result
   comes from the notation or is written by hand, never from running the engine. The tests cover
   at least:
   - every `@resource` argument and value: `get: NONE | MANY`, `list: NONE`, `create: NONE`,
     `update: NONE`, `delete: NONE | SOFT`, `plural`
   - every `@input` combination, and `!` on create and on update
   - every validation directive and argument, on create and on update, including each error
   - `"""descriptions"""` appearing in the OpenAPI file, and `# comments` appearing nowhere
   - custom methods on one item, on the collection, and with `readOnly: true`
   - references to one resource and to a list of resources
   - paging on every list, `@filterBy` and `@sortBy`, and a list with neither
   - ownership chains, an ownership loop, and nesting deeper than 3 levels
   - `DateTime`, `Date` and `LocalTime`
   - each problem code, checking its kind, code, severity and `where` span
10. The OpenAPI file passes an OpenAPI 3.1 validator with no errors. Tests compare the parsed file
    with hand-written expected files for structural equality; key order isn't compared.
11. When one type has an error, the design is still returned for every type that doesn't depend
    on it through ownership or a reference. Types that depend on it are left out.
    - Acceptance: `Publisher` and `Author` are valid and `Book` has two `ID!` fields. The result
      has the design for `Publisher` and `Author`, and a problem located in `Book`.
    - Acceptance: adding a valid `Chapter` owned by that `Book` leaves `Chapter` out too, with a
      problem saying why.
12. A syntax error in one definition doesn't stop the others. The engine still returns the design
    for every definition that parses and doesn't depend on the broken one.
    - Acceptance: SDL with valid `Publisher` and `Author` resources and `type Book @resource { titl`
      between them returns the design for `Publisher` and `Author`, and a syntax problem located in
      `Book`.

## Scope boundaries

- The CLI and the playground are separate products with their own requirements.
- The classic REST, GraphQL and Google gRPC targets come later.
- Holding back problems while someone is still typing is the editor's job; the engine doesn't do it.
- The notation's deferred items (overrides, client-chosen IDs) are not in v0.
