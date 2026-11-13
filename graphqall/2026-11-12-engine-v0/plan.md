# Engine v0 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `@graphqall/core` and `@graphqall/style-aip` so that
`render(sdl, googleHttpTarget(options))` turns SDL written in the graphqall notation into the
resource graph, the AIP design, one OpenAPI 3.1 file and the problems.

**Architecture:** A chain of small immutable steps in core (split blocks, parse, build the graph
with one check function per rule, remove broken types), then a target: the AIP design style followed
by the AIP OpenAPI format. Checks return neverthrow `Result`s; each step returns an `Outcome` (value
plus problems). Method builders come from a factory keyed by a tagged `MethodChoice`.

**Tech Stack:** TypeScript 6, Node 26, pnpm 12 workspace, Vitest 5, `graphql`, `neverthrow`,
`pluralize`, `openapi3-ts`, `@seriousme/openapi-schema-validator` (tests only).

**Spec:** `../specs/graphqall/2026-11-12-engine-v0/spec.md`, with `requirements.md` beside it. The
notation is `docs/notation.md` in the graphqall repository. Read all three before any task.

## Global Constraints

- Every type, name, code, path and mapping in the spec is used exactly as written there. Where this
  plan and the spec differ, the spec wins; report the difference.
- Dependencies, pinned to exact versions: `graphql` 17.1.0, `neverthrow` 8.2.0, `pluralize` 8.0.0,
  `@types/pluralize` 0.0.33, `openapi3-ts` 4.6.2; dev only: `@seriousme/openapi-schema-validator`
  2.12.0. No other dependency (AGENTS.md: ask first).
- `packages/*/src` uses no Node, DOM, filesystem or network API (R2). Tests may use Node.
- Data shapes use `type`; `interface` only for the `I`-prefixed contracts (`IDesignStyle`,
  `IFileFormat`, `ITarget`, `IMethodBuilder`). Every field is `readonly`.
- No `try`/`catch` except the two the spec names: graphql-js syntax errors in `parseBlocks` and in
  the `@subset` selection parse. Each rethrows anything that is not a `GraphQLError` (TS-01 rule 3).
- No class `extends` anything (TS-04). Factories return the interface type (TS-03 rule 2).
- Tests: Vitest, no snapshots, no `vi.mock`, no `retry` (TS-07). A failure test asserts `kind`,
  `code`, `severity`, `where` and `path`, never `message`. Expected values are written by hand from
  the notation and the spec, never copied from engine output (P-07 rule 1).
- Prerequisite: the checks copied from naimo for P-01, P-03, P-04 and P-07 (ESLint rules, including
  `foundation/require-test-factories`, and `pnpm mutation`) are on `main` before Task 1. This plan
  adds no lint rule of its own.
- Test data (P-07, TS-07): a value of a type declared in a package's `src` that a test hands to the
  code under test is built by that type's `make<Type>(overrides)` factory in the package's
  `tests/factories.ts`. The factory owns every default and returns a new object on each call; the
  test overrides only the facts its case depends on. One factory per type; a task that adds a type
  tests pass in adds its factory. Expected values are plain literals written inside `toEqual` or
  `toMatchObject`, never in a variable or `it.each` table typed with a `src` type. SDL input stays
  written out as text.
- After a task's tests pass, `pnpm mutation` reports every mutant on the task's changed lines as
  caught. A surviving mutant gets a test; a mutant that can't change behavior goes to the owner.
- A defect found while building gets a failing test first, run red, then the fix.
- Spans are 1-based; `endColumn` is the column just after the last character. A span on one token
  `Book` that starts at column 6 is `{ line, column: 6, endLine: line, endColumn: 10 }`.
- Each task's tests are written, or reviewed before the task's code is written, by an agent that
  hasn't read the implementation, working from the notation, the requirements and the spec (P-07
  rule 2; spec, Testing).
- Work on one branch, `engine-v0`, one commit per task. Run `pnpm check` before opening the pull
  request.

## Review Focus

1. **Windows line endings.** SDL with `\r\n` gives the same spans as the same SDL with `\n` (Task 2,
   Task 3).
2. **Empty and comment-only input.** `render('')` and `render('# notes\n')` return an empty graph,
   an empty design (built-in models only, Decision 3), no problems, and an `openapi.json` with
   `"paths": {}` (Task 11, Task 18).
3. **A resource with every operation off.**
   `@resource(get: NONE, list: NONE, create: NONE, update: NONE, delete: NONE)` builds, has no
   methods and keeps its read model (Task 13, Task 15).
4. **A resource that owns itself.** `parent: Folder! @owner` on `Folder` is an `ownership_loop` on
   that one field (Task 9).
5. **Non-ASCII text before a problem.** `"""Café"""` on the line above a field: the field's span
   still counts one column per character (Task 3).

## Decisions needed

Each task that depends on one of these names it and builds the recommended option.

1. **Standard method names on custom methods.** The notation says a custom method "can't take a
   standard method's name (such as `get` or `list`)". The spec's `method_name_taken` fires only on a
   collision with a method the design actually has, so `list` as a custom method on a resource with
   `list: NONE` would pass. Recommendation: `method_name_taken` also fires for a custom method named
   `get`, `batchGet`, `list`, `create`, `update`, `delete` or `undelete`, whatever the operations.
2. **What `Model.sdl` looks like.** The spec says each model is "written as SDL" but not how.
   Recommendation: `print` from graphql-js of an object type named after the model, with each
   field's description, its AIP wire type (the ID field as `name`, a reference as `String` keeping
   the field's list shape and `!` marks, so `[Author!]!` becomes `[String!]!`; `DateTime` as
   `DateTime`; `Date` and `LocalTime` as `Date` and `TimeOfDay`), and its validation directives as
   written. The outer `!` follows `ModelField.required`, so every field of an update body is
   nullable in its SDL. Read-only fields carry no marker.
3. **Built-in models.** The spec lists `Empty`, `Status`, `Date` and `TimeOfDay` as generated models
   but not when they appear. Recommendation: all four are always in `design.models` and in
   `components.schemas`, so every file has the same base.
