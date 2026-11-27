# @formulavize/renderer-api-formulavize

The renderer contract for [formulavize](https://github.com/formulavize). It
defines what a renderer must implement to draw a compiled fiz `Dag`, and the
registry that resolves which renderer a recipe selected.

This package holds **only the contract**. It contains no renderer, drawing code,
or vocabulary belonging to any particular renderer.

## Install

```sh
pnpm add @formulavize/renderer-api-formulavize
```

`@formulavize/compiler-frontend-fiz` is a peer dependency — the host supplies
the single copy of it, so that `Dag` is the same type on both sides of the
boundary. `vue` is an **optional** peer: it is needed only to type a renderer's
drawing component, so a non-browser consumer such as the CLI can use the
registry without installing it.

## Every module is reachable on its own subpath

```ts
import { RendererRegistry } from "@formulavize/renderer-api-formulavize/registry";
import type { IRenderer } from "@formulavize/renderer-api-formulavize/types";
```

Or take everything from the package root:

```ts
import {
  ExportFormat,
  RendererRegistry,
  type IRenderer,
  type RendererPlugin,
} from "@formulavize/renderer-api-formulavize";
```

## Writing a renderer

A renderer is a **plugin**: a descriptor plus the drawing component.

```ts
import {
  ExportFormat,
  type RendererDescriptor,
  type RendererPlugin,
} from "@formulavize/renderer-api-formulavize";
import MyRenderer from "./MyRenderer.vue";

// The descriptor: plain data and pure functions, no component.
export const myRendererMeta: RendererDescriptor = {
  name: "mine", // what a recipe writes as '^mine{ }'
  displayName: "My Renderer",
  supportedExportFormats: [ExportFormat.SVG],
};

// The plugin: the descriptor plus the thing that draws.
export const myPlugin: RendererPlugin = {
  ...myRendererMeta,
  component: MyRenderer,
};
```

The component implements `IRenderer` (`updateDag`, `export`) and accepts
`RendererProps` (`dag`, `isDark`).

Hand the plugin to the editor by adding it to its `defaultRenderers.ts`; nothing
else in the app needs to learn your renderer's name.

### Why the descriptor and the plugin are separate

A `RendererDescriptor` is deliberately free of the drawing component so that a
node-side caller (i.e. the CLI) can describe a renderer, resolve it from a
recipe, and ask it to render, without loading a Vue component that would reach 
for `window` the moment it is imported. A renderer that can draw outside a 
browser puts that on its descriptor's optional `renderHeadless`. One that cannot 
simply omits it, and the CLI reports that it cannot draw rather than silently 
falling through to another renderer the recipe never asked for.

### Offering editor completions

A renderer states its own vocabulary through the optional `completions` field,
described in editor-library-agnostic terms (`PropertyCompletion`) so a renderer
never has to depend on CodeMirror:

```ts
completions: {
  styleProperties: (elementType) => [{ name: "stroke-width", detail: "number" }],
  directiveProperties: (declared) => [{ name: "layout", detail: "string" }],
}
```

`directiveProperties` receives the `key: value` pairs the `^mine{ }` block
already declares, so a renderer can narrow what it offers based on them (for
example, offering only the options the chosen `layout` accepts). Omitting
`completions` is normal: the editor then suggests nothing inside your blocks.

## Renderer selection

`RendererRegistry.resolveFor(dag)` decides which renderer draws a recipe:

- **Last declared wins.** If a recipe names several registered renderers, the
  last `^<name>{ }` directive decides, matching the "later declarations
  override earlier" precedence used for style properties.
- **Unregistered names are skipped.** The compiler does not validate a renderer
  name against the registry, so a renderer directive naming an unknown renderer
  is ignored rather than being an error.
- **Absent falls back** to the default (the first registered, unless another is
  named).

## Development

```sh
pnpm install
pnpm build         # tsc -> dist/ (ESM + .d.ts)
pnpm test          # vitest
pnpm test:format   # prettier --check
pnpm all           # build then test
```

Requires Node >= 26 and pnpm >= 12.
