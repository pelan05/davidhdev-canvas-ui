# Animation roadmap — gaps and recommended work

Findings from a full read of `src/lib` (33 engines × 6 flavors), the demo/docs layer, and the registry build. Companion to `CLAUDE.md` / `AGENTS.md`.

Nothing here is committed work — this is a prioritized plan.

---

## Baseline: what's already right

Worth stating, because it constrains everything below. Verified across all 33 engines:

- `prefers-reduced-motion` handled in all 33.
- `IntersectionObserver` pausing in all 33.
- On-demand RAF: loop halts when nothing animates, wakes on paint / pointer / resize / option change.
- `destroy()` releases every observer, listener, `onpaint` hook, and GL object.
- DPR clamped to `min(devicePixelRatio, 2)`.
- Premultiplied-alpha discipline is consistent.

The gaps are **not** hygiene. They're expressive range.

---

## Gap 1 — Engines are more capable than the wrappers expose

Five engines ship imperative methods:

| Engine | Method |
| --- | --- |
| `Liquid` | `splat(x, y, dx, dy)` |
| `Ripple` | `splash(x, y, strength?)` |
| `ForceField` | `impact(x, y)` |
| `Frost` | `melt(x, y)` |
| `Glitch` | `burst()` |

There is no `forwardRef`, `useImperativeHandle`, or `defineExpose` anywhere in `src/lib`. Verified:

```bash
grep -rl "useImperativeHandle\|forwardRef\|defineExpose" src/lib/   # → no matches
```

**Consequence:** in React, Vue, Svelte, Solid, and Preact — five of the six shipped flavors — these methods are unreachable. `Liquid` can only ever be a cursor toy; you cannot splat on a button click.

---

## Gap 2 — No lifecycle; every effect is a steady state

The `Instance` contract is `setOptions` / `resize` / `destroy`. Nothing else, in all 33.

- No `pause()` / `resume()` — the only way to stop an effect is to destroy it and lose its GPU state.
- No `progress`, no `play`/`reverse`, no `onComplete`. Nothing can be sequenced or replayed.
- `ForceField.reveal` (0–1, with a noise-dissolve wavefront) is the **only** progress-driven prop in the library. It works well and is used by exactly one component.
- `setOptions` snaps instantly. There is no tween between two configurations, so the playground's shareable-config URLs can never become shareable *animations*.

---

## Gap 3 — Interaction is pointer-or-nothing

Driver taxonomy across the 33:

| Driver | Count | Components |
| --- | --- | --- |
| Pointer | ~24 | most of the library |
| Scroll (as a driver) | 3 | `Laser`, `ParticleScroll`, `Bend` — each hardcodes its own mapping |
| Time / ambient only | 6+ | `Blaze`, `Glitch`, `VHS`, `FlameWrap`, and the six `*Object` scenes |
| **In-view** | **0** | — |

`IntersectionObserver` appears in all 33 files and is used **exclusively for pausing**, never as a trigger. There is no scroll-*velocity* input, no audio input, no device orientation. `Ripple.trigger: "click" | "hover" | "none"` is the only trigger enum in the library.

---

## Gap 4 — Nothing animates text as text

`DecryptReveal` and `Asciify` are pixel operations on a page texture. They cannot know where a word is, so per-character, per-word, and per-line motion is impossible. For a creative-effects library this is the most conspicuous absence.

---

## Gap 5 — No state-to-state transition

`next.config.ts` sets `experimental.viewTransition: true` and `globals.css` styles `::view-transition-old/new(root)` — but that's the *docs site*. The library has nothing that transitions between two DOM states. No FLIP, no shared-element morph, no route-change effect.

---

## Gap 6 — No adaptive quality, no composition

- DPR≤2 is the only performance lever. No frame-time governor, no dynamic internal-resolution downscale. 33 heavy shaders, all fixed-cost, mostly untested under thermal throttling.
- Two effects cannot nest — each wants to own the html-in-canvas source canvas. There's no documented composition story and no `<CanvasStack>`.

---

## Recommended work, ranked

### ① `progress` + `trigger` as a cross-library convention
**Effort:** medium, spread across all 33 · **Value:** highest in the repo · **New shaders:** none

Add to every `Options` interface:

```ts
/** What drives the effect. */
trigger?: "pointer" | "scroll" | "inview" | "manual" | "auto";
/** 0–1 drive value, used when trigger is "manual" (or as the scroll/in-view target). */
progress?: number;
```

`ForceField.reveal` already is this — generalize it rather than invent it. Each engine already has a single scalar that gates its intensity; this exposes that scalar and lets something other than the pointer write to it.

Payoff: Shatter that assembles as its section scrolls into view. Decrypt scrubbed by scroll position. `ParticleScroll`'s mechanic available on all 33. Three new interaction modes × 33 components, by re-plumbing existing uniforms.

Sequencing note: land it on 3–4 engines first (`Shatter`, `DecryptReveal`, `ParticleReveal`, `Frost`) to settle the semantics, then roll out. Each engine also needs its `API_REFERENCE` table and demo control schema updated — see the checklist in `CLAUDE.md` §6.