4. **Wording of the `filter` and `orderBy` descriptions.** The spec says they list the fields.
   Recommendation: `Fields you can filter by: title, genre.` and
   `Fields you can sort by: title, createdAt.`, fields in declaration order. The list builder puts
   them in `Param.description`, so the format only copies them.
5. **Request and response models when an operation is off.** The spec lists `Create<Name>Body`,
   `Update<Name>Body`, `List<Plural>Response` and `BatchGet<Plural>Response` for each resource
   without saying whether they exist when their method doesn't. Recommendation: a request or
   response model exists only when its method does; the read model always exists. The names stay
   reserved either way (`model_name_taken`).
6. **Model fields with no graph source.** The spec says each `ModelField` keeps
   `from: Field | Argument`, but the list and batch-get responses (`books`, `nextPageToken`) and the
   built-in models' fields come from no graph field. Recommendation: `from?: Field | Argument`,
   absent on generated and built-in fields. Only graph `Field`s of resource kind become `String`; a
   generated response field such as `ListBooksResponse.books` keeps the resource kind, so the format
   writes `$ref Book`. The built-in schemas are written by `modelSchema` from fixed values: `Status`
   is `{ code: integer (int32), message: string, details: array of object }`, `Date` is
   `{ year, month, day }` and `TimeOfDay` is `{ hours, minutes, seconds, nanos }`, all
   `integer (int32)`.

---

## Core (`@graphqall/core`)

### Task 1: Workspace, problems and shared types

**Files:**

- Modify: `package.json` (no change to scripts), `packages/core/package.json` (dependencies),
  `packages/core/tsconfig.json`, `packages/core/vitest.config.ts` (remove `passWithNoTests`),
  `eslint.config.mjs`
- Create: `packages/core/tsconfig.src.json`, `packages/core/src/problems.ts`,
  `packages/core/tests/factories.ts`
- Test: `packages/core/tests/problems.test.ts`

**Interfaces:**

