# UI update playbook

How the Seemplify marketing site and the Recruiter landing were rebuilt in September 2026: the tools, the optimisations, the mistakes that cost time, and the rules to apply to any site we touch next.

Two sites were changed:

| Site | Path | Stack | Dev port |
|---|---|---|---|
| Seemplify marketing site | `marketing-site/` | Next 15.1, React 19, Tailwind 3.4, framer-motion 11, npm | 3001 |
| Recruiter landing | `recruiter/frontend/` (page at `app/page.tsx`) | Next 16.3, React 19, Tailwind 3.4, framer-motion 12, pnpm | 5000 |

Both are pinned to the same 3D toolchain: `three` 0.185.1, `@react-three/fiber` 9.7.0, `@react-three/drei` 10.7.8.

---

## 1. Tools and libraries

| Tool | Used for | Notes |
|---|---|---|
| **three.js** + **@react-three/fiber (R3F)** | All 3D scenes | R3F canvas is `alpha: true`, transparent over the page. Recruiter scenes use `flat` (no tone mapping) so brand colours stay exact. |
| **@react-three/drei** | `RoundedBox`, `ContactShadows`, `Environment` + `Lightformer`, `PerformanceMonitor`, `Html` | `Html transform` is used only on the marketing hero; see the gotchas in section 5 before using it anywhere else. |
| **cobe** | Marketing "regions" globe | Light and dark palettes; drag to spin; `update()` loop stops when off screen. |
| **ogl** | Aurora layer over the marketing AI section | Port of React Bits "Aurora". |
| **@paper-design/shaders-react** (`MeshGradient`) | Recruiter page backdrop | Installed on the marketing site too but no longer used there. |
| **framer-motion** | Reveals, counters, marquee, parallax, magnetic buttons, `useScroll` for scroll-linked 3D | `useReducedMotion()` gates every 3D scene. |
| **React Bits ports** (MIT + Commons Clause) | Silk, Aurora, ElectricBorder, StarBorder, MagicRings, ShinyText, scroll-velocity marquee, tilted card | Ported into `components/effects` and `components/motion`, rewritten on our tokens. |
| **Canvas-painted textures** (`recruiter/frontend/components/landing/three/scene-textures.ts`) | Every "paper" or "UI" face in the Recruiter scenes | Text drawn in the page's own fonts; nothing in the WebGL layer is DOM. |
| **Playwright** (`@playwright/test`, installed under `recruiter/frontend/node_modules`, launched with `channel: 'chrome'`) | All visual verification | The in-app Browser pane blanks WebGL after scrolling; use Playwright. |
| **`marketing-site/scripts/darken-screenshots.js`** | Generates the `-dark.png` twin of every product capture | OKLab lightness inversion; see section 6. |

---

## 2. Where things live

```
marketing-site/
  app/globals.css                 theme tokens (:root light, [data-theme='dark'])
  app/fluid-scale.css             1061–1440px proportional scaling
  components/three/HeroOrbit.tsx  hero frame: WebGL check, viewport gating, drag input
  components/three/HeroCardsScene.tsx  record-card carousel (drei Html faces on slabs)
  components/globe/MarketsGlobe.tsx     cobe globe
  components/effects/*            AmbientField, AuroraLayer, ElectricBorder, StarBorderLink, MagicRings
  components/motion/*             TextReveal, NumberTicker, Marquee, Parallax, ShinyText, ...
  components/ProductShowcase.tsx  tilting product capture with callouts (ThemedImage)
  components/ThemedImage.tsx      light image + generated dark twin, swapped by CSS
  scripts/darken-screenshots.js   dark twin generator
  public/images/product-showcases/<name>.png and <name>-dark.png

recruiter/frontend/
  app/landing-brand.css           Seemplify skin for the landing (scoped to .landing-seemplify)
  components/landing/three/ShortlistScene.tsx     hero: CV pile to ranked shortlist
  components/landing/three/SchedulerScene.tsx     scheduling board: one slot, three clocks
  components/landing/three/scene-textures.ts      canvas painters + INK tokens
  components/landing/backdrop/ShaderBackdrop.tsx  mesh-gradient page backdrop
```

---

## 3. Design system rules

**Tokens.** Every visual reads `--marketing-*` tokens. Light palette (after the contrast pass):

| Token | Light | Dark |
|---|---|---|
| canvas | `#efece5` | `#0f0e13` |
| surface | `#ffffff` | `#18161f` |
| surface-soft / sunken | `#f6f3ed` / `#e6e1d8` | `#211e29` / `#0b0a0d` |
| text / muted / faint | `#191816` / `#55514b` / `#6d685f` | `#f7f5fa` / `#bbb5c2` / `#948e9a` |
| line / line-strong | `rgba(49,45,57,.16)` / `rgba(49,45,57,.34)` | `rgba(49,45,57,.72)` / `#312d39` |
| brand | `#6a3fe6` | `#8b63ff` |
| positive / warning | `#007a56` / `#9f6200` | `#2ed3a0` / `#f3bd58` |

