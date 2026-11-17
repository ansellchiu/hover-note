# Which frameworks let a page element be traced back to its source file?

Research date: 2026-11-17. Status: findings for a design decision, not a spec.

**Question.** At runtime in a dev build, how can a DOM element be traced to its component name, source file and line, per framework, without hover-note depending on that framework? What is the cheapest design for an optional source-location hint on an Anchor?

**How to read this.** "Sourced" statements carry a link. Paragraphs marked **Synthesis** are my inference from those sources. Nothing here was executed against a running app: every claim comes from reading source, release notes or issue threads. Confidence is stated per section.

## Short answer

1. There are only two families of mechanism. **Runtime internals** (a property the framework leaves on the DOM node in dev) and **build-time stamping** (a plugin writes a `data-*` attribute with `file:line:col`).
2. Runtime internals give file and line for free only in **Svelte** (`__svelte_meta`), and file plus component name in **Vue** (`__vueParentComponent.type.__file`). Both are a few lines to read and both are dev-only.
3. **React 19 no longer gives a file and line cheaply.** `_debugSource` was removed. The replacement, `_debugStack`, is an `Error` whose frames point at bundled code, so a tool needs a source-map fetch and decoder (hundreds of lines) or a dev-server endpoint. The component *name* is still a few lines away.
4. **Solid, Qwik, Astro, Angular templates and plain HTML** have no usable runtime property. They depend on stamped attributes (Qwik and Astro stamp their own in dev; Solid needs a plugin).
5. Every tool surveyed that still works on React 19 either ships a source-map resolver (React Grab via bippy, LocatorJS) or stamps attributes at build time (LocatorJS, code-inspector-plugin, react-dev-inspector, Onlook). click-to-component reads only `_debugSource`.
6. **Recommendation:** the core reads stamped `data-*` attributes, `__svelte_meta`, Vue's `__file`, and the React component name: about 0.9 KB gzipped unminified in a sketch. React 19 file and line, and anything for staging builds, belong to an optional plugin or to an existing third-party stamper. The fallback that always works is a component name and a short text snippet for grep.

## Framework versions checked