- Produces: `Span`, `ProblemKind`, `ProblemCode` (union of the 46 codes in the spec's table),
  `Problem<C>`, `Outcome<T>`, `Source` exactly as in the spec's "Shared types";
  `problem<C extends ProblemCode>(code: C, where: Span, path?: string): Problem<C>`, which fills
  `kind`, `severity` and `message` from one table keyed by code. Test factory
  `makeSpan(overrides?: Partial<Span>): Span`, default
  `{ line: 1, column: 1, endLine: 1, endColumn: 2 }`.

- [ ] **Step 1: Add the dependencies and the runtime boundary.** Add `graphql`, `neverthrow`,
      `pluralize`, `@types/pluralize` to `@graphqall/core`. `tsconfig.src.json` extends the base
      with `lib: ["ES2022"]`, `types: []`, `include: ["src/**/*.ts"]`; `tsconfig.json` keeps tests
      and the Vitest config with Node types; `typecheck` runs both
      (`tsc --noEmit -p tsconfig.src.json && tsc --noEmit`). Split each `no-restricted-imports`
      block in `eslint.config.mjs` into a `packages/<name>/src/**` block (boundary patterns plus
      `node:*`) and a block for the package's other files (boundary patterns only), as the spec's
      "Runtime boundary" says.
- [ ] **Step 2: Verify the boundary fails.** Temporarily add `import 'node:fs'` to
      `packages/core/src/index.ts`. Run `pnpm lint`. Expected: FAIL naming `node:fs`. Remove it.
- [ ] **Step 3: Write the failing test.** One `it.each` row per code in the spec's Problem codes
      table, asserting kind and severity:

```ts
it.each([
  ["syntax_error", "invalid_sdl", "error"],
  ["query_argument_nested", "invalid_design", "error"],
  ["ownership_too_deep", "invalid_design", "warning"],
  ["directive_ignored", "invalid_design", "warning"],
  // … every row of the spec's table
] as const)("gives %s the kind %s and severity %s", (code, kind, severity) => {
  expect(problem(code, makeSpan({ line: 3, endLine: 3 }), "Book.title")).toMatchObject({
    code,
    kind,
    severity,
    where: { line: 3, column: 1, endLine: 3, endColumn: 2 },
    path: "Book.title",
  });
});
it.each(CODES)("gives %s a message", (code) => {
  // CODES: the first column of the hand-written table above
  expect(problem(code, makeSpan()).message).not.toBe("");
});
```

- [ ] **Step 4: Run it to see it fail.**
      `pnpm --filter @graphqall/core exec vitest run tests/problems.test.ts`. Expected: FAIL,
      `problem` not exported.
- [ ] **Step 5: Implement `problems.ts`.** Messages are written for display in a UI (notation,
      Problems); tests never check them.
- [ ] **Step 6: Run it to see it pass**, then `pnpm check` and `pnpm mutation`. Expected: PASS,
      every mutant caught.
- [ ] **Step 7: Commit** `core: problem codes and shared types`.

### Task 2: `splitBlocks`

**Files:** Create `packages/core/src/split-blocks.ts`. Test
`packages/core/tests/split-blocks.test.ts`.

**Interfaces:**

- Produces:
  `type Block = { text: string; start: { line: number; column: number }; keyword?: string; name?: string }`;
  `splitBlocks(sdl: string): readonly Block[]`.

- [ ] **Step 1: Write the failing tests**, one per rule in the spec's step 1. Each asserts the whole
      `Block[]` (text, start, keyword, name):
  - two definitions on separate lines give two blocks with starts `{1,1}` and `{line,1}`;
  - a field named `type` (`  type: String`) and an argument list (`input(x: Int)`) do not start a
    block;
  - a `"""` description on the lines before `type Book` belongs to Book's block, and `start` is the
    description's first line;
  - leading `# notes` and blank lines belong to no block; leading `garbage` is one unnamed block;
  - an indented definition (`  type Book @resource {`) starts at column 3;
  - `type Book @resource { titl` keeps `keyword: 'type'`, `name: 'Book'`;
  - an unterminated `"` ends at the end of its line, so the next line's `type Author` starts a
    block;
  - an unterminated `"""` followed by a line `type Author @resource {` splits there; the `"""` stays
    in the previous block;
  - Review Focus 1: the same SDL with `\r\n` gives the same starts;
  - `splitBlocks('')` and `splitBlocks('# only\n')` give `[]`.
- [ ] **Step 2: Run them to see them fail.**
      `pnpm --filter @graphqall/core exec vitest run tests/split-blocks.test.ts`.
- [ ] **Step 3: Implement `splitBlocks(sdl)`** as a character scanner that never throws.
- [ ] **Step 4: Run them to see them pass.**
- [ ] **Step 5: Commit** `core: split SDL into definition blocks`.

### Task 3: `parseBlocks` and span mapping

**Files:** Create `packages/core/src/parse-blocks.ts`, `packages/core/src/locate.ts`. Test
`packages/core/tests/parse-blocks.test.ts`.

**Interfaces:**

- Consumes: `Block`, `problem`, `Outcome` (Tasks 1, 2).
- Produces:
  `type ParsedBlock = { block: Block; document?: DocumentNode; stub?: { keyword: string; name: string } }`;
  `parseBlocks(blocks: readonly Block[]): Outcome<readonly ParsedBlock[]>`;
  `toSpan(block: Block, loc: Location): Span` (maps a graphql `Location` to the caller's SDL by the
  spec's formula); `sourceOf(block: Block, node: ASTNode): Source`.

- [ ] **Step 1: Write the failing tests.**
  - R12 location: SDL with `Publisher` on lines 1–5, `type Book @resource { titl` on line 7 and
    `Author` on lines 9–13 gives exactly one problem:
    `{ kind: 'invalid_sdl', code: 'syntax_error', severity: 'error', where: { line: 7, column: 23, endLine: 7, endColumn: 27 }, path: 'Book' }`
    (the `<EOF>` error is placed on the last token `titl`), and `Publisher` and `Author` parse.
  - A mid-line error (`type Book @resource { title String }` on line 2, where `String` is
    unexpected) spans from the error column to the end of line 2.
  - `sourceOf` on a field in the second block gives `where` in caller lines, and on an indented
    first line adds the indent to the column.
  - `Source.sdl` of `"""A book."""\ntype Book { title: String # note\n}` contains `A book.` and not
    `note`.
  - Review Focus 1: the R12 SDL and the mid-line error SDL written with `\r\n` give the same `where`
    as with `\n`; no span ends one column late because of `\r`.
  - Review Focus 5: a field below `"""Café"""` has the column you count by hand.
  - A failed block with a name becomes a stub; a failed unnamed block yields only its problem.
- [ ] **Step 2: Run them to see them fail.**
      `pnpm --filter @graphqall/core exec vitest run tests/parse-blocks.test.ts`.
- [ ] **Step 3: Implement.** Catch only `GraphQLError` from `parse`; rewrite its message from the
      table of the four forms in the spec (step 2); rethrow anything else.
- [ ] **Step 4: Run them to see them pass.**
- [ ] **Step 5: Commit** `core: parse blocks and map spans`.

### Task 4: Prelude, graph types and the graph for valid SDL

**Files:** Create `packages/core/src/prelude.ts`, `packages/core/src/graph.ts`,
`packages/core/src/build-graph.ts`, `packages/core/src/plural.ts`. Test
`packages/core/tests/build-graph.test.ts`, `packages/core/tests/plural.test.ts`,
`packages/core/tests/support.ts`.

**Interfaces:**

- Consumes: `ParsedBlock`, `sourceOf` (Task 3).
- Produces: every type in the spec's "The resource graph" (`ResourceGraph`, `Resource`,
  `Operations`, `Field`, `TypeRef`, `Validation`, `CustomMethod`, `Argument`, `AppliedDirective`,
  plus `ObjectType`, `InputType`, `EnumType`, `DirectiveDeclaration`, `Selection` with the members
  the spec lists);
  `type Definition = { name: string; kind: 'resource' | 'object' | 'input' | 'enum' | 'directive' | 'stub'; broken: boolean; refs: readonly Ref[]; value?: GraphNode }`;
  `type Ref = { name: string; where: Span; path: string }`;
  `type DraftGraph = readonly Definition[]`;
  `buildGraph(parsed: readonly ParsedBlock[]): Outcome<DraftGraph>`; `PRELUDE: string`;
  `pluralOf(name: string): string`. Test helper `draftOf(sdl): Outcome<DraftGraph>` in
  `tests/support.ts` runs split, parse and build; `valueOf(draft, name)` returns one definition's
  built node.

- [ ] **Step 1: Write the prelude** as one SDL string, from the notation: the scalars `DateTime`,
      `Date`, `LocalTime`; the enums `Off { NONE }`, `GetMode { NONE MANY }`,
      `DeleteMode { NONE SOFT }`, `Format { EMAIL URI UUID }`; the directives
      `@resource(get: GetMode,     list: Off, create: Off, update: Off, delete: DeleteMode, plural: String) on OBJECT`,
      `@created`, `@updated`, `@deleted`, `@owner`, `@filterBy`, `@sortBy`, `@expandable`, `@query`,
      `@mutation` on `FIELD_DEFINITION`, `@input(create: Off, update: Off) on FIELD_DEFINITION`,
      `@length(min: Int, max: Int)`, `@range(min: Float, max: Float)`, `@pattern(regex: String!)`,
      `@format(as: Format!)` on `FIELD_DEFINITION | ARGUMENT_DEFINITION | INPUT_FIELD_DEFINITION`,
      `@subset(fields: String!) on FIELD_DEFINITION`, `@number(value: Int!) on FIELD_DEFINITION`.
- [ ] **Step 2: Write the failing tests.** For the notation's own example (Publisher, Book, Author,
      Genre), assert by hand: `Book.plural === 'books'`; `Book.id.name === 'bookId'`;
      `Book.operations` is `{ get: 'ON', list: true, create: true, update: true, delete: 'SOFT' }`;
      `Book.owner` is `{ field: <publisher field>, resource: 'Publisher' }`; the `isbn` field's
      `input` is `{ create: true, update: false }` and `validation.pattern` is `'^[0-9]{13}$'`;
      `title.validation.length` is `{ min: 1, max: 200 }`, `filterBy` and `sortBy` true;
      `authors.type` is
      `{ name: 'Author', kind: 'resource', nonNull: true, list: { itemNonNull: true } }`; `archive`
      is `{ target: 'item', kind: 'mutation', args: [bookId, reason] }`; `Genre` is an enum with
      values `FICTION`, `NONFICTION`; server-set and owner fields have `input` all false. R9 every
      `@resource` value, as a table from the argument to the expected `Operations`: none given →
      `{ get: 'ON', list: true, create: true, update: true, delete: 'ON' }`; `get: NONE` →
      `get: 'NONE'`; `get: MANY` → `'MANY'`; `list: NONE` → `list: false`; `create: NONE` →
      `create: false`; `update: NONE` → `update: false`; `delete: NONE` → `'NONE'`; `delete: SOFT` →
      `'SOFT'`. R9 every `@input` combination: none → `{ create: true, update: true }`;
      `create: NONE` → `{ create: false, update: true }`; `update: NONE` →
      `{ create: true, update: false }`; both → `{ create: false, update: false }`. A declared
      foreign directive `@auth(role: "admin")` on a field is kept as
      `{ name: 'auth', args: { role: '"admin"' } }` (R7). `pluralOf`: `Book` → `books`, `Person` →
      `people`, `BookShelf` → `bookShelves`; `@resource(plural: "people")` wins.
