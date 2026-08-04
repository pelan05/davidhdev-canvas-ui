# Canvas UI — repo guide

Library of 33 creative canvas/WebGL effect components that run **over live, interactive HTML**, plus the docs site that documents them and a build script that publishes them as a shadcn-compatible registry.

Ships as **source you copy**, not a package. Nothing here is published to npm. `package.json` `name: "canvasui"` is the docs site, not the library.

---

## 1. The core idea (read this first)

Most components use the experimental **html-in-canvas** API (`CanvasRenderingContext2D.drawElementImage` + `<canvas layoutsubtree>` + `canvas.requestPaint()`). The real DOM lives *inside* a canvas element, stays selectable and clickable, and gets painted into a texture every frame. A WebGL2 canvas on top reads that texture and distorts it.

Two consequences that drive nearly every design decision:

1. **The effect never owns the content.** Text stays real text, links stay clickable. The overlay canvas is `aria-hidden` and `pointer-events: none`.
2. **html-in-canvas only exists in Chromium behind `chrome://flags/#canvas-draw-element` (or an origin trial token).** Every component must therefore render *something reasonable* when the API is absent — the "WebGL overlay fallback": the DOM renders normally outside the canvas and the shader runs with `uHasContent = 0`, drawing a synthetic/abstract version of the effect.

Six components (`*Object`) don't touch the DOM at all — they render a GLB/glTF/SVG/image in a three.js studio scene. They work everywhere, no flag needed.

---

## 2. Layout

```
src/lib/<Base>/                 THE LIBRARY. One folder per component.
  <Base>Vanilla.ts              The engine. All canvas/WebGL/three.js logic. 430–2000 lines.
  <Base>.tsx                    React wrapper (thin)
  <Base>.vue                    Vue wrapper
  <Base>.svelte                 Svelte wrapper
  <Base>.solid.tsx              Solid wrapper
  <Base>.preact.tsx             Preact wrapper
src/lib/registry.ts             Component manifest + standalone-source assembly for the registry
src/lib/utils.ts                cn() helper
src/lib/github.ts               star count fetch
scripts/build-registry.mts      Writes public/r/*.json. Runs on `dev` and `prebuild`.

src/demos/<slug>-demo.tsx       Full-page interactive demo per component (wraps children)
src/components/playground/entries/<slug>-entry.tsx
                                Bespoke playground entry for components that can't just wrap a page
src/components/demos/           Control-panel infrastructure (schema, panel UI, color picker, snippet store)
src/components/docs/            Docs page chrome: code tabs, API table, install tabs, sidebar, shiki highlight
src/components/landing/         Marketing page sections
src/components/ui/              shadcn/base-ui primitives
src/hooks/                      use-demo-controls (URL-backed control state), use-preference, use-theme-toggle
src/data/components.ts          Showcase metadata: name, href, description, preview video
src/app/                        Next.js 16 App Router docs site (static export → Cloudflare Workers)
  (shell)/docs/components/<slug>/page.tsx   One page per component
public/assets/videos/<slug>.webm          Looping preview used in cards + sidebar hover
public/r/                       Generated. Do not edit. Gitignored-ish build output.
public/llms.txt                 Hand-maintained AI-facing index
```

**Stack:** Next.js 16 (`output: "export"`, `experimental.viewTransition`), React 19, Tailwind v4 (CSS-first, `@theme inline` in `globals.css`), TypeScript strict, three.js for `*Object` components, `motion` for site chrome only (never inside `src/lib`), nuqs for URL state, shiki for code blocks, Cloudflare Workers via wrangler.

---

## 3. Anatomy of a component

### 3.1 The engine — `<Base>Vanilla.ts`

Self-contained. **Zero imports except `three` for `*Object` components.** No shared utilities, no cross-component imports, ever — the file has to stand alone when a user copies it into their repo. Duplication between engines is intentional and correct.

Fixed export shape:

```ts
export interface <Base>Options { /** JSDoc on every prop — it becomes the docs. */ }
export interface <Base>Elements { source; content; output }   // or { canvas } for *Object
export interface <Base>Instance {
  setOptions: (o: <Base>Options) => void;
  resize: () => void;
  destroy: () => void;
  // optional imperative extras, e.g. splat / splash / impact / melt / burst
}
const DEFAULTS: Required<<Base>Options> = { ... };
export function supportsHtmlInCanvas(): boolean   // html-in-canvas components only
export function create<Base>(elements, options?): <Base>Instance | null
```