---

### ② Ref / handle passthrough in all six flavors
**Effort:** ~half a day, mechanical · **Value:** high · **Unlocks:** already-shipped code

Expose the engine instance from every wrapper, and extend the `Instance` contract while you're in there:

```ts
export interface <Base>Instance {
  setOptions(o: <Base>Options): void;
  resize(): void;
  pause(): void;      // new — stop the loop, keep GPU state
  resume(): void;     // new
  destroy(): void;
  // plus existing per-engine extras: splat / splash / impact / melt / burst
}
```

Per flavor: React/Preact `useImperativeHandle` (React 19 takes `ref` as a plain prop), Solid a `ref` callback or `props.onReady`, Vue `defineExpose`, Svelte an exported instance getter or bindable prop.

`pause()`/`resume()` are cheap — the engines already have `running` / `start()` internals; this is exposure, not new machinery.

Watch the registry inliner: `makeStandalone` in `src/lib/registry.ts` strips a single-line `export type { … };` from react/solid/preact wrappers. Keep that re-export on one line.

---

### ③ `Morph` — a view-transition component
**Effort:** large · **Value:** headline · **Novelty:** only this library can build it

Capture the outgoing DOM as a texture via html-in-canvas, capture the incoming DOM, then dissolve between the two through any existing shader — liquid, shatter, dither, decrypt. Route changes and tab switches that melt, shatter, or decrypt between states.

Every other animation library has to fake this with cloned nodes and `getBoundingClientRect` math. This repo already owns the DOM-as-texture primitive. Pair it with `document.startViewTransition` where available; fall back to a crossfade where not.

Open questions to resolve during design: holding two source canvases alive at once (Gap 6's composition problem in miniature), and what the non-html-in-canvas fallback does.

---

### ④ Kinetic text via DOM metrics
**Effort:** large (one genuinely new engine) · **Value:** high · **Fills:** Gap 4

Use `Range.getClientRects()` to extract per-word and per-character quads from *real* text, then animate those quads on the GPU:

- staggered physics reveal (per word, per line, per character)
- gravity scatter and reassemble
- magnetic repulsion / attraction around the cursor
- variable-font weight waves

Text stays selectable and accessible because you never replace it — you animate over live glyphs. Complements `Asciify` and `DecryptReveal` (pixel-space) instead of duplicating them. Needs a `ResizeObserver` re-measure path and a plan for web-font load timing.

---

### ⑤ Shared input drivers: audio + scroll velocity
**Effort:** small each (~100 lines) · **Value:** good, broad

Two modules that write to uniforms which already exist:

- **`AudioReactive`** — WebAudio `AnalyserNode` → band energy → uniform. `Blaze`, `Liquid`, `GlyphRain`, `ForceField` all light up for free.
- **Scroll velocity** — the classic smear/warp. `Displacement` and `VHS` are one uniform away from it and currently can't do it at all.

These want to compose with ① (`trigger: "manual"` + an external driver writing `progress`), so land ① first.

---

### ⑥ `quality: "auto"`
**Effort:** medium · **Value:** unglamorous but load-bearing

A frame-time governor that drops internal render resolution under sustained load and recovers when headroom returns. This is what makes 33 heavy effects genuinely shippable on mid-range mobile. Currently the only lever is the DPR≤2 clamp.

Constraint: engines are deliberately dependency-free and cross-import-free (see `CLAUDE.md` §7), so this ships as a duplicated ~30-line block per engine, not a shared import. That is consistent with how the library already works — duplication between engines is intentional.

---

## Explicitly not recommended

- **Custom-cursor component.** `Bubble` covers most of it and the space is saturated elsewhere.
- **Multi-effect nesting / `<CanvasStack>`.** The html-in-canvas source-ownership conflict makes it expensive for a narrow payoff. Revisit only if `Morph` (③) produces a reusable multi-source abstraction as a side effect.

---

## Suggested order

1. **② ref passthrough** — half a day, unlocks shipped code, zero risk.
2. **① `progress` + `trigger`** — pilot on 3–4 engines, settle semantics, then roll out.
3. **⑤ input drivers** — cheap once ① exists.
4. **④ kinetic text** — the new engine people write about.
5. **③ `Morph`** — the headline, once ①'s progress plumbing is proven.
6. **⑥ `quality: "auto"`** — before any serious mobile push.

① and ② together are roughly one focused stretch of work and change what the whole library can express.

---

## Housekeeping found along the way

Unrelated to animation, but real:

- `public/llms.txt` documents **25 of 33** components and never mentions Preact.
- `.github/CONTRIBUTING.md` points at `src/components/demos/` for demos; they live in `src/demos/`. (`src/components/demos/` holds control-panel infrastructure.)
- `src/components/common/site-search.tsx` keeps its own hardcoded `ENTRIES` array — a fifth place a new component must be registered, alongside `src/data/components.ts`, `docs-sidebar.tsx`, `sitemap.ts`, and `playground/demos.tsx`. A single generated source for all five would remove a recurring drift bug.