- [ ] **Step 3: Run them to see them fail.**
- [ ] **Step 4: Implement** `buildGraph` for valid input: parse the prelude once, read each
      definition into its graph node and its refs. Each check from Tasks 5–9 plugs in as a function
      returning `Result`; combine a definition's independent checks with
      `Result.combineWithAllErrors`, and chain dependent checks with `andThen` (spec, Shared types).
      A definition with any error is `broken` and has no `value`.
- [ ] **Step 5: Run them to see them pass.**
- [ ] **Step 6: Commit** `core: prelude and resource graph`.

Every check task below (5–9) has the same shape: an `it.each` table where each row is a small SDL
snippet and the exact list of problems (kind, code, severity, `where`, `path`), with rows that pass
giving `[]`. Run with `pnpm --filter @graphqall/core exec vitest run tests/<file>`. Each task
commits once its table passes and `pnpm mutation` catches every mutant. Spans in the rows are
counted by hand from the snippet.

### Task 5: SDL validity checks

**Files:** Create `packages/core/src/checks/sdl-validity.ts`,
`packages/core/src/checks/directives.ts`. Test `packages/core/tests/checks/sdl-validity.test.ts`,
`packages/core/tests/checks/directives.test.ts`.

**Interfaces:** Each check is `(definition, context) => Result<…, Problem<codes>>` with the codes it
can return in its type (TS-01 rule 4). `context` holds the prelude and the other definitions by
name.

- [ ] **Step 1: Write the failing tables**, one row per code at least, located as the spec's table
      says:
  - `duplicate_definition` (two `type Author`, and a user `enum Format`, which clashes with the
    prelude); `scalar DateTime` gives both `duplicate_definition` and `unsupported_definition`;
    `unsupported_definition` for `interface`, `union`, `scalar Money`, `extend type`, `schema`, and
    `implements` on a type; `duplicate_field`; `duplicate_argument` (in a field and in a directive
    declaration); `duplicate_enum_value`; `unknown_type`; `wrong_type_position` (an object type and
    a resource type as an argument type, an object type as an input field type, an input type as a
    field type).
  - `undeclared_directive` (`@auth` with no declaration); `directive_wrong_location` (`@resource` on
    a field); `directive_repeated` (`@created @created`); `unknown_directive_argument`
    (`@length(maximum: 3)`); `missing_directive_argument` (`@pattern`); `invalid_directive_argument`
    (`@format(as: PHONE)`, `@resource(get: SOFT)`, `@resource(plural: "my books")`);
    `duplicate_argument` for the same argument twice in one directive use; a declared
    `@auth(role: String!)` used as `@auth` and as `@auth(role: 1)`.
  - Passing rows: the notation's example; `directive @auth(role: String!) on FIELD_DEFINITION` used
    as `@auth(role: "admin")`.
- [ ] **Step 2: Run, see fail. Step 3: Implement. Step 4: Run, see pass. Step 5: Commit**
      `core: SDL validity checks`.

### Task 6: Identity and timestamps

**Files:** Create `packages/core/src/checks/identity.ts`, `packages/core/src/checks/timestamps.ts`.
Test `packages/core/tests/checks/identity.test.ts`, `packages/core/tests/checks/timestamps.test.ts`.

- [ ] **Step 1: Write the failing tables.**
  - Identity: no `ID!` → `missing_id` on the type name; `id: ID!` and `bookId: ID!` → `multiple_ids`
    on the second field; `id: ID` (nullable) → `missing_id`; a value type without an `ID!` field
    passes.
  - Timestamps: `missing_created`, `multiple_created`, `missing_updated`, `multiple_updated`;
    `createdAt: String! @created` → `timestamp_wrong_type` on `String!`; `@updated` missing with
    `update: NONE` passes; `delete: SOFT` without `@deleted` → `missing_deleted`; two `@deleted` →
    `multiple_deleted`; `@deleted` without `delete: SOFT` → `deleted_without_soft_delete`;
    `deletedAt: DateTime! @deleted` → `timestamp_wrong_type` (it must be nullable `DateTime`).
- [ ] **Steps 2–5** as above. Commit `core: identity and timestamp checks`.

### Task 7: Field directives and validation

**Files:** Create `packages/core/src/checks/field-directives.ts`. Test
`packages/core/tests/checks/field-directives.test.ts`.

