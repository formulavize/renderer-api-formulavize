# AGENTS.md

Design notes for `@formulavize/renderer-api-formulavize`. Read this before
changing anything about the renderer contract — it is depended on by every
renderer package and by the editor, so a change here is a breaking change
somewhere else.

## Project overview

This package is the **contract between the formulavize editor and any
renderer**. It was extracted from `editor-formulavize`'s `src/rendererApi/`
with its history intact. It exists because a renderer cannot implement a
contract that lives inside the application that consumes it.

It defines:

- `IRenderer` / `RendererProps` — what a drawing component must do and accept
- `RendererDescriptor` / `RendererPlugin` — how a renderer describes itself
- `RendererRegistry` — how a recipe's `^<name>{ }` directive picks one
- `ExportFormat`, `FileExportOptions`, `HeadlessRenderOptions` — export vocabulary
- `RendererCompletions` / `PropertyCompletion` — how a renderer states its
  editor vocabulary without depending on an editor library
- `getRendererDirectiveProperties` — resolving a `^<name>{ }` block's
  properties, with style tags applied in declaration order and inline
  properties winning

## Setup

Node >= 26, pnpm >= 12.

```sh
pnpm install
```

## Build and test commands

```sh
pnpm build         # tsc -p tsconfig.build.json -> dist/
pnpm test          # vitest run
pnpm test:watch    # vitest
pnpm test:format   # prettier . --check
pnpm format        # prettier . --write
pnpm all           # run-s build test
```

## Package boundary

**This package is the contract and nothing else.** Keep out:

- **Any renderer's vocabulary.** Cytoscape style property names, layout keys,
  ELK option prefixes and selector maps belong to the renderer that understands
  them, in `renderer-cytoscape-formulavize`. If a type here has to name a
  concrete renderer, the abstraction is wrong.
- **Editor libraries.** No CodeMirror. `RendererCompletions` exists precisely so
  a renderer can state its vocabulary as plain data (`PropertyCompletion`) and
  the editor can turn it into whatever its completion library wants.
- **`node:*` and DOM APIs.** `HeadlessRenderOptions` describes a headless render
  in the abstract; it returns `Uint8Array` and names no file system.
- **Compiler internals.** `Dag` arrives through the peer dependency; nothing
  here reaches into the AST or the compilation pipeline.

### Dependency rules

`@formulavize/compiler-frontend-fiz` is a **peerDependency**, not a dependency,
so the host supplies exactly one copy. Two copies would make `Dag` two
unrelated types and break every signature across the boundary. It is also a
devDependency so that the build and tests resolve locally.

`vue` is an **optional** peerDependency. It is used by exactly one line —
`import type { Component }` in `types.ts` — and the `import type` is load
bearing: it keeps `vue` out of the emitted JavaScript entirely, which is what
lets the CLI depend on this package and use `RendererRegistry` without
installing Vue. **Do not turn that into a value import.**

## Architecture

Flat `src/`, one module per file, each its own subpath export:

| Module            | Holds                                                                                        |
| ----------------- | -------------------------------------------------------------------------------------------- |
| `exportFormat.ts` | `ExportFormat` (an open string union) — dependency-free                                      |
| `types.ts`        | Every interface in the contract                                                              |
| `registry.ts`     | `RendererRegistry` — framework-free, so the editor and the CLI resolve renderers identically |
| `directives.ts`   | `getRendererDirectiveProperties`                                                             |
| `index.ts`        | The barrel, exposed as the `"."` export                                                      |

`exports` uses the wildcard `"./*": { "types": "./dist/*.d.ts", "import":
"./dist/*.js" }`. Because `src/` is flat and `tsc` emits 1:1 into `dist/`, a new
module is reachable on its own subpath with no per-module bookkeeping — but it
**must also be added to the `index.ts` barrel**, kept alphabetical.

### Why `RendererDescriptor` and `RendererPlugin` are split

A descriptor is plain data and pure functions. A plugin is a descriptor plus a
Vue component. The split is what lets a node-side entry point describe and
resolve a renderer without importing a component that touches `window` on load.
Anything added to the contract that a headless caller could need belongs on the
**descriptor**, not the plugin.

### Conventions

- `src` modules import each other relatively **with `.js` extensions**
  (`./types.js`), which is what makes the emitted ESM resolvable under Node.
  Tests use the `src/*` path alias instead (`src/index`).
- `tsconfig.build.json` resets `"paths": {}` deliberately — emitted JavaScript
  cannot carry a path alias.
- `ExportFormat` is intentionally both a const and a type of the same name, and
  intentionally an open union (`BuiltInExportFormat | (string & {})`): a
  renderer may declare a format of its own. Treat an unrecognized format as a
  runtime possibility, not a type error.

## Testing instructions

`tests/unit/*.test.ts`, run by vitest. The registry tests use stub plugins
rather than any real renderer — keep it that way, or this package grows a
dependency on something it is supposed to be independent of.

## Commit and PR guidelines

Conventional commits; they feed release-please, which opens the version-bump PR.

## Publishing

Manual, as with the other formulavize packages:

```sh
pnpm install && pnpm build && pnpm test
pnpm publish --access public
```

release-please opens the bump PR; the publish itself is a human step. Consumers
resolve this package **from the registry, not through a workspace link**
(`link-workspace-packages` is off in the monorepo overlay), so local edits here
are invisible to the editor and the renderer packages until a publish — or
until a consumer points at `workspace:*` on purpose for local verification.

**This package publishes before its consumers.** The renderer packages and the
editor all peer-depend on it, so a release here goes out first.