The Recruiter skin (`landing-brand.css`) defines the same tokens and remaps the page's Tailwind utility classes onto them, scoped to `.landing-seemplify`, so the content markup is untouched. The 3D scenes mirror the tokens in `INK` inside `scene-textures.ts`; keep the two in step.

**Type.** Space Grotesk 600 for headings (tight tracking), IBM Plex Sans for body, IBM Plex Mono for IDs. Canvas text reads the font stacks from the skin's CSS variables and repaints when `document.fonts.ready` resolves.

**Three theme states.** `theme-sync` stamps `data-theme` on `<html>`; canvas visuals subscribe through `useMarketingTheme()` (a MutationObserver on that attribute) because they cannot read CSS variables. Every scene has a light and a dark palette object; never derive one from the other by inversion.

**Light-mode contrast pass** (what fixed "cream on cream"): white surfaces on a slightly deeper canvas, hairlines at 0.16/0.34 alpha, muted text at `#55514b`, a real shadow token, dark chips over hero imagery (`[data-chip='dark']`), framed 3D scenes with a border and lift, sheets with a hairline edge and stronger contact shadows.

**Screenshots.** Real captures only, in a tokenised frame with a light chrome bar, a tilt on hover, and three capability callouts drawn from product data. Dark mode uses the generated twin.

---

## 4. 3D and motion patterns

- **Load lazily, fall back gracefully.** Scenes are `dynamic(() => import(...), { ssr: false })` behind a WebGL check and `useReducedMotion()`. The fallback is a CSS poster with the same silhouette so layout never jumps.
- **Only spend GPU on screen.** An `IntersectionObserver` flips the canvas `frameloop` between `'always'` and `'never'`. Scene clocks are integrated from `delta` inside `useFrame`, so a hidden scene pauses rather than jumping.
- **Adaptive resolution.** `PerformanceMonitor` moves `dpr` between 1.5 and 1.
- **Timelines, not state.** Loops are pure functions of the scene clock (`u = time % CYCLE`), so React never re-renders during animation and every card/sheet/chip computes its pose from `u`.
- **Drag to spin (mouse and touch).** The frame captures the pointer (`setPointerCapture`), accumulates horizontal movement as radians pending for the next frame, keeps the last rate for a fling, and resumes the idle drift after release. `touch-action: pan-y` keeps vertical scrolling alive on touch. Cursor states via `data-dragging`.
- **Hover slows, scroll spreads.** Hover damps the drift; `useScroll` progress feeds a `spreadRef` that widens the ring or sinks the set as the hero scrolls away.
- **Painted faces over DOM faces.** For anything that must sort against 3D geometry, paint it to a canvas texture and put it on a mesh or sprite. DOM faces (drei `Html`) always paint above the WebGL canvas and can never be occluded by it.

---

## 5. Gotchas that cost a day (read before touching drei)

1. **`ContactShadows` must sit at world x = z = 0.** Its blur pass renders a helper plane with an identity matrix while its camera sits at the group's position, so any x/z offset shifts every shadow sideways. Cover off-centre content with a larger `scale`, never with `position`.
2. **drei `Html transform` needs large world units.** It maps one world unit to one CSS pixel at the perspective plane. A scene authored in 2-unit cards sits a few "pixels" from the eye and is magnified ~150×, so Chrome's sub-pixel snapping of the overlay root (a fractional page position such as `top: 170.6px`) becomes a 30–60px offset between the DOM face and its mesh, while `getBoundingClientRect` still reports the correct spot. Render the subtree inside `<group scale={100}>`, move camera, shadows and backdrop by the same factor, raise `far`, and keep `distanceFactor` as computed from the unscaled size.
3. **drei `Html` lags moving meshes by one frame.** It reads `matrixWorld` in its own `useFrame` (priority 0) before three refreshes matrices at render. Move meshes in `useFrame(cb, -2)`, rotate the parent in `useFrame(cb, -1)`, and call `parent.updateMatrixWorld(true)` there.
4. **drei `Html` z-indices.** The default `zIndexRange` of 20 steps rounds every face to the same value, so stacking falls back to DOM order. Use a wide range such as `[1000, 0]`, and give the frame `isolation: isolate` so those values cannot climb above the site header (`z-index: 50`).
5. **Crisp text on transformed DOM.** Chrome rasterises a continuously transformed layer at its layout size; a 280px card shown at 360px is upscaled and blurs. Lay the face out at 2× (`zoom: 2`) and map it back to the same world size so the compositor downsamples.
6. **Slabs behind DOM faces must match the face exactly.** Same footprint, radius and surface colour; otherwise the slab reads as a rim once there is no busy backdrop to hide it.
7. **Transparent materials always paint after opaque ones.** A `depthTest: false` marker still disappears behind a `transparent` slab unless the marker is transparent too (it then sorts by `renderOrder`).
8. **Tone mapping desaturates brand colours.** Recruiter scenes use `<Canvas flat>`; light intensities were retuned (ambient ~1.3–1.5, key ~1.7–2.0) and the scan effect tints paper by multiplying its colour instead of adding light, which would clip to white.
9. **Sub-pixel structure elsewhere.** Ancestor `scale` entrance animations shrink an R3F canvas because it measures its container during the animation; animate `y` or opacity instead.