- [ ] **Step 1: Write the failing table.** R9 every validation directive and argument:
      `@length(min: 1, max: 200)` on `String` and on `[String!]`, `@length(min: 3, max: 3)` and
      `@range(min: 2, max: 2)` pass (the boundary), `@length(min: 5, max: 2)` →
      `min_greater_than_max`; `@length` on `Int` → `validation_wrong_type`; `@range(min: 0, max: 5)`
      on `Int` and on `Float`, `@range` on `String` → `validation_wrong_type`,
      `@range(min: 5,     max: 1)` → `min_greater_than_max`; `@pattern(regex: "^a$")` on `String`,
      on `Int` → `validation_wrong_type`; `@format(as: EMAIL|URI|UUID)` on `String`, on `Boolean` →
      `validation_wrong_type`. The same rows on an argument and on an input field. Each passing row
      also asserts the `Validation` value. `@created`, `@updated`, `@deleted`, `@owner`, `@input`,
      `@filterBy`, `@sortBy`, `@subset`, `@expandable`, `@number`, `@query`, `@mutation` on a field
      of a value type → `directive_outside_resource`.
- [ ] **Steps 2–5.** Commit `core: field directive checks`.

### Task 8: References and custom methods

**Files:** Create `packages/core/src/checks/references.ts`,
`packages/core/src/checks/custom-methods.ts`. Test `packages/core/tests/checks/references.test.ts`,
`packages/core/tests/checks/custom-methods.test.ts`.

- [ ] **Step 1: Write the failing tables.**
  - References, on a field `author: Author!` where `Author` has `id: ID!`, `displayName: String!`,
    `address: Address` (a value type with `city: String`) and `agency: Agency!` (a resource):
    `@subset` and `@expandable` on a `String` field → `subset_not_reference`,
    `expandable_not_reference`; `@subset(fields: "{ id")` → `subset_syntax_error`;
    `"{ id nickname }"` → `subset_unknown_field`; `"{ id address { zip } }"` →
    `subset_unknown_field`; `"{ displayName }"` → `subset_missing_id`; `"{ id agency }"` →
    `subset_contains_resource`; `"{ id displayName }"` passes and gives `Selection`
    `[id, displayName]`, also on `authors: [Author!]!`; `"{ id address { city } }"` passes with a
    nested selection. Each `where` is the selection string. A selection that fails to parse reports
    only `subset_syntax_error` (the `andThen` chain stops).
  - Custom methods: a field with arguments and no marker → `arguments_without_method_kind`; both
    markers → `query_and_mutation` on the second;
    `archive(id: ID!, reason: String): Book! @mutation` → `target: 'item'`;
    `search(query: String!): [Book!]! @query` → `target: 'collection'`;
    `move(id: ID!, to: ID!): Book! @mutation` → item, with `to` an ordinary argument; a `@query`
    with an input argument whose field is an input object → `query_argument_nested` on the argument;
    a `@query` with an input of plain fields passes; `refresh: Book! @mutation` with no arguments is
    a collection custom method. These `@query` rows cover R9's "readOnly" case.
- [ ] **Steps 2–5.** Commit `core: reference and custom method checks`.

### Task 9: Ownership

**Files:** Create `packages/core/src/checks/ownership.ts`. Test
`packages/core/tests/checks/ownership.test.ts`.

- [ ] **Step 1: Write the failing table.** Two `@owner` fields → `multiple_owners` on the second;
      `@owner` on `String!`, on a nullable `Publisher`, on `[Publisher!]!` → `owner_not_resource` on
      the field's type; `Book` owns `Chapter` owns `Book` → `ownership_loop` on both `@owner`
      fields; Review Focus 4: `Folder` with `parent: Folder! @owner` → one `ownership_loop`;
      `A ← B ← C ← D` (D owned by C owned by B owned by A) → one `ownership_too_deep` warning on D's
      `@owner` field, and D is not broken; `A ← B ← C` passes.
- [ ] **Steps 2–5.** Commit `core: ownership checks`.

### Task 10: `removeTypes`, `toGraph` and `graphDefinitions`

**Files:** Create `packages/core/src/remove-types.ts`. Test
`packages/core/tests/remove-types.test.ts`.

**Interfaces:**

- Produces:
  `removeTypes(definitions: DraftGraph, broken: readonly Definition[]): Outcome<DraftGraph>`.
  `broken` holds definitions, not names: with two definitions of one name only the broken one goes,
  and a reference is a dependency on a removed definition only when no unbroken definition of that
  name remains (spec step 3: "references resolve to the first");
  `toGraph(definitions: DraftGraph): ResourceGraph`;
  `graphDefinitions(graph: ResourceGraph): DraftGraph`. Test helper `graphOf(sdl)` in
  `tests/support.ts` runs split, parse, build, `removeTypes` and `toGraph`.

- [ ] **Step 1: Write the failing tests.**
  - R11 acceptance 1: `Publisher` and `Author` valid, `Book` with two `ID!` fields → the graph holds
    `Publisher` and `Author` only, and the one problem is `multiple_ids` in `Book`.
  - R11 acceptance 2: add a valid `Chapter` with `book: Book! @owner` → `Chapter` is gone too, with
    `depends_on_invalid_type` at `Book` in `book: Book!`, `path: 'Chapter.book'`.
  - The chain continues: `Verse` owned by `Chapter` is removed with its own
    `depends_on_invalid_type`.
  - A dependency through an argument type, a return type and a foreign directive each removes the
    dependent.
  - Two `type Author`, the first valid, and a `Book` with `author: Author!`: the only problem is
    `duplicate_definition` on the second `Author`, and the first `Author` and `Book` stay.
  - A stub `Publisher` (Task 3) referenced as `publisher: Publisher! @owner`: the only problem
    besides the stub's `syntax_error` is `depends_on_invalid_type` (no `unknown_type`, no
    `owner_not_resource`).
  - `toGraph(graphDefinitions(g))` equals `g`.
- [ ] **Steps 2–5.** Commit `core: remove broken types and their dependents`.

### Task 11: Targets and `render`

**Files:** Create `packages/core/src/target.ts`, `packages/core/src/render.ts`; modify
`packages/core/src/index.ts` (export the public API). Test `packages/core/tests/render.test.ts`.

