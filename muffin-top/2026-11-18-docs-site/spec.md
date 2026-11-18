# Muffin Top site: landing page and docs

Agreed with Jason on 2026-11-18, in the project thread "Plan the marketing site and docs", from
mockups of the landing page and of a docs page (`site-mockups/` in the project's files).

## Problem

Muffin Top has a playground and a CLI, but nothing that says what it is, how to install it, or how
to write a schema. People learn the notation only from `docs/notation.md`, which is the engine's
requirements, not a guide. The aim is to send people to the CLI, with documentation good enough that
a newcomer understands each part in a minute.

## Decision

One site, one deploy:

```text
muffin-top.com
├─ /          Landing page   dark only, built by hand
├─ /play      Playground     the app we have, moved under /play
└─ /docs      Docs           Starlight, following the reader's light or dark setting
     ├─ Introduction             what it does, why, the philosophy
     ├─ Getting started          install, first schema, first generate
     ├─ Schema                   resources, fields, references, custom methods, lists
     ├─ Directives               one page per directive        (summary generated)
     ├─ Errors & status codes    @errors, and how each output renders them
     ├─ Outputs                  REST, JSON:API, Google AIP, gRPC, GraphQL
     ├─ CLI                      commands, options, codegen recipes
     ├─ Editor snippets          each snippet's word and what it inserts (generated)
     └─ Problems                 one page per rule code        (later; see Open)
```

There is no download page. The landing page has an install box, and the header links GitHub and
npm.

### Rules

- **Every fact lives in one place.** Docs are Markdown (MDX) in the repo, so a pull request that
  changes behavior changes its docs. The README stays short (one line, install, a ten-line example,
  a link to the docs) and copies nothing from them.
- **Nothing the engine knows is written twice.** A directive's summary and arguments come from the
  engine's `NOTATION_SDL`, which autocomplete and hover already read. The snippet list comes from
  the language service's `snippetsAt`. Example outputs come from rendering the example's SDL. The
  site reads them while it builds.
- **The build fails when the docs drift.** A directive with no page, a page for a directive that
  no longer exists, and an example whose SDL has errors each fail the build.
- **Examples are the backbone.** Each concept is a small Book snippet we write, then the same thing
  in every output as tabs, rendered by the engine, with Copy and "Open in playground".
- **Written for newcomers.** Each explanation assumes the reader doesn't know Muffin Top: it says
  what a list endpoint is before it says what `@filterBy` does to one.
- **Copy is Jason's.** Every word on the landing page lives in one file, which he rewrites. The
  first version is a real draft, not lorem ipsum.

### Landing page

Top to bottom, dark only:

1. **Hero:** the crowned muffin, large; the headline; one line on what it does; Get started; Try
   the playground; the install command with a copy button.
2. **The demo:** a Book schema next to its output, with a tab per output and "Open in playground".
3. **How it works:** one small schema fanning out to the same list call in all five outputs.
4. **Quick hits:** six cards, each linking into the docs: one schema, five outputs; standards built
   in; problems that explain themselves; feeds your codegen; built for agents; free and MIT.
5. **Codegen:** a `package.json` script piping `generate` into openapi-typescript.
6. **Agents:** a before and after. An agent writing routes by hand gets a different shape each run;
   an agent writing SDL gets the standard's routes every run, checked with `muffin-top check`.
7. **Why SDL:** a short paragraph linking to the introduction.
8. **Footer:** Docs, Playground, GitHub, npm, MIT.

### Docs page

Starlight's layout: our table of contents on the left, the page in the middle, "On this page" on
the right, search, and a light/dark toggle. A directive page has a one-line summary, a plain
explanation, an example with output tabs, where the directive may go, related pages, and previous
and next links.

### Look

One theme for the playground and the site. The playground's `muffin.css` (shadcn's CSS variables,
light and dark) moves to a package both import. The landing page is Tailwind with those tokens, as
the playground is. The docs use Starlight's Tailwind plugin, `@astrojs/starlight-tailwind`, with
one small file that points Starlight's accent and gray at our tokens and its `data-theme="dark"`
at our dark values. Nunito Black for headings and the wordmark, as the playground has.

## Parts

| Part                  | Where                     | Does                                                                                                                    |
| --------------------- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Theme                 | `packages/theme`          | The CSS variables, light and dark, and Tailwind's utilities for them, in place of Tailwind's own palette. No code.      |
| Site                  | `apps/site`               | Astro with Starlight. The landing page, the docs, and the playground's build copied under `/play`.                      |
| Landing copy          | `apps/site/src/content/`  | One file holding every word on the landing page, checked by a schema so a missing field fails the build.               |
| `<Example>`           | `apps/site/src/components` | Takes SDL from the page, renders it in each output, and shows tabs. Fails the build when the SDL has an error.          |
| `<Directive>`         | `apps/site/src/components` | Takes a directive's name and shows its summary, arguments and where it may go, read from `NOTATION_SDL`.                |
| Snippet page          | `apps/site/src/pages`     | Lists each snippet's word and the SDL it inserts, from `snippetsAt` at the top of a schema and inside a resource.        |
| Playground link       | shared                    | Writes the `#sdl=…&style=…` fragment the playground opens. The playground's encoder moves where both can import it.     |

## Plan

Small pull requests, each reviewed before the next:

1. Move the theme into `packages/theme`; the playground imports it. Nothing looks different.
2. Add `apps/site` with Starlight, the theme, the sidebar, and empty pages for each section.
3. Add `<Directive>` and a page per directive, with the check that every directive has one.
4. Add `<Example>` and the shared playground link.
5. Add the snippet page.
6. Add the landing page and its copy file.
7. Serve the playground under `/play`, and deploy the site in its place.
8. Write the docs, in passes.

## Open

- **Problems catalog.** A page per rule needs each rule to say what it checks. Rules carry only a
  name and a standard's link today, so this needs a small engine change, designed with Jason
  first.
- **CLI help on the CLI page.** Generating it needs the CLI to export its help text.
- **Hosting.** Moving the playground to `/play` changes the Cloudflare build command, which Jason
  sets.