---

## 6. Screenshots and the dark twins

- Light captures live in `public/images/product-showcases/`. Every `<name>.png` gets a `<name>-dark.png` from:

  ```bash
  node marketing-site/scripts/darken-screenshots.js marketing-site/public/images/product-showcases
  ```

  Pass a filename substring as a second argument to regenerate one capture.
- The generator works in **OKLab** so hue never flips: neutral pixels have their lightness inverted into the violet-black palette with their own tint kept; saturated fills (buttons, badges, charts) are found by erode/dilate, their label holes closed (radius 12), and kept together with everything drawn on them; thin coloured strokes (including ClearType fringes) are desaturated and lifted continuously in chroma; a source whose mean lightness is already below 0.5 is copied through.
- `ThemedImage` renders both images and swaps them with `.marketing-theme-light-only` / `.marketing-theme-dark-only` on `data-theme`. No JavaScript is involved in the swap.
- Always eyeball a regenerated twin. If a label on a coloured fill speckles, the fill was missed as a candidate (chroma under 0.09), not a transform bug.

---

## 7. Verification

- **Use Playwright, not the Browser pane, for WebGL.** Scripts in the session scratchpad captured whole pages (`pshots.js`), timed element shots of looping scenes (`sshots.js`), drag tests, and alignment diagnostics. Launch with `chromium.launch({ channel: 'chrome' })`; set the `seemplify_theme` cookie and `colorScheme` for the theme. Use `headless: false` when checking paint issues; headless Chrome composites differently.
- **Diagnose DOM-in-3D alignment with tints.** Turn faces into red outlines with an injected stylesheet, add a small transparent marker mesh at each origin, and compare in one screenshot. Two plain 2D dots at reported rect centres tell you whether layout or paint is wrong.
- **Check three widths** (1440, 1200, 390) and both themes before pushing. The 1061–1440px band scales with `clamp()` and `vw` so 1200px reads like a smaller 1440px, not a bigger one.
- **Type-check both sites** (`npx tsc --noEmit` / `pnpm exec tsc --noEmit`). The Recruiter repo has pre-existing errors outside the landing; filter for `components/landing` and `app/page.tsx`.
- **Dev-server hygiene.** Many concurrent Playwright sessions during hot reloads can make Next time out a chunk (`ChunkLoadError` on `app/layout`). It is transient; reload.

---

## 8. Guidelines for any website

**Direction**
- Visuals must be tangible and about the product: a record card, a résumé pile, a calendar board, a globe of markets. Abstract "sciency" orbs, floor rings, busy shader backdrops and glossy silk were all rejected by stakeholders. If a backdrop competes with the subject, remove it; a plain surface with a soft brand halo and contact shadows is enough.
- One idea per scene, told as a loop the viewer can read in ten seconds.
- Keep the brand system intact: cream canvas, hairline grid, brand purple, Space Grotesk on IBM Plex Sans. Reskin markup with scoped token remaps rather than rewriting content.

**Build**
- Tokens first, in `:root`, then `[data-theme='dark']`. Give `body` an explicit token background. Design dark as its own palette.
- Every canvas: dynamic import, WebGL check, reduced-motion fallback, viewport-gated frame loop, adaptive dpr, clock-driven timelines.
- Prefer painted textures and sprites to DOM inside 3D. If DOM is unavoidable, apply every item in section 5.
- Keep dependencies pinned across sites (`three`, R3F, drei) so a fix ports cleanly.

**Images**
- Real captures, in a tokenised frame, with generated dark twins; regenerate twins whenever a capture changes.

**Verify**
- Playwright captures at three widths and both themes, headed for anything 3D; check console for errors from your components (ignore the localhost CORS rejections from the visit tracker).
- Push only after the captures look right; commit messages explain the why.

**Contrast checklist (light mode)**
- Secondary text no lighter than `#55514b` on `#efece5`.
- Hairlines at least 0.16 alpha; card borders 0.34.
- Cards white on cream with a real shadow; chips over imagery dark-on-light.
- 3D frames: border, lift, and a defined rim under boards or sheets.