**Interfaces:**

- Produces: `IDesignStyle<D>`, `IFileFormat<D>`, `ITarget<D>`, `OutputFile`, `RenderResult<D>`,
  `composeTarget`, `render` exactly as in the spec's step 5. `index.ts` exports these, the graph and
  problem types, `problem`, `Definition`, `Ref`, `DraftGraph`, `removeTypes`, `toGraph` and
  `graphDefinitions`, which is everything style-aip uses from core.

- [ ] **Step 1: Write the failing tests** with a stub style and stub format written in the test.
  - `composeTarget(style, format).run(graph)` passes the style's design to the format and returns
    both steps' problems.
  - R1: `render(sdl, target)` returns `graph`, `design`, `files` and `problems`.
  - R12 acceptance: the SDL from Task 3 returns a graph with `Publisher` and `Author` and the one
    `syntax_error`.
  - Problems come back ordered by `where` (line, then column).
  - Review Focus 2: `render('')` and `render('# notes\n')` give an empty graph and no problems.
- [ ] **Steps 2–5.** Commit `core: targets and render`.

## AIP style (`@graphqall/style-aip`)

### Task 12: Package, design types, names and AIP checks

**Files:** Create `packages/style-aip/` (`package.json` with dependencies `@graphqall/core`
(workspace), `graphql` and `openapi3-ts`, dev dependency `@seriousme/openapi-schema-validator`, and
the same `typecheck` script as core, `tsc --noEmit -p tsconfig.src.json && tsc --noEmit`;
`tsconfig.json`, `tsconfig.src.json`, `vitest.config.ts`, `tests/setup.ts` copied from core),
`src/design.ts`, `src/names.ts`, `src/checks.ts`. Test `tests/names.test.ts`,
`tests/checks.test.ts`, `tests/factories.ts`.

**Interfaces:**

- Produces: `AipDesign`, `AipResource`, `AipMethod`, `Param`, `Model` from the spec, and
  `type ModelField = { name: string; description?: string; type: TypeRef; required: boolean; readOnly: boolean; validation: Validation; from?: Field | Argument }`
  (Decision 6), where `type` is the AIP wire type: a graph field of resource kind becomes `String`
  (keeping its list shape) and the ID field becomes `name: String!`;
  `resourcePattern(resource, graph): string`; `collectionPath(resource, graph): string`;
  `customMethodName(method, resource): string`;
  `aipChecks(graph): { problems: readonly Problem[]; broken: readonly string[] }`;
  `ignoredDirectives(graph): readonly Problem[]`. Test factories in `tests/factories.ts`:
  `makeResource(overrides?): Resource` (default: a resource `Book` with plural `books`, an `id: ID!`
  field, `createdAt` and `updatedAt`, every operation on, no owner) and
  `makeGraph(overrides?): ResourceGraph` (default: one `makeResource()`). Tasks 13 to 17 build every
  resource and graph they pass to a builder, the style or the format with these, and
  `graphFrom(sdl)` gets a graph from SDL through core's public `render` with a pass-through stub
  target (core's `tests/support.ts` isn't exported). `package.json` has
  `"name": "@graphqall/style-aip"` and `"exports": { ".": "./src/index.ts" }`.

- [ ] **Step 1: Write the failing tests.**
  - Names: `Book` owned by `Publisher` → pattern `publishers/{publisher}/books/{book}`, collection
    path `publishers/*/books`; `BookShelf` → `bookShelves/{bookShelf}`; `archive` on an item of
    `Book` → `ArchiveBook`; `search` on the collection → `SearchBooks`.
  - Checks table: a field `name: String` on a resource → `field_name_reserved`, resource dropped; a
    resource whose ID field is `name: ID!` passes; a value type `Status` → `model_name_taken`; a
    value type `CreateBookBody` beside `Book @resource(create: NONE)` → `model_name_taken` (Decision
    5); a custom method `get` on `Book` → `method_name_taken`; the item method
    `createBook(shelfId: ID!): Shelf! @mutation` on `Shelf` beside resource `BookShelf` →
    `method_name_taken` (`CreateBookShelf`; on the collection it would be `CreateBookShelves` and
    collide with nothing); Decision 1: a custom `list` on a resource with `list: NONE` →
    `method_name_taken`; `Book` and `Novel @resource(plural: "books")` → `path_taken` on `Novel`;
    `@subset`, `@expandable`, `@number` and a foreign directive → one `directive_ignored` warning
    each, at the directive use (R6, R7).
- [ ] **Step 2: Run, see fail.**
      `pnpm --filter @graphqall/style-aip exec vitest run tests/checks.test.ts tests/names.test.ts`.
- [ ] **Steps 3–4: Implement, see pass. Step 5: Commit** `style-aip: package, names and checks`.

### Task 13: `chooseMethods`, the factory and the standard builders

**Files:** Create `src/choose-methods.ts`, `src/builders.ts`. Test `tests/choose-methods.test.ts`,
`tests/builders.test.ts`.

**Interfaces:**

- Produces: `MethodChoice`, `chooseMethods`, `methodBuilder`, `IMethodBuilder` exactly as in the
  spec.

- [ ] **Step 1: Write the failing tests.**
  - `chooseMethods`, R9 every `@resource` value: default →
    `[get, list{showDeleted: false, filterFields: [], sortFields: []}, create, update, delete{soft: false}]`;
    `get: MANY` adds `batchGet` after `get`; `delete: SOFT` gives `delete{soft: true}`, `undelete`
    and `showDeleted: true`; each `NONE` removes its choice; Review Focus 3: all off → `[]`;
    `@filterBy` on `genre` and `title`, `@sortBy` on `title` and `createdAt` →
    `filterFields: ['title', 'genre']`, `sortFields: ['title', 'createdAt']` in declaration order;
    custom methods come last, in written order.
  - One test per builder, asserting the whole `AipMethod` for `Book` owned by `Publisher`, from the
    spec's "Paths and names" table. For example get:
    `{ name: 'GetBook', verb: 'GET', path: '/v1/publishers/{publisher}/books/{book}', pathParams: ['publisher', 'book'], query: [], response: { model: 'Book' }, from: book }`;
    list with neither `@filterBy` nor `@sortBy` and no soft delete: query exactly `pageSize` (Int)
    and `pageToken` (String); list with filter and sort: also `filter` (String) and `orderBy`
    (String) with the Decision 4 descriptions, and `showDeleted` (Boolean) when soft; response
    `{ model: 'ListBooksResponse' }`; batchGet: `names` repeated String; update: query `updateMask`,
    body `UpdateBookBody`; delete: response `Empty`, or `Book` when soft; undelete:
    `POST …/books/{book}:undelete`, body `Empty`. A resource with no owner has no parent
    (`/v1/books`).