Returns `null` when WebGL2 is unavailable — wrappers use that to fall back.

Internal structure, consistent across all 33:

1. GLSL source as template strings (`#version 300 es`, `precision highp float`).
2. `supportsHtmlInCanvas()` — feature-probes `drawElementImage` + `requestPaint` on a throwaway canvas.
3. `create<Base>()`:
   - `getContext("webgl2", { alpha: true, depth: false, stencil: false, antialias: false, premultipliedAlpha: true })`
   - `paintable.onpaint = () => { ctx.reset(); ctx.drawElementImage(content, 0, 0); contentDirty = true; wake(); }`
   - `compile()` / `link()` helpers; uniform locations harvested by iterating `ACTIVE_UNIFORMS`
   - full-screen triangle-strip quad at attribute location 0
   - `dpr()` clamped to `Math.min(devicePixelRatio, 2)`
   - `syncCanvasSize()` sizing both source and output canvases
   - **On-demand render loop**: `frame()` returns early and sets `running = false` when nothing is animating and `!contentDirty`; `start()` restarts it. `wake` is wired to `start`.
   - `ResizeObserver` on output + content, `IntersectionObserver` to pause offscreen, `matchMedia("(prefers-reduced-motion: reduce)")` to stop motion.
   - pointer/scroll listeners attached to `output.parentElement ?? output`
   - `destroy()` disconnects every observer/listener and deletes every GL program, shader, texture, framebuffer, buffer.

### 3.2 The five wrappers

All five are structurally identical per component. They only: create refs, feature-detect, instantiate on mount, forward option changes, destroy on unmount, and render the fixed three-element DOM shape.

React/Preact/Solid pattern:

```tsx
const [initialOptions] = useState(options);   // frozen: engine is created once
const [failed, setFailed] = useState(false);
const supported = useSyncExternalStore(emptySubscribe, supportsHtmlInCanvas, () => false);  // SSR-safe: false on server
const native = supported && !failed;

useEffect(() => { instance = create<Base>({source, content, output}, initialOptions);
                  if (native && !instance) setFailed(true);
                  return () => instance?.destroy(); }, [initialOptions, native]);

useEffect(() => { instanceRef.current?.setOptions(options); });   // no dep array — runs every render
```

DOM shape (`native` decides which branch mounts the content):

```tsx
<div className={className} style={{ position: "relative", ...style }}>
  <canvas layoutsubtree="true" suppressHydrationWarning style={native ? fillAbsolute : {display:"none"}}>
    {native ? <div ref={contentRef} style={fillRelativeAutoOverflow}>{children}</div> : null}
  </canvas>
  {!native ? <div ref={contentRef} style={fillRelativeAutoOverflow}>{children}</div> : null}
  <canvas ref={outputRef} aria-hidden style={{...fillAbsolute, pointerEvents: "none"}} />
</div>
```

Per-flavor notes:

| Flavor | Notes |
| --- | --- |
| React | `// @ts-expect-error experimental html-in-canvas attribute` above `layoutsubtree`. `suppressHydrationWarning`. |
| Preact | Sets `layoutsubtree` via a ref callback (`el.setAttribute`) instead of a JSX prop. Ships its own local `useSyncExternalStore` shim. |
| Solid | `/** @jsxImportSource solid-js */`, `attr:layoutsubtree="true"`, `splitProps`, `children()`, `untrack` for initial options, content root via a signal (`setContentEl`) because it moves between branches. Augments `solid-js` `JSX.ExplicitAttributes`. |
| Vue | `<script setup lang="ts">`, `defineProps<Options>()`, `watch(() => ({...props}), …, {deep:true})`. Re-creates by flipping `native` + `nextTick` when the engine returns null. |
| Svelte | Svelte 5 runes (`$props`, `$state`, `$effect`). A `<script module lang="ts">` block declares `layoutsubtree` on `HTMLCanvasAttributes`. Registry target path is `src/lib/components/canvasui/`, not `components/canvasui/`. |

`FlameWrap` is the one that breaks the mold: its output canvas is inset *negatively* (`top: -reach`, etc.) so flames and glow can spill outside the element bounds. `*Object` wrappers are much simpler — a single canvas, no source/content pair.

