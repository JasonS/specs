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
muffin-top generate <schema> --to <output> [--out <folder>] [--format json|yaml] [--casing <casing>] [--reporter pretty|json]
muffin-top check <schema> --to <output> [--reporter pretty|json]
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
- A reporter shows the problems, chosen with `--reporter` (Biome's name for it; `--format` already
  picks json or yaml). `pretty`, the default, prints to standard error, so a pipe carries only the
  file: each problem's severity, message and rule code, then a code frame at its span in the
  schema, then the standard's link when the rule has one and the title of each quick fix it offers,
  and last how many errors and warnings there were. `json` prints the engine's problems as they are,
  each with its file, to standard output for agents and CI, `[]` when there are none. Since
  `generate` without `--out` prints the file there, `--reporter json` needs `--out` with it.
- An error in the schema writes and prints no file. The file is all or nothing, so a code generator
  never reads a design with types removed.
- Exit codes follow ESLint's: 0 when it worked (warnings allowed), 1 for errors in the schema, 2
  for a usage mistake, such as an unknown output or a schema file that can't be read. A missing
  file and a folder are said in the CLI's words; any other system error, such as permission, in
  the system's (`EACCES: permission denied, open 'api/openapi.json'`).
- `--help` lists the commands, the outputs (from the engine, never a copy), the options and
  examples. It is plain text in the terminal; nothing opens a web page.
- Colors come from `util.styleText`, off when the stream isn't a terminal or `NO_COLOR` is set,
  on with `FORCE_COLOR`, as Node decides for each stream.

The CLI holds no rule, output or naming of its own. It reads files, calls the engine, and prints
what the engine returns. Anything it shows, a library user gets from `@muffin-top/engine` too.

### Dependencies

None new. Node gives `util.parseArgs` for arguments and `util.styleText` for colors. The code frame
is the CLI's own, because `@babel/code-frame` highlights JavaScript, not GraphQL.

### Parts

- `main.ts`, the composition root and the `bin`: makes every part below, runs the CLI with
  `process.argv`, and sets `process.exitCode`.
- `parseCommand(args, outputs)`: a function from the arguments to a `Command` (help, version,
  generate or check) or a `CliError` naming what is wrong.
- `ICli.run(argv): Promise<ExitCode>`, with `argv` as `process.argv`: parses, prints help or the version, or hands generate and
  check to the runner, and prints any `CliError`. It takes the engine's outputs, the runner and the
  two streams.
- `ISchemaRunner.run(command)`: reads the schema, renders it, reports the problems and, for
  generate without errors, puts the files. It returns whether the schema had errors, or a
  `CliError`.
- `IReporter.report(problems, schema)`: `PrettyReporter` on standard error and `JsonReporter` on
  standard output, as ESLint's formatters and Biome's reporters are. Each takes its stream.
- `IFileOutput.put(files, out)`: writes the files under `out`, saying each, or prints the one file
  without it.
- `IFileReader` and `IFileWriter`, over `node:fs/promises`: each returns a `Result` whose error is
  a file that can't be read or written; writing makes the folders it needs.
- `formatHelp` and the code frame: functions from the engine's data to text, given a `paint`
  function so tests read plain text.

An unexpected engine error (`code: 'unexpected'`) is thrown with its cause, as P-01 asks: Node
prints it and exits 1.

### Tests

Written from this spec, each part with hand-written fakes of the parts it takes: each exit code,
every usage error, each reporter's text, the file on standard output or under the folder, nothing
written on an error, and every file error. The file reader and writer run on a temporary folder.
A few tests run the `muffin-top` command itself with the real engine on a Book.

## Later

- `explain`, as `rustc --explain`: `muffin-top explain` lists every directive with its one-line
  description and ends with a hint to run `explain @name`; `muffin-top explain @subset` or
  `explain core/one-id-field` prints a directive's usage, arguments and an example, or what a rule
  checks and how to fix it.
- One source for every description. A directive's and a rule's description and example live once,
  in the engine, and everything that shows them reads them from there: editor autocomplete, CLI
  help and `explain`, and the reference pages of the website's docs, which are generated from it
  when the site is built, never copied by hand. A check in `pnpm check` fails when the generated
  pages are stale. This is a requirement, not a nicety: the docs and the CLI must never disagree.

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