- [ ] **Steps 2–5.** Commit `style-aip: standard method builders`.

### Task 14: The custom method builder

**Files:** Modify `src/builders.ts`. Test `tests/custom-builder.test.ts`.

- [ ] **Step 1: Write the failing tests** (R9 item, collection, `@query`):
  - `archive(bookId: ID!, reason: String): Book! @mutation` →
    `{ name: 'ArchiveBook', verb: 'POST', path: '/v1/publishers/{publisher}/books/{book}:archive', pathParams: ['publisher', 'book'], query: [], body: 'ArchiveBookBody', response: { type: <Book!> }, from: archive }`.
  - `search(query: String!): [Book!]! @query` →
    `{ name: 'SearchBooks', verb: 'GET', path: '/v1/publishers/{publisher}/books:search', pathParams: ['publisher'], query: [{ name: 'query', schema: <String!>, repeated: false }], response: { type: <[Book!]!> }, from: search }`,
    with no `body`.
  - A `@query` with `tags: [String!]` → `tags` repeated; with `filter: BookFilter` whose fields are
    `title: String` and `genre: Genre` → params `filter.title` and `filter.genre`.
  - A `@mutation` on the collection with no arguments gets an empty body model; no paging or extra
    parameters are ever added (notation, Custom methods).
- [ ] **Steps 2–5.** Commit `style-aip: custom method builder`.

### Task 15: Models

**Files:** Create `src/models.ts`, `src/model-sdl.ts`. Test `tests/models.test.ts`.

**Interfaces:**

- Produces: `buildModels(resource, graph): readonly Model[]`; `valueModels(graph): readonly Model[]`
  (objects, inputs, enums); `BUILTIN_MODELS: readonly Model[]` (Decision 3);
  `modelSdl(model): string` (Decision 2).

- [ ] **Step 1: Write the failing tests.**
  - R9 every `@input` combination and `!` on create and update, using `Book` from the notation: read
    model `Book` has every field, `name` (from `bookId`) read-only and required, `createdAt`,
    `updatedAt`, `deletedAt`, `publisher` read-only; `isbn` required and not read-only;
    `CreateBookBody` has `title`, `isbn`, `authors`, `genre`, `publishedOn`, with `title`, `isbn`,
    `authors` required; `UpdateBookBody` has `title`, `authors`, `genre`, `publishedOn` (no `isbn`)
    and nothing required.
  - A field with `@input(create: NONE, update: NONE)` is read-only and in neither body.
  - `title`'s `validation` (`length` 1 to 200) is on its `CreateBookBody` and `UpdateBookBody`
    fields too (R9, on create and on update).
  - `ListBooksResponse` has `books: [Book]` and `nextPageToken: String`; `ArchiveBookBody` has
    `reason`. A `get: MANY` resource has `BatchGetBooksResponse` with `books`; the notation's `Book`
    (`get` on) has no `BatchGetBooksResponse` (Decision 5).
  - Review Focus 3: a resource with every operation off has only its read model.
  - References: `author: Author!` is a required string field; `authors: [Author!]!` a list of
    strings.
  - Value types, inputs and enums become models under their own names; built-ins are always present.
  - `modelSdl` of `CreateBookBody` equals a hand-written SDL string per Decision 2.
- [ ] **Steps 2–5.** Commit `style-aip: models`.

### Task 16: `aipStyle()`

**Files:** Create `src/style.ts`. Test `tests/style.test.ts`.

**Interfaces:**

- Produces: `aipStyle(): IDesignStyle<AipDesign>`.

- [ ] **Step 1: Write the failing tests.**
  - `design(graph)` runs the AIP checks, removes their broken types with core's `removeTypes`
    (called with `defs = graphDefinitions(graph)` and the entries of `defs` whose names `aipChecks`
    returned; names are unique once core has removed duplicates, and the spec's step 4 passes names
    here), adds `directive_ignored` warnings, then builds each remaining resource's methods in the
    spec's order and its models.
  - A graph where `Book` has `name: String` returns a design without `Book` and without `Chapter`
    owned by `Book`, with `field_name_reserved` and `depends_on_invalid_type`.
  - `deviations` is `[]`.
  - The SDL-based rows get their graph from `graphFrom(sdl)` (Task 12).
- [ ] **Steps 2–5.** Commit `style-aip: design style`.

### Task 17: `aipOpenApiFormat`

**Files:** Create `src/openapi/schema.ts`, `src/openapi/format.ts`. Test
`tests/openapi-schema.test.ts`, `tests/openapi-format.test.ts`.

**Interfaces:**

