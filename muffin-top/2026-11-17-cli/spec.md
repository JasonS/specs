# Muffin Top CLI v1

Approved by Jason on 2026-11-17, in the project thread "Design the Muffin Top CLI", from a mock of
its terminal output.

## Problem

A project keeps its schema in an SDL file and wants the OpenAPI, proto or GraphQL file its own code
generators read. Today the only way to get that file is to copy it out of the playground.

## Direction

The CLI should come to drive schema maintenance in a project: making the contract files, keeping
them current, and later checking changes for breaks. v1 is the first step. Agents use it the same
way people and CI do, by running it and reading what it prints.

## Decision

One package, `@muffin-top/cli`, with two commands, modeled on openapi-generator, protoc and ESLint.
Arguments come first; there is no config file.

```text
muffin-top generate <schema> --to <output> [--out <folder>] [--format json|yaml] [--casing <casing>]
muffin-top check <schema> --to <output>
muffin-top --help
muffin-top --version
```

- `--to` takes an output kind the engine lists, such as `rest` or `aip-grpc`. `--format` and
  `--casing` take what that output lists, as in the playground.
- With `--out`, each file the output makes is written under that folder, at the path the output
  gives it (`api/openapi.json`, `proto/library_api/v1/library_api.proto`).
- Without `--out`, the file is printed to standard output, so it can be piped into a generator. An
  output that makes more than one file needs `--out`.
- `check` reports the schema's problems under that output's rules and never reads or writes a
  contract file. It takes `--to` because each standard warns about different things.
- CI that keeps the contract committed runs `generate` and then `git diff --exit-code`, so a
  stale file fails the build with the diff in its log. The docs give this recipe.
- Every problem prints to standard error, so a pipe carries only the file. A problem shows its
  severity, message and rule code, then a code frame at its span in the schema, then the standard's
  link when the rule has one and the title of each quick fix it offers.
- An error in the schema writes and prints no file. The file is all or nothing, so a code generator
  never reads a design with types removed.
- Exit codes follow ESLint's: 0 when it worked (warnings allowed), 1 for errors in the schema, 2 for a usage mistake, such as an unknown output or a schema file that
  can't be read.
- `--help` lists the commands, the outputs (from the engine, never a copy), the options and
  examples. It is plain text in the terminal; nothing opens a web page.
- Colors come from `util.styleText`, which turns them off when the stream isn't a terminal or
  `NO_COLOR` is set.

The CLI holds no rule, output or naming of its own. It reads files, calls the engine, and prints
what the engine returns. Anything it shows, a library user gets from `@muffin-top/engine` too.

### Dependencies

None new. Node gives `util.parseArgs` for arguments and `util.styleText` for colors. The code frame
is the CLI's own, because `@babel/code-frame` highlights JavaScript, not GraphQL.

### Parts

- `main.ts`, the composition root and the `bin`: makes the engine, the file reader and writer and
  the CLI, runs it with `process.argv`, and sets `process.exitCode`.
- `parseCommand(args, outputs)`: a function from the arguments to a `Command` (help, version,
  generate or check) or a usage error naming what is wrong.
- `ICli` with `run(args): Promise<ExitCode>`: reads the schema, renders it, prints problems, and
  writes or prints the files. It takes `IEngine`, a file reader, a file writer and the two
  streams through its constructor.
- `IFileReader` and `IFileWriter`, implemented over `node:fs/promises`: reading returns a
  `Result` whose error is a file that can't be read; writing makes the folders it needs.
- `formatProblem`, `formatHelp`: functions from the engine's data to text, given a `paint` function
  so tests read plain text.

An unexpected engine error (`code: 'unexpected'`) is thrown with its cause, as P-01 asks: Node
prints it and exits 1.

### Tests

Written from this spec against `ICli` with a fake engine and fake files: each exit code, problems
on standard error and the file on standard output, nothing written on an error, `check`
passing and failing, and every usage error. The code frame and help get tests of their text. One test runs the real engine on the Book example.

## Later

- `breaking`: compare the contract with a committed snapshot or a git branch, through each output's
  own checker (buf for proto, graphql-inspector for GraphQL). Each output owns its checker.
- A config file for projects with many outputs, and outputs loaded by package name.
- A local MCP server, once agents have been watched using the CLI.
- Docs recipes: a `package.json` script that pipes `generate` into openapi-typescript,
  `buf generate` or graphql-codegen, and an example repository.
- Publishing to npm. Node won't run TypeScript from `node_modules`, so the workspace's packages need
  a build step before any is published.

## Sources

- openapi-generator CLI, `generate -i -g -o`: <https://openapi-generator.tech/docs/usage>
- git diff `--exit-code`: <https://git-scm.com/docs/git-diff#Documentation/git-diff.txt---exit-code>
- ESLint CLI exit codes: <https://eslint.org/docs/latest/use/command-line-interface#exit-codes>
- Node.js `util.parseArgs` and `util.styleText`: <https://nodejs.org/api/util.html>
- NO_COLOR: <https://no-color.org/>