| Framework | Current stable on 2026-11-17 | Source |
|---|---|---|
| React | 19.3.1 | `npm view react version`; [releases](https://github.com/facebook/react/releases) |
| Vue | 3.5.45 | `npm view vue version`; [vuejs/core package.json](https://github.com/vuejs/core/blob/main/package.json) |
| Svelte | 5.57.3 | [releases](https://github.com/sveltejs/svelte/releases) |
| Solid | 2.0.0, published today (1.9 was stable until now) | `npm view solid-js` (modified 2026-11-17T05:19Z); [releases](https://github.com/solidjs/solid/releases) |
| Angular | 22.2.6 | `npm view @angular/core version` |
| Astro | 7.4.12 | `npm view astro version`; [releases](https://github.com/withastro/astro/releases) |
| Qwik | 1.20.4 stable; 2.0.0-rc.5 pre-release | `npm view @builder.io/qwik version`; [releases](https://github.com/QwikDev/qwik/releases) |

## Per-framework findings

### Summary table

"Read cost" is the code hover-note would need, not the size of the tools that do it today.

| Framework | Mechanism | Gives | Dev build | Production / staging build | Stability | Read cost |
|---|---|---|---|---|---|---|
| React ≤18 | `el.__reactFiber$<rand>` then `fiber._debugSource` | name, file, line, column | Yes, if the JSX source transform runs | No (`_debugSource` stripped) | Private | ~10 lines |
| React 19 | Same fiber key, then `fiber._debugStack` (an `Error`) | name cheaply; file and line only after source-map resolution | Yes, with limits | Fiber exists; names minified; no stack | Private, changed once already | Name ~10 lines; file and line 500–1000+ lines |
| Vue 3 | `el.__vueParentComponent.type.__file` / `.__name` | name, file; no line | Yes | Only with `__VUE_PROD_DEVTOOLS__`, and then basename only | Private, long-lived | ~3 lines |
| Svelte 4/5 | `el.__svelte_meta.loc` | file, line, column of the element | Yes | No | Private, long-lived, used by Svelte's own inspector | ~2 lines |
| Solid 1.x / 2.0 | None on the DOM node; needs stamped attribute | via plugin: file, line, column | Via plugin | Via plugin if enabled | n/a | attribute read |
| Angular | `ng.getOwningComponent(el)` | component class name; file and line only via private `ɵcmp.debugInfo` | Yes | No (`ng` not published) | `ng.*` is public API; `debugInfo` private | ~3 lines for name |
| Astro | `data-astro-source-file` / `-loc`, stamped by the compiler in dev | file, line, column | Yes, but the dev toolbar removes the attributes after reading them; absent in Astro 7.0–7.3.4 | No | Undocumented internal | attribute read |
| Qwik 1.x | `data-qwik-inspector="file:line:col"` | file, line, column | Yes | No | Internal, user-overridable | attribute read |
| Plain HTML / server templates | Nothing | — | — | — | — | — |

### React

**Finding the fiber (sourced).** react-dom stores the fiber on each host node under `'__reactFiber$' + randomKey`, where `randomKey` is `Math.random().toString(36).slice(2)`; this code is not dev-gated ([ReactDOMComponentTree.js](https://github.com/facebook/react/blob/main/packages/react-dom-bindings/src/client/ReactDOMComponentTree.js)). Tools find it by scanning `Object.keys(el)` for the prefix ([LocatorJS `getReactFiber`](https://github.com/infi-pc/locatorjs/blob/52e6739df2e5043cb5b81a0ecc67474bea4606e3/packages/shared/src/pageProbe.ts#L80-L92)). The alternative is `window.__REACT_DEVTOOLS_GLOBAL_HOOK__.renderers` and `renderer.findFiberByHostInstance(el)`, which only exists if React DevTools or a tool installed the hook before React loaded ([click-to-component](https://github.com/ericclemmons/click-to-component/blob/f8be09c4c1f7228168731c2d7e7dc2352f68f0b7/packages/click-to-react-component/src/getReactInstanceForElement.js), [bippy `getFiber`](https://github.com/aidenybai/bippy/blob/fbafc407ca31ad82f391404c6c6539008f9e0d79/packages/bippy/src/core.ts#L947-L982)). Both tools try the hook first and fall back to the key scan.

**`_debugSource` and its removal (sourced).** Up to React 18 the dev JSX transform passed `__source` (`fileName`, `lineNumber`, `columnNumber`) and React copied it to `fiber._debugSource`. PR [#28265](https://github.com/facebook/react/pull/28265) removed `__self`/`__source` handling "along with all the places using it like the `_debugSource` on Fiber"; the reasons given are that it only works with JSX and that the location is not carried through source maps. The transform may still pass the values; React ignores them. A React maintainer's position on the follow-up issue: "The internal fiber structure is internal and will break between any release" ([#29092](https://github.com/facebook/react/issues/29092)). The request to restore it ([#32574](https://github.com/facebook/react/issues/32574)) was closed by [#35240](https://github.com/facebook/react/pull/35240), which adds a "React Element" pane to React DevTools with an open-in-editor affordance. That is a DevTools feature, not an API a page script can call. [#31981](https://github.com/facebook/react/issues/31981) (opt-in `_debugSource`) is still open.

**What React 19 has instead (sourced).** In dev, each JSX call creates `Error('react-stack-top-frame')` and stores it as `element._debugStack`; the fiber copies it as `fiber._debugStack` alongside `fiber._debugOwner` ([ReactJSXElement.js](https://github.com/facebook/react/blob/main/packages/react/src/jsx/ReactJSXElement.js), [ReactFiber.js](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiber.js)). Two limits are visible in the same file: the real stack is only captured while `recentlyCreatedOwnerStacks++ < ownerStackLimit`, otherwise a shared `unknownOwnerDebugStack` is stored; and `Error.stackTraceLimit` is temporarily set to a small `ownerStackTraceLimit`. The public counterpart, `captureOwnerStack()` (React 19.1+), returns a string, is dev-only, and returns `null` outside render, effects and React event handlers, including in a custom DOM event handler ([docs](https://react.dev/reference/react/captureOwnerStack), [changelog 19.1.0](https://github.com/facebook/react/blob/main/CHANGELOG.md)). It cannot be called for an arbitrary element after the fact, so it is no use to a hover tool.

**Why `_debugStack` is expensive (sourced, then synthesis).** The frames are positions in the code the browser executed. The authors of the affected tools report that these resolve to bundler virtual paths and compiled line numbers, not source lines ([#28265 comment](https://github.com/facebook/react/pull/28265), [#32574](https://github.com/facebook/react/issues/32574)). **Synthesis:** on an unbundled dev server (Vite) the frame URL usually contains the project-relative file path, so the *file* is often recoverable without a source map but the *line* is not; on bundled dev servers (webpack, Turbopack) neither is. I did not test this.

**Production (sourced).** The fiber key still exists. `_debugOwner` is never assigned in production builds, which LocatorJS uses as its dev/prod test ([`isDevFiber`](https://github.com/infi-pc/locatorjs/blob/52e6739df2e5043cb5b81a0ecc67474bea4606e3/packages/shared/src/pageProbe.ts#L94-L97)). Component function names are whatever the minifier left.

**Confidence:** high on the removal and on the shape of `_debugStack`; medium on what stack frames look like per bundler.

### Vue 3

**Sourced.** The renderer defines `el.__vnode` and `el.__vueParentComponent` on every element it mounts, guarded by `__DEV__ || __FEATURE_PROD_DEVTOOLS__` ([renderer.ts, v3.5.43](https://github.com/vuejs/core/blob/v3.5.43/packages/runtime-core/src/renderer.ts)). `@vitejs/plugin-vue` attaches `__file` to the component "during serve for devtools to pickup": the full filename in dev, `path.basename(filename)` when production devtools are enabled, and nothing otherwise ([main.ts](https://github.com/vitejs/vite-plugin-vue/blob/main/packages/plugin-vue/src/main.ts)). LocatorJS reads exactly `parentComponent.type.__file` and `.__name` and reports line 1, column 1 ([vueAdapter.ts](https://github.com/infi-pc/locatorjs/blob/52e6739df2e5043cb5b81a0ecc67474bea4606e3/packages/runtime/src/adapters/vue/vueAdapter.ts)).

**Line numbers need stamping (sourced).** `vite-plugin-vue-inspector` rewrites templates to add `data-v-inspector="<relativePath>:<line>:<column>"` and applies only when `command === 'serve'` ([template.ts](https://github.com/webfansplz/vite-plugin-vue-inspector/blob/65ce7c79703465839ad532a26d32022b296e97b3/packages/core/src/compiler/template.ts), [index.ts](https://github.com/webfansplz/vite-plugin-vue-inspector/blob/65ce7c79703465839ad532a26d32022b296e97b3/packages/core/src/index.ts)).

**Confidence:** high for Vue 3.5 with Vite. Not checked: vue-loader/webpack, and Vapor mode (Vue 3.6), which has no virtual DOM and may not set `__vueParentComponent`.

### Svelte 4 and 5

**Sourced.** In Svelte 5 dev builds the compiler wraps templates with `add_locations`, which assigns `element.__svelte_meta = { parent: dev_stack, loc: { file, line, column } }` to each element ([dev/elements.js](https://github.com/sveltejs/svelte/blob/main/packages/svelte/src/internal/client/dev/elements.js)). Svelte 4 had the same property with an extra `char` field ([issue #1499](https://github.com/sveltejs/svelte/issues/1499), [issue #11389](https://github.com/sveltejs/svelte/issues/11389), which also confirms Svelte 5 restored the element location). It is the element's own location in the `.svelte` file, which is more precise than any other framework offers. LocatorJS reads `loc.file`, `loc.line + 1`, `loc.column + 1` ([svelteAdapter.ts](https://github.com/infi-pc/locatorjs/blob/52e6739df2e5043cb5b81a0ecc67474bea4606e3/packages/runtime/src/adapters/svelte/svelteAdapter.ts)).

**Caveats (sourced).** There is no component *name*, only a file. Locations can be wrong when a preprocessor changes line counts ([#8360](https://github.com/sveltejs/svelte/issues/8360), [PR #18725](https://github.com/sveltejs/svelte/pull/18725), not merged when read).

**Confidence:** high. Whether `line` is 0- or 1-based in Svelte 5 I did not verify; LocatorJS adds 1.

### Solid

**Sourced.** Solid exposes a dev-only `DEV` export with hooks such as `afterCreateOwner`; it is `undefined` in production ([docs](https://docs.solidjs.com/reference/rendering/dev)). The solid-devtools author documents that mapping elements to components means walking `owner.value` and calls it "the only way to map elements to components" and "super brittle" ([solid-usage.md](https://github.com/thetarnav/solid-devtools/blob/main/solid-usage.md)). Its locator relies on a Babel transform that stamps `data-source-loc` ([debugger README](https://github.com/thetarnav/solid-devtools/blob/main/packages/debugger/README.md)). React Grab does the same for Solid with its own Babel plugin and `data-react-grab-source-location="file:line:col"` ([plugin](https://github.com/aidenybai/react-grab/blob/ea4bbec9e80f4802e8ae19ad18431edb9ddbb670/packages/react-grab/solid-source-location-babel-plugin.ts), [reader](https://github.com/aidenybai/react-grab/blob/ea4bbec9e80f4802e8ae19ad18431edb9ddbb670/packages/react-grab/src/utils/resolve-solid-source-location.ts)).

Solid 2.0's compiler can emit component names and `file:line:col` as extra `createComponent` arguments (`sourceNames`, on by default in dev) ([release notes](https://github.com/solidjs/solid/releases), [dev diagnostics doc](https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/08-dev-diagnostics.md)). These label *owners* for diagnostics. I found no documented path from a DOM element to its owner.

**Confidence:** medium. Solid 2.0 went stable the day this was written; I read release notes, not the runtime.

### Angular

**Sourced.** `ng.getComponent(el)` and `ng.getOwningComponent(el)` are public API, published on `window.ng` "when Angular is in development mode and are automatically stripped away from prod mode" ([global_utils.ts](https://github.com/angular/angular/blob/main/packages/core/src/render3/util/global_utils.ts), [discovery_utils.ts](https://github.com/angular/angular/blob/main/packages/core/src/render3/util/discovery_utils.ts)). They return a component *instance*; the name is `instance.constructor.name`. A component definition may carry `debugInfo: { className, filePath?, lineNumber? }`, set by `ɵsetClassDebugInfo` under `ngDevMode` ([definition.ts](https://github.com/angular/angular/blob/main/packages/core/src/render3/interfaces/definition.ts), [set_debug_info.ts](https://github.com/angular/angular/blob/main/packages/core/src/render3/debug/set_debug_info.ts)). That is the class's `.ts` file, not the template line, and `ɵ`-prefixed members are private.

**Confidence:** high on `ng.*`; low on whether `filePath` is populated in a normal `ng serve` build (not tested).

### Astro

**Sourced.** In dev, when the dev toolbar is enabled, Astro asks its compiler to stamp `data-astro-source-file` (absolute path) and `data-astro-source-loc` (`line:col`) on every element; the condition is `viteConfig.command === 'serve'` plus toolbar settings ([issue #18162](https://github.com/withastro/astro/issues/18162) quoting `compile.ts`; [PR #9275](https://github.com/withastro/astro/pull/9275) stopped them appearing in builds). Two things make this unreliable to read:

- Since April 2025 the toolbar's Audit app copies the values into a `WeakMap` and **removes the attributes from the DOM**. The PR calls them "an internal API that we've never documented" and notes no public replacement exists ([PR #13602](https://github.com/withastro/astro/pull/13602)).
- Astro 7's Rust compiler shipped with the option as a stub, so Astro 7.0 through 7.3.4 emitted nothing ([compiler-rs #96](https://github.com/withastro/compiler-rs/issues/96)). Emission returned with `@astrojs/compiler-rs` 0.5.1 in Astro 7.3.5 according to the regression report it caused ([#18162](https://github.com/withastro/astro/issues/18162)). At least one third-party tool gave up and stamps its own attributes on Astro 7 ([Frontman docs](https://frontman.sh/docs/integrations/astro)).

Framework islands inside an Astro page (React, Vue, Svelte) behave like those frameworks.

**Confidence:** medium. The 7.3.5 behaviour comes from an issue thread, and I did not establish *when* the toolbar strips the attributes relative to page load.

### Qwik

**Sourced.** In Qwik 1.x dev, the SSR renderer appends `data-qwik-inspector="<file>:<line>:<column>"` to elements when `qDev && qInspector` ([render-ssr.ts, v1.5.5](https://github.com/QwikDev/qwik/blob/v1.5.5/packages/qwik/src/core/render/ssr/render-ssr.ts)); `qInspector` defaults to development mode only ([vite.ts](https://github.com/QwikDev/qwik/blob/main/packages/qwik/src/optimizer/src/plugins/vite.ts)). A hand-written `data-qwik-inspector` is preserved, so users can override it ([issue #5016](https://github.com/QwikDev/qwik/issues/5016)). Path format has varied between absolute and root-relative ([issue #7135](https://github.com/QwikDev/qwik/issues/7135)). Qwik 2 keeps a click-to-source feature ([changelog](https://github.com/QwikDev/qwik/blob/main/packages/qwik/CHANGELOG.md)).

**Confidence:** high for 1.x; the attribute name in 2.0 RC is unverified.

### Plain HTML and server templates

No mechanism exists. A server template engine could stamp attributes, but I found no convention. The only general hint is content: tag, id, classes and a text snippet that an agent can grep for.

## Build-time stamping tools

All of these write a `data-*` attribute during compilation and a small runtime reads it with `closest()`. They are dev-only by default; each can technically be enabled in any build, which is the only route to source hints on a staging build.

| Tool | Attribute | Value | Frameworks | Source |
|---|---|---|---|---|
| code-inspector-plugin 2.0.9 | `data-insp-path` | file, line, column, tag | Vue 2/3, React, Preact, Solid, Qwik, Svelte, Astro; Vite, webpack, Turbopack, esbuild, mako | [constants](https://github.com/zh-lx/code-inspector/blob/6235e8a97a484237b7308e13d7d700e5e7f3fea4/packages/core/src/shared/constant.ts), [transform](https://github.com/zh-lx/code-inspector/blob/6235e8a97a484237b7308e13d7d700e5e7f3fea4/packages/core/src/server/transform/transform-jsx.ts), [README](https://github.com/zh-lx/code-inspector/blob/6235e8a97a484237b7308e13d7d700e5e7f3fea4/README.md) |
| LocatorJS 2.0.0 (`@locator/babel-jsx`, webpack loader) | `data-locatorjs` (path with location) or `data-locatorjs-id` (id into `window.__LOCATOR_DATA__`) | path-based form is used for Server Components | JSX frameworks | [babel-jsx](https://github.com/infi-pc/locatorjs/blob/52e6739df2e5043cb5b81a0ecc67474bea4606e3/packages/babel-jsx/src/index.ts), [webpack-loader](https://github.com/infi-pc/locatorjs/blob/52e6739df2e5043cb5b81a0ecc67474bea4606e3/packages/webpack-loader/src/index.ts) |
| react-dev-inspector | `data-inspector-relative-path`, `-line`, `-column` | separate attributes | React | [visitor.ts](https://github.com/zthxxx/react-dev-inspector/blob/3292e226597e1bafc11bbeefeded416b02c05c58/packages/babel-plugin/src/visitor.ts) |
| vite-plugin-vue-inspector 7.0.0 | `data-v-inspector` | `relativePath:line:column` | Vue | see Vue section |
| solid-devtools transform | `data-source-loc` | file, line, column | Solid | see Solid section |
| React Grab Solid plugin | `data-react-grab-source-location` | `file:line:col` | Solid | see Solid section |
| Onlook | `data-oid` | opaque id; the id-to-location index is kept outside the page | React/Next.js | [README "How it works"](https://github.com/onlook-dev/onlook/blob/main/README.md), [architecture wiki](https://github.com/onlook-dev/desktop/wiki/Architecture) |

**Onlook detail.** The project describes it as "we inject an attribute into the DOM elements at build-time that points back to the code like a sourcemap" (wiki, desktop era) and "we instrument the code in order to map elements to their place in code" (current README). Secondary analysis says the current web version writes `data-oid` into the JSX files themselves and keeps an index ([DeepWiki](https://deepwiki.com/onlook-dev/onlook/5.1-file-system-architecture)); I did not read that code. An opaque id is useless to hover-note without Onlook's index. **Confidence: low to medium.**

**react-dev-inspector's runtime** still reads `fiber._debugSource` and two levels of `_debugOwner` ([inspect.ts](https://github.com/zthxxx/react-dev-inspector/blob/3292e226597e1bafc11bbeefeded416b02c05c58/packages/inspector/src/Inspector/utils/inspect.ts)); its last commit is January 2025 and its maintainer opened the React issue asking for `_debugSource` back. On React 19 only its Babel-plugin attributes work.

## How four tools implement it today

### React Grab 0.2.0 (via bippy 0.7.3)

React Grab's only runtime dependency is bippy ([package.json](https://github.com/aidenybai/react-grab/blob/ea4bbec9e80f4802e8ae19ad18431edb9ddbb670/packages/react-grab/package.json)). It gets the fiber with `getFiberFromHostInstance` and the location with `getSource` / `getOwnerStack` from `bippy/source` ([context.ts](https://github.com/aidenybai/react-grab/blob/ea4bbec9e80f4802e8ae19ad18431edb9ddbb670/packages/react-grab/src/core/context.ts), [element-adapter.ts](https://github.com/aidenybai/react-grab/blob/ea4bbec9e80f4802e8ae19ad18431edb9ddbb670/packages/react-grab/src/core/element-adapter.ts)).

bippy's documented resolution order ([get-source.ts](https://github.com/aidenybai/bippy/blob/fbafc407ca31ad82f391404c6c6539008f9e0d79/packages/bippy/src/source/get-source.ts)):

1. `_debugSource` (React <19).
2. The fiber's own `_debugStack`, trusted only if React's `react-stack-bottom-frame` sentinel is present, parsed with V8's structured `Error.prepareStackTrace` call sites where available ([parse-debug-stack.ts](https://github.com/aidenybai/bippy/blob/fbafc407ca31ad82f391404c6c6539008f9e0d79/packages/bippy/src/source/parse-debug-stack.ts)).
3. An owned child's `_debugStack` bottom frame.
4. The legacy "throwing trick": re-invoking the component to capture a stack.
5. Searching fetched source maps for the component's function name.

Raw frames are then symbolicated by fetching and decoding source maps. The `bippy/source` directory is about 4,000 lines, of which `symbolication.ts` is 1,048 and `owner-stack.ts` 668 (line counts from the clone at the commit above). React Grab adds Next.js dev-server frame resolution ([next-server-frames.ts](https://github.com/aidenybai/react-grab/blob/ea4bbec9e80f4802e8ae19ad18431edb9ddbb670/packages/react-grab/src/core/next-server-frames.ts)). bippy's README warns "This project may break production apps and cause unexpected behavior" and that "Production builds may omit source information" ([README](https://github.com/aidenybai/bippy/blob/fbafc407ca31ad82f391404c6c6539008f9e0d79/README.md)). React Grab's install docs gate it on `NODE_ENV === "development"`.

### LocatorJS 2.0.0

Three adapters. Svelte and Vue are the direct property reads described above. React has a synchronous path that probes a list of locations (`_debugSource`, `type._source`, `memoizedProps.__source`, `_debugInfo` stacks and more), walking `_debugOwner` ([findDebugSource.ts](https://github.com/infi-pc/locatorjs/blob/52e6739df2e5043cb5b81a0ecc67474bea4606e3/packages/runtime/src/adapters/react/findDebugSource.ts)), and an asynchronous path for React 19 ([clickSourceResolver.ts](https://github.com/infi-pc/locatorjs/blob/52e6739df2e5043cb5b81a0ecc67474bea4606e3/packages/runtime/src/adapters/react/clickSourceResolver.ts)): React DevTools `rendererInterfaces.inspectElement`, then `_debugInfo`, then `_debugStack`, each followed by its own source-map resolver with a hand-written VLQ decoder ([sourceMapResolver.ts](https://github.com/infi-pc/locatorjs/blob/52e6739df2e5043cb5b81a0ecc67474bea4606e3/packages/runtime/src/adapters/react/sourceMapResolver.ts), 486 lines), falling back to Next.js's internal `/__nextjs_source-map` and `/__nextjs_original-stack-frames` endpoints. The React adapter directory is about 3,000 lines excluding tests. Independently of all that, its Babel plugin and webpack loader stamp `data-locatorjs*` attributes, which a separate JSX adapter reads ([jsxAdapter.ts](https://github.com/infi-pc/locatorjs/blob/52e6739df2e5043cb5b81a0ecc67474bea4606e3/packages/runtime/src/adapters/jsx/jsxAdapter.ts)).

### click-to-component (click-to-react-component 1.1.3)

Finds the fiber through the DevTools hook, then `_reactRootContainer`, then the `__reactFiber` key scan ([getReactInstanceForElement.js](https://github.com/ericclemmons/click-to-component/blob/f8be09c4c1f7228168731c2d7e7dc2352f68f0b7/packages/click-to-react-component/src/getReactInstanceForElement.js)). The source lookup is `instance._debugSource` and nothing else ([getSourceForInstance.js](https://github.com/ericclemmons/click-to-component/blob/f8be09c4c1f7228168731c2d7e7dc2352f68f0b7/packages/click-to-react-component/src/getSourceForInstance.js)). Last commit September 2025. **Synthesis:** on React 19 it finds components but no source. This is the 20-line design that React 19 broke.

### Agentation 3.1.2

React is an optional peer dependency. Source lookup lives in one 916-line file ([source-location.ts](https://github.com/benjitaylor/agentation/blob/0e3236eb1a0f5577852ab7bb5121f5dd46d42f1d/package/src/utils/source-location.ts)):

1. Key scan for `__reactFiber$` / `__reactInternalInstance$`.
2. `_debugSource` on the fiber or its `_debugOwner`, walking up.
3. A "React 19" pass that probes guessed keys (`_debugSource`, `__source`, `_source`, `debugSource`) on the fiber and its props. It does not read `_debugStack`.
4. A probe: it swaps React's hooks dispatcher (through `__CLIENT_INTERNALS_DO_NOT_USE_OR_WARN_USERS_THEY_CANNOT_UPGRADE.H`) for a `Proxy` that throws, **calls the component function with `{}`**, and parses the thrown error's stack. No source-map step; the comment states "In dev mode, stack frames already contain original source paths."

The result goes into the agent-facing output as `**Source:** path:line:column` and `**React:** <component names>` ([generate-output.ts](https://github.com/benjitaylor/agentation/blob/0e3236eb1a0f5577852ab7bb5121f5dd46d42f1d/package/src/utils/generate-output.ts)). The README says the open-in-editor action "is hidden when no trustworthy source location is available. React development metadata is best effort" ([README](https://github.com/benjitaylor/agentation/blob/0e3236eb1a0f5577852ab7bb5121f5dd46d42f1d/package/README.md)). There is no Vue, Svelte or Astro path.

**Synthesis:** step 4 executes user component code outside a render, with empty props, to get a hint. A component with side effects in its body, or one that reads a prop before calling a hook, will misbehave or yield nothing. It also needs the page's own `react` module, so it cannot be done from a framework-agnostic script. hover-note should not copy this.

## Recommendation: the cheapest design for an optional source hint

This section is synthesis.

### Shape

One optional object on the Anchor, written once when the Annotation is created, never used to re-find the element:

```
source?: { component?: string, file?: string, line?: number, column?: number, via: string }
```

`via` records which mechanism produced it, so an agent can judge how much to trust it and so stale readers can be diagnosed.

### What the core reads (zero dependencies)

Walk from the target element up through its ancestors and stop at the first hit:

1. **Stamped attributes.** A hover-note-owned attribute first (so any server template or build plugin can opt in), then the known third-party ones: `data-astro-source-file` + `-loc`, `data-qwik-inspector`, `data-insp-path`, `data-locatorjs`, `data-v-inspector`, `data-source-loc`, `data-inspector-relative-path` + `-line` + `-column`. Store the raw value; do not try to normalise every format.
2. **Svelte:** `el.__svelte_meta.loc` gives file, line, column.
3. **Vue:** `el.__vueParentComponent.type` gives `__file` and `__name`.
4. **React:** find the `__reactFiber$` key, walk `_debugOwner` (falling back to `return`) to the nearest fiber whose `type` is not a string, and take `displayName || name`. If `_debugSource` exists (React ≤18), take file and line too.
5. **Angular (optional, three lines):** `window.ng?.getOwningComponent(el)?.constructor.name`.

A sketch of items 1–4 came to 1,756 bytes, 886 bytes gzipped, before minification. It was syntax-checked only and never run against a real app. Everything is wrapped in one `try`/`catch` and returns `null` on any surprise, because items 2–4 are private internals.

Order matters: attributes first, because they are explicit, carry a line, and are the only thing that survives into a staging build.

### What belongs in an optional plugin

- **React 19 file and line.** This needs `_debugStack` parsing plus source-map fetch and decode, or a call to a dev-server endpoint. The two tools that do it spend roughly 1,500–4,000 lines on it and still special-case Next.js. It cannot fit a 10 KB core and should not try.
- **A build-time stamper**, if hover-note wants its own. The cheaper move is to not write one: document that installing an existing stamper (code-inspector-plugin covers seven frameworks and five bundlers) makes hover-note pick up `file:line:col` automatically, including on React 19 and Solid.
- **Solid, and Astro 7.0–7.3.4**, which have nothing to read without stamping.
- **Path cleanup** (absolute to project-relative) if it needs anything beyond stripping a configured prefix.

### Fallback

When nothing above hits (plain HTML, server templates, production builds, React 19 without a plugin gives name only): store what is always available and greppable. That is the tag, id, a few stable class names, `data-testid`, and the first ~80 characters of the element's own text. For an agent, a component name plus a text snippet is usually enough to find the file with one search; this is also what remains useful when a stored `file:line` has gone stale.

### What not to do

- Do not re-invoke component functions to capture a stack (Agentation step 4, bippy step 4).
- Do not install or patch `__REACT_DEVTOOLS_GLOBAL_HOOK__`. It must be done before React loads, which a drop-in script cannot guarantee, and the key scan works without it.
- Do not parse `_debugStack` frames in the core to guess a file. It yields bundled positions, and a wrong `file:line` is worse for an agent than a correct component name.

## Implications for hover-note

1. **The hint is best-effort by construction.** No framework except Svelte guarantees a file and line from runtime internals, and the largest framework removed its mechanism in the current major. The Anchor schema should treat every field of the hint as optional and should keep the grep fallback even when a hint is present.
2. **The core cost is small and bounded**: under 1 KB gzipped for attributes, Svelte, Vue and React names. It has no dependency on any framework package.
3. **React users get a component name, not a line, unless they add a plugin or a stamper.** This should be stated plainly in the docs, since React is where most users will expect click-to-source parity with React Grab.
4. **Stamped attributes are the common interface.** Reading a short list of existing attribute names gives line-level hints across React 19, Solid, Vue, Qwik and Astro without hover-note shipping a compiler plugin. It is also the only thing that can work on staging, where the project's stated audience includes "dev/staging builds".
5. **Staging and production builds lose almost everything.** Svelte, React and Angular give nothing; Vue gives a basename only if `__VUE_PROD_DEVTOOLS__` is on. Agents reading Annotations made on staging will depend on the text-snippet fallback.
6. **All runtime reads are private API.** Expect silent breakage on framework upgrades. Guard every read, record `via`, and keep the readers small enough to fix in minutes.
7. **Paths are a privacy and portability concern.** Vue's `__file`, Astro's attribute and sometimes Qwik's are absolute paths that include the developer's home directory. With a files-in-repo Backend those would be committed. Hints should be stored project-relative, or the absolute prefix stripped.
8. **A stored `file:line` goes stale** as soon as the file is edited. The hint describes where the element was when the Annotation was made; `component` and the text snippet age better than `line`.

## Open questions

1. Should the core record a low-confidence file (no line) from a React 19 `_debugStack` frame on unbundled dev servers, or leave React 19 at name-only? Needs a test on Vite, Next.js (webpack and Turbopack) and one RSC app.
2. Should hover-note define its own stamped attribute name, or only read others'? Defining one gives server-template users a documented hook at near-zero cost.
3. On Astro, can a script read `data-astro-source-*` before the dev toolbar's Audit app removes them? Timing was not established. If not, Astro needs a stamper or an early-reading snippet.
4. Does Vue Vapor mode (3.6) still set `__vueParentComponent`? Not checked.
5. Does Solid 2.0 expose any element-to-owner link in dev that would let the core read the new `sourceNames` locations? None was found in the release notes; the runtime was not read.
6. Is `ɵcmp.debugInfo.filePath` populated under a standard `ng serve`, and is reading it worth a private-API dependency for a class file path that is not the template line?
7. Who makes paths project-relative: the core (needs a configured root), the Backend, or the agent?
8. Should the hint be refreshed when an Annotation is re-anchored in a later session, or frozen at creation?

## Not verified

- Nothing was run. All behaviour is inferred from source and issue threads; the sketch was not tested on any framework.
- Preact was not researched.
- React: the numeric values of `ownerStackLimit` and `ownerStackTraceLimit`; which React DevTools release ships the Element pane from #35240; what dev stack frames look like per bundler.
- Vue 3.6 release status and Vapor behaviour; vue-loader behaviour.
- Svelte 5 `loc.line` base (0 or 1).
- Astro 7.3.5+ re-emitting the attributes (from an issue thread, not from compiler source or a test).
- Qwik 2.0 RC attribute name.
- Onlook's current instrumentation (README is vague; details come from a secondary source).
- Gzipped sizes of the surveyed tools: the bundle-size service was unreachable from this environment, so only line counts and npm unpacked sizes were available, and unpacked sizes are not comparable to shipped bytes.

## Method notes

Tool repositories were shallow-cloned and read, never installed or run; commit hashes are in the permalinks above. Framework sources were fetched from `raw.githubusercontent.com` at `main` on 2026-11-17 (linked here as `blob/main`, so those links can drift). Versions came from `npm view`. Web search used Parallel until its free tier rate-limited, then Exa; Tavily was not needed.