- Produces: `schemaFor(type: TypeRef, validation: Validation): SchemaObject | ReferenceObject` (from
  `openapi3-ts/oas31`) for a field, parameter or response: an enum, object, input or resource kind
  is a `$ref` to its model (a resource's read model); `modelSchema(model: Model): SchemaObject` for
  a model, which writes an enum model's `enum` or `oneOf` from `Model.from`;
  `aipOpenApiFormat(options): IFileFormat<AipDesign>`.

- [ ] **Step 1: Write the failing tests.**
  - One row per line of the spec's mapping table: `String` and `ID` → `{ type: 'string' }`;
    `Boolean` → `{ type: 'boolean' }`; an object or input type → `$ref` to its model; `Int` →
    `{ type: 'integer', format: 'int32' }`; `Float` → `{ type: 'number', format: 'double' }`;
    `DateTime` → `{ type: 'string', format: 'date-time' }`; `Date` →
    `$ref '#/components/schemas/Date'`; `LocalTime` → `$ref '#/components/schemas/TimeOfDay'`; a
    field of enum type → `$ref` to the enum's model; `modelSchema` of an enum model without value
    descriptions → `{ type: 'string', enum: [...] }`, with one → `oneOf` of
    `{ const, description }`; `@length(min: 1, max: 5)` on a String → `minLength: 1, maxLength: 5`,
    on a list → `{ type: 'array', minItems: 1, maxItems: 5 }`; `@range`, `@pattern`,
    `@format(as: EMAIL)` → `format: 'email'`; a description lands on the property.
  - Resources: a reference field (wire type `String!` in the model) → `{ type: 'string' }`; a list
    of references → `{ type: 'array', items: { type: 'string' } }`; a custom method returning
    `Book!` → `$ref '#/components/schemas/Book'`; returning `[Book!]!` → an array of that `$ref`.
  - The format writes one file `openapi.json`, `openapi: '3.1.0'`, `info` from the options,
    two-space JSON; each resource a tag; each method an operation with `operationId` equal to its
    name and a `default` response `$ref` `Status`; `updateMask` a string query parameter; the
    `filter` and `orderBy` descriptions copied from the design (Decision 4); required fields listed
    in `required`; read-only fields have `readOnly: true`; no schema has `null` in its type.
- [ ] **Steps 2–5.** Commit `style-aip: OpenAPI format`.

### Task 18: `googleHttpTarget`, end-to-end examples and R4

**Files:** Create `src/target.ts`, `src/index.ts`; `examples/notation/`, `examples/time-types/`,
`examples/references/`, `examples/ownership-chain/`, `examples/descriptions-and-comments/`,
`examples/empty/` (each `schema.graphql` and a hand-written `openapi.json`). Test
`tests/examples.test.ts`, `tests/custom-format.test.ts`.

**Interfaces:**

- Produces: `googleHttpTarget(options): ITarget<AipDesign>`; `index.ts` exports `aipStyle`,
  `aipOpenApiFormat`, `googleHttpTarget` and the design types.

- [ ] **Step 1: Write the expected files by hand** from the notation and the spec. The
      `openapi.json` files are reviewed in the pull request and never regenerated from engine output
      (TS-07 rule 2).
- [ ] **Step 2: Write the failing tests.**
  - For each example: render with `googleHttpTarget({ title: 'Example', version: '1.0.0' })`,
    `JSON.parse` the file, `toEqual` the expected file, and validate it with the OpenAPI 3.1
    validator with no errors (R10). `examples/empty` covers Review Focus 2.
  - `descriptions-and-comments`: a `"""` description appears as `description`; the `#` comment text
    `zzz-private-note` appears nowhere in `JSON.stringify` of the whole `RenderResult` (R9).
  - R6: rendering SDL with `@subset`, `@expandable` and `@number` (written on the same lines as
    other text, so no span moves) and the same SDL without them gives `toEqual` parsed
    `openapi.json` files, and the only extra problems are the three `directive_ignored` warnings.
  - R4: importing only `@graphqall/core` by package name and the style through its entry file
    `../src/index.ts` (the ESLint style boundary forbids `@graphqall/style-*` imports inside a style
    package), `composeTarget(aipStyle(), stubFormat)` passes the AIP design to the stub and `render`
    returns the stub's files.
- [ ] **Steps 3–5.** Run `pnpm check`. Commit `style-aip: Google HTTP target and examples`.

### Task 19: Problem codes in the notation (owner approval)

**Files:** Modify `docs/notation.md` (Problems section).

- [ ] **Step 1:** Add a table of every code from the spec's Problem codes table, with its kind,
      severity and a one-line meaning (R8). Add nothing else.
- [ ] **Step 2:** Run `pnpm lint`. Expected: PASS.
- [ ] **Step 3:** Commit `notation: problem codes` and say in the pull request that this section
      needs the owner's approval (AGENTS.md: a change to the notation).

## Test coverage

| Requirement                                                 | Tests                                                                                                                                                                                                                                                                       | Tasks          |
| ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| R1 inputs and outputs                                       | `render` returns all four parts                                                                                                                                                                                                                                             | 11             |
| R2 browser and Node                                         | `src` type-check without Node types; ESLint forbids `node:`                                                                                                                                                                                                                 | 1              |
| R3 Google HTTP, Google JSON mapping                         | builder tests (paths, verbs); schema rows for `Date`, `TimeOfDay`; examples                                                                                                                                                                                                 | 13, 14, 17, 18 |
| R4 add a format                                             | custom-format test                                                                                                                                                                                                                                                          | 18             |
| R5 every notation rule                                      | check tables, builder, model and schema tests                                                                                                                                                                                                                               | 4–10, 12–17    |
| R6 `@subset`, `@expandable`, `@number` warn, change nothing | `directive_ignored` rows; openapi.json equality                                                                                                                                                                                                                             | 12, 18         |
| R7 foreign directives kept, warned                          | graph test for `@auth`; `directive_ignored` row; undeclared row                                                                                                                                                                                                             | 4, 5, 12       |
| R8 one code per problem, in the notation                    | problem table test; notation task                                                                                                                                                                                                                                           | 1, 19          |
| R9 every rule, directive, argument and code tested          | check tables (each code), `@resource` and `@input` tables, validation table, custom method tests (`@query` for "readOnly"), reference tests, list builder tests (with filter and sort, and with neither), ownership table, time-type schema rows and example, comments test | 1–18           |
| R10 valid OpenAPI 3.1, compared with hand-written files     | examples test                                                                                                                                                                                                                                                               | 18             |
| R11 partial result on an error                              | acceptance 1 and 2                                                                                                                                                                                                                                                          | 10, 16         |
| R12 syntax error stays in its definition                    | location test; render acceptance                                                                                                                                                                                                                                            | 3, 11          |