---

## 4. The registry

`src/lib/registry.ts` is the single source of truth for what ships.

- `COMPONENTS: Record<slug, { base, description, dependencies?, devDependencies? }>` — the slug is the kebab-case public name; `base` is the PascalCase folder/file name.
- `REGISTRY_ITEMS` = every `<slug>-<flavor>` pair (33 × 6 = 198 items).
- `getRegistryItem(name)` builds one shadcn `registry-item.json`. Solid/Preact get `solid-js`/`preact` prepended to dependencies.
- `getComponentSources(slug)` returns all six flavors, each with the engine **inlined** so the copied file is standalone.

**Inlining rules (`makeStandalone`) — the sharp edge:**

- It regex-deletes the wrapper's `import { … } from "./<Base>Vanilla";` line and splices the engine body in its place. Keep that import exactly in that form (single import statement, path `./<Base>Vanilla`, trailing semicolon).
- For react/solid/preact it also strips a line matching `export type { … };` — keep the re-export on one line.
- Vue: engine goes into a **new** `<script lang="ts">` block prepended before the existing `<script setup>`.
- Svelte: engine is injected into the existing `<script module lang="ts">` block (or one is created).
- `getRegistryItem`'s name regex is `^([a-z-]+)-(react|solid|preact|vue|svelte|vanilla)$` → **slugs must be lowercase kebab-case only**.

`npm run registry` regenerates `public/r/`. It runs automatically before `dev` and `build`. Never hand-edit `public/r/`.

Install name for users: `npx shadcn@latest add @canvas-ui/<slug>-<flavor>`.

---

## 5. Demos, controls, and docs pages

### Control schema
`src/components/demos/control-schema.ts` provides builders: `scrub()`, `toggle()`, `color()`, `radio()`, `custom()`. Each accepts an optional `when: (values) => boolean` for conditional rows. `useDemoControls(CONTROLS)` (`src/hooks/use-demo-controls.ts`) binds the whole schema to the URL query string via nuqs (`history: "replace"`), so **every control state is a shareable link**. Blob/data URLs are held in transient local state instead of the URL.

`<DemoControls title snippet controls rows? portal? onReset? />` renders the floating panel. The `snippet` prop publishes the live config to `snippet-store.ts` so the copy buttons can emit exactly-configured JSX.

### Demo file — `src/demos/<slug>-demo.tsx`
Client component, takes `children`, wraps them in the effect fixed to the viewport, and renders `<DemoControls>`. Calls `useDemoScrollbarGutter()` and attaches the returned setter to the scrolling content div (it fakes a scrollbar, since the real one lives inside the canvas). Color controls are hex strings; convert to `[r,g,b]` 0–1 at the boundary if the engine wants that.

### Docs page — `src/app/(shell)/docs/components/<slug>/page.tsx`
Server component. Exports `metadata`, a `DESCRIPTION`, and an `API_REFERENCE: ApiProp[]` table that **must mirror the engine's JSDoc and `DEFAULTS`**. Awaits `getComponentSources(slug)` → `highlight()` per flavor, then renders the demo wrapping `<ComponentDoc>`. Whole-page demos omit `preview`; boxed ones pass it.

### Playground — `src/components/playground/demos.tsx`
`PLAYGROUND_DEMOS` with `kind: "wrap"` (reuses `src/demos/<slug>-demo.tsx` around `<MockSite />`) or `kind: "custom"` (a bespoke entry under `entries/`, used by all `*Object` components and by effects needing specific content, e.g. `peel`, `flame-wrap`, `particle-reveal`, `decrypt-reveal`). All lazy-loaded with `ssr: false`.

---

## 6. Adding a component — full checklist

Nine places, in order. Missing any one of these produces a component that half-exists.

1. `src/lib/<Base>/<Base>Vanilla.ts` — the engine (JSDoc every option).
2. `src/lib/<Base>/<Base>.tsx`, `.vue`, `.svelte`, `.solid.tsx`, `.preact.tsx` — copy the five wrappers from the closest existing component and rename. Don't write them from scratch.
3. `src/lib/registry.ts` → add the `COMPONENTS[slug]` entry (+ `dependencies: ["three"]`, `devDependencies: ["@types/three"]` if 3D).
4. `src/demos/<slug>-demo.tsx` — demo + control schema.
5. `src/app/(shell)/docs/components/<slug>/page.tsx` — docs page + `API_REFERENCE`.
6. `src/data/components.ts` → `COMPONENTS` entry; add to `LIGHT_VIDEOS` if the preview video is light-on-dark.
7. `public/assets/videos/<slug>.webm` — looping preview.
8. `src/components/docs/docs-sidebar.tsx` → `sections[1].items` (alphabetical by label) and `NEW_HREFS`.
9. `src/components/playground/demos.tsx` → `PLAYGROUND_DEMOS` (+ an `entries/<slug>-entry.tsx` for `kind: "custom"`).

Then the discovery surfaces: `src/app/sitemap.ts`, `src/components/common/site-search.tsx` (its own hardcoded `ENTRIES` array), `public/llms.txt`, and the README table + component-count badge.

> These lists have drifted before. As of now `public/llms.txt` documents 25 of 33 components and omits Preact; `.github/CONTRIBUTING.md` points at `src/components/demos/` for demos when they actually live in `src/demos/`. Worth fixing if you touch them.

**Editing an existing component:** change the engine, then re-check the `API_REFERENCE` table, the demo's control schema ranges/defaults, and the engine JSDoc — those three duplicate `DEFAULTS` and silently rot.

---

## 7. Invariants — violate these and things break subtly

- **Engine owns all logic. Wrappers stay thin.** No effect logic in a wrapper.
- **Engines are dependency-free and cross-import-free.** `three` is the only exception.
- **Premultiplied alpha.** Contexts use `premultipliedAlpha: true`; shaders output `rgb * alpha`. Never un-premultiply in a shader. Safari is the strictest here.
- **Both paths must work.** html-in-canvas *and* the WebGL overlay fallback. Test with the flag off — that's what most visitors get.
- **On-demand rendering.** Stop the RAF loop when nothing is animating; wake on paint, pointer, resize, option change.
- **Honour `prefers-reduced-motion`** (all 33 do) and pause via `IntersectionObserver` when offscreen (all 33 do).
- **`destroy()` must release everything** — observers, listeners, `onpaint`, and every GL object.
- **DPR clamped to 2.**
- **Overlay canvas is `aria-hidden` + `pointer-events: none`.**
- **No `motion`/framer inside `src/lib`.** Site chrome only.
- **`API_REFERENCE` mirrors `DEFAULTS`.**

---

## 8. Commands

```bash
npm install
npm run dev        # registry build, then next dev
npm run registry   # regenerate public/r/ only
npm run build      # prebuild runs registry; static export to out/
npm run preview    # build + wrangler dev
npm run deploy     # build + wrangler deploy
npm run lint
npx tsc --noEmit
```

CI (`.github/workflows/ci.yml`, Node 22) runs exactly `npx tsc --noEmit`, `npm run lint`, `npm run build`. Run all three before opening a PR. There is **no test suite** — verification is visual, in both the html-in-canvas and fallback paths, in Chrome and Safari.

Origin trial token: `NEXT_PUBLIC_HTML_IN_CANVAS_OT_TOKEN` (injected as a `<meta http-equiv="origin-trial">` in `src/app/layout.tsx`). That's how canvasui.dev works in a stock Chrome install.

License is MIT + Commons Clause — free to use, not to resell the library itself.

---

## 9. Conventions

- Options are flat and primitive (`number`, `string`, `boolean`, `[number,number,number]`). Colors are CSS strings in three.js engines, `[r,g,b]` 0–1 in raw WebGL ones.
- JSDoc every option with its unit and range — it is the API docs.
- Descriptions in `registry.ts` end with "The HTML stays interactive. No dependencies." or "Built on three.js."
- Shared ease across the site: `cubic-bezier(0.23, 1, 0.32, 1)`, exported as `const EASE = [0.23, 1, 0.32, 1]` in motion components.
- Engines smooth values with `1 - Math.exp(-delta * k)`, not linear lerps.
- Tailwind v4 only, no config file; tokens live in `@theme inline` in `src/app/globals.css`. Use `oklch()` colors.
- Import order: React/framework → third party → `@/` aliases. `@/*` → `./src/*`.
- Prefer `useSyncExternalStore` over `useEffect` + `useState` for browser-capability and media-query reads (SSR-safe, no hydration flash).
