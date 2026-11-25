# F13LD.bundle — UI audit (stage 1)

Branch `ui-refresh`, base `d833c3c` (v0.4.0). Read-only audit; no code changed.
Screenshots: `docs/ui-audit/` (headless Chrome, swiftshader). Reference: F13LD.tpms v1.4.0, template `liquid-ui/templates/f13ld/`.

## 1. Current state

| Item | Value |
|---|---|
| Version | `v0.4.0` (header, line 110) |
| File | single `index.html`, 2200 lines, 127 KB. No `CLAUDE.md`, no `tests/`. |
| CSS | one `<style>` (8–81), tokens absent, hard-coded hex throughout, many inline styles (homog block 438–465, flip button 405). |
| Markup | header (85–121) · viewer `#cw` (122–143) · one scrolling `#panel` with every group (144–468). `#container` max-width 1100 px, `height:100vh`. |
| App script | 470–1686: shared `F13LD-SHADE v1` (472–519) and `F13LD-VIEW v1` (520–589) blocks, state + `getParams` (591–638), JS SDF (647–821), `Raymarcher` (824–1261), axis gizmo / clip handle (1273–1321), handlers (1325–1340), stiffness glyph (1343–1458), MIL-HS homogenization (1461–1549), cell aspect + export + Mesh/Lab hand-off (1557–1681). |
| Drop-ins | help v1.2 (`#f13h-theme`, `#f13h-style`, `#f13h-script`, 1689–2155) · queue client loader (2156–2199, remote `f13ld-queue.js`, `tool:'f13ld.bundle'`) · view menu (`f13ViewInit`, in the app script). No shared header message (`#hdrMsg`); one `alert()` (2187). |
| Renderer | WebGL2 fragment ray-marcher, one uber-shader for all four structures (`uStructure` branch), every parameter a uniform. 3×3 neighbour union per SDF eval. Shared F13 shading (48-step soft shadow, 5-tap AO). Separate WebGL1 stiffness-glyph canvas `#ellcv`. 2D `#axisCanvas` for axis labels + clip handle. |
| Families | Bundle, Helicoid, Braid, Weave (`#structure` select) — each shows its own control block. |

**Export JSON** (`buildExportJSON`, 1580–1646):

```
{ family:'bundle',
  meta:{ version:'1.1.0', tool:'bundle-builder', timestamp, preset:<structure> },
  surface:{ type:'ihb', structure, topology:'half-solid'|'void'|'sheet', iso_offset, sheet_width|null },
  geometry:{ …per-structure keys, inactive ones null…, scale_xyz:[1,1,Lz/Pxy],
             z_ramp_x, z_ramp_y, z_checker_step },
  homogenization?:{ grid, E_solid_GPa, poisson, volume_fraction, Ex/Ey/Ez/Gxy_GPa, nu_eff, anisotropy, method:'MIL-HS', note } }
```

- `meta.tool` = **`bundle-builder`** — keep it. Already in the template's `LEGACY` map (`recipe-import.js` line 39).
- Top-level `family:'bundle'` **already exists** and must stay byte-identical. Routing depends on both:
  - Mesh `50-recipe-router.js`: `json.family` → `FAMILIES.bundle.fromFamilyField` first; legacy detector `fam-bundle.js` line 11 matches `meta.tool==='bundle-builder'` or `surface.type==='ihb'`.
  - Lab `60-add-design.js` `labRecipeInfo` line 228: `json.family || json.tool` → `'bundle'`; fallback line 237 on `surface.type==='ihb'`.
  - No new tag needed. Do not rename, do not remove `family`.
- Schema unchanged since the initial release (`git log -S` on `z_handed`, `warp_frame`, `scale_xyz`, `z_checker_step`, `version:'1.1.0'` → all `6da5a54`). No older versions to migrate; missing keys → app defaults.

## 2. Findings

Severity: H = blocks the standard, M = visible defect, L = polish.

| # | Issue | Where | Sev | Rule |
|---|---|---|---|---|
| F1 | All groups stacked in one scrolling panel (≈13 groups, up to 39 rows); no sections. | `#panel` 144–468 | H | Lesson 6; one focus per screen |
| F2 | No two-pane desktop: container capped at 1100 px, viewer 1070×352 at 1440×900; model cropped at the top in every default shot. | 10–11, 21; `desktop-1440.png` | H | Layout |
| F3 | Every control native and hand-styled: 2 selects, 29 ranges, 2 numbers, 9 button groups, 4 pseudo-toggles, 5 checkboxes. | 144–468, 551 | H | Lesson 5 |
| F4 | Manual Low/Med/High quality switch; quality bakes `ITERS` into the shader → each switch recompiles. No adaptive scale, no strip rest frame. | 138–142, 861, 1253, 1259, 1328 | H | Lesson 2 |
| F5 | While the first program is still compiling (`this.prog` null) or after a failed link, **every** `setParams` call (each slider tick) submits a new compile. | 1259 (`needRebuild=!prev\|\|!this.prog\|\|…`) | M | Lesson 1 |
| F6 | Main loop re-queues `requestAnimationFrame` forever (wakes 60/s idle; draws only when dirty). | 1261 | M | Lesson 2 (loop sleeps) |
| F7 | Stiffness glyph spins on its own endless rAF after the first homogenization: 40 draws/s idle in swiftshader, also off-screen, ignores Reduce Motion. | 1422, 1451 | M | render-quality §4; Reduce Motion |
| F8 | `#axisCanvas` width/height reassigned on every render (reallocates the 2D buffer each frame). | 1277 | L | Render cost |
| F9 | Slider input goes through a 50 ms debounce before the view updates; with uniform-only changes it only adds lag. | 1325 | L | Respond fast |
| F10 | `getContext('webgl2')||getContext('webgl')`, but the shader is `#version 300 es`: the WebGL1 fallback can never compile. No `webglcontextrestored` handler; loss shows "GPU reset" and stops. Context loss seen in 2 of ~7 swiftshader loads (`Shader error: null` / `Link failed: null`). | 829, 1263 | M | Robustness |
| F11 | Exported homogenization is stale after any edit: `lastHomogResult` is attached to every later export and Mesh/Lab hand-off, whatever changed since the run. | 1634–1644 | M | Recipe correctness |
| F12 | Glyph icons in UI text: `⬡ ▦ ⊕ ▾ ◐ ● ◍ ■ ⚠ ✓ ×` and `?`. | 108, 109, 162–164, 406, 464, 548, 1675, 2028, 2136, 2169 | M | Icons: drawn SVG only |
| F13 | Text below 11 px: 10 px section labels, overlays, quality, version; help engine 8–9 px. Sizes in use: 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18 px — no scale. | 13–20, 23, 31, 46, 1729, 1761, 1777, 1796 | M | Type 11/13/15/20/28 |
| F14 | Contrast: `#555` and `#444` on `#06080f`/`#111118` (status, indicators, homog status, notes ≈2.5:1); version `#2a3a2a` ≈1.4:1. | 13, 16, 20, 437, 442–454, 464, 46 | M | Contrast ≥ 4.5/4.5/3 |
| F15 | 60 targets under 44 px at 390 wide: slider tracks 5 px tall, mode buttons ~20 px, clip pills 27×21, quality 34×19, help (i) 14×14, header pills 31 px. | various | M | 44 px targets |
| F16 | Four competing accents: cyan sliders `#22d3ee`, violet active buttons `#7c6ff7`, green section labels `#18f370`, family purple `#7B4F9E`; help theme `#a78bfa`. | 23, 26–28, 32, 37, 53, 105, 1698 | M | One accent per tool |
| F17 | `100vh`, no `dvh`, no safe-area insets. | 10, 1743 | M | Dynamic viewport units |
| F18 | Warp amplitude/frequency/frame shown disabled (opacity 0.35) when warp is Off instead of hidden. | 218, 1332 | L | Fewer choices, contextual |
| F19 | `@import` for Exo 2 sits after other rules, so it is ignored. | 38 | L | Housekeeping |
| F20 | Help anchor `{row:'#clipPillX'}` resolves to nothing (pills are not in a `.sc-group`/label): 41 cards, 35 (i). | 1817 | L | Lesson 7 |
| F21 | Queue offline fallback uses `alert()`. | 2187 | L | Shared header message |
| F22 | Viewer overlay clutter: shader-status pill, "Ready" status, view, clip pills, quality — five chrome pieces over a 352 px tall view. | 125–142 | L | One focus |
| F23 | Stiffness glyph box is `padding-bottom:100%` of half the panel (≈517 px square at 1440) holding a ~80 px glyph. | 459; `desktop-homog.png` | L | Layout |
| F24 | `_rebuildPending` is set but never read (dead). `slicePos` unused. | 833, 593 | L | Housekeeping |

Checked and fine:
- No horizontal scroll at 390 / 700 / 1024 / 1440 (page and `#panel`; no element past the viewport).
- No page errors desktop or phone, apart from the intermittent swiftshader context loss (F10).
- Slider drag: **0 compiles** (20-tick drag on `#beamR`: 0 compile, 0 link, 18 draws).
- Structure, shape, twist mode, warp, frame, starts, strand count, topology, flip and clip changes: 0 compiles each.

## 3. Proposed section map

Five sections, one at a time (rail on desktop, tab bar on phone). Workflow: choose and size the structure → twist it → tile it → make it solid → analyse. Per-structure rows swap inside a section (existing `#block*` show/hide).

| Section | Icon | Controls (bundle / helicoid / braid / weave) | Max rows |
|---|---|---|---|
| **Structure** | `strands` (new, drawn) | `#structure` + — Bundle: shape, radius, beams/side, spacing, blend k · Helicoid: pitch, thickness, inner radius, outer radius, starts · Braid: count N, braid radius, pitch, fiber radius, blend k · Weave: fiber radius, pitch, amplitude, blend k | 6 |
| **Twist** | `twist` (new, drawn) | Bundle: twist rate, twist mode, warp axis, + amplitude, frequency, frame when warp ≠ Off · Helicoid: column XY, layer Z · Braid: column XY · Weave: none (see decisions) | 6 |
| **Tiling** | `cell` (from TPMS) | Bundle: column gap · Helicoid: blend k, column gap · Braid: column gap · Weave: layer gap — plus z-step ramp X, ramp Y, checker step (not weave) | 5 |
| **Solid** | `volume` (from TPMS) | mode Half-solid/Sheet, flip side (half-solid), sheet width (sheet), iso offset | 3 |
| **Analyze** | `analyze` (from TPMS) | grid, E_s, ν, Run; results (metrics, Zener bar, stiffness glyph); results badge | 4 + results |

Viewer overlay: clip radio X/Y/Z (bottom-left), view menu (top-left). Quality switch removed (adaptive). Shader-status pill → transient header message while compiling; "Ready" status shown only for errors.
Header (`.fh-right`): help trigger, **Recipe** menu (Open recipe…, Export JSON, Queue, Choose queue…, Mesh, Lab), version, tool icon. `#exportBtn`, Mesh, Lab and the queue pills become hidden natives driven by the menu.

## 4. Control → widget map

Totals (grep of the markup + live DOM count, cross-checked): **76 controls** plus **35 help (i)** buttons.

| Type | Count | Grep / DOM |
|---|---|---|
| `<select>` | 2 | `grep -c '<select'` = 2 |
| `input[type=range]` | 29 | `oninput="onP` = 29; DOM 29 |
| `input[type=number]` | 2 | DOM 2 |
| checkbox (view menu, runtime) | 5 | DOM 5 |
| static `<button>` (inline `onclick`) | 32 | 24 in 9 segmented groups + 4 toggles + run + export + Mesh + Lab |
| runtime buttons | 6 | queue main + caret (or `#f13ldq-fallback`), view, view reset, help trigger, help close |
| help (i) `.f13h-i` | 35 | DOM 35 (stay native, help engine) |

### Selects (2)

| Native | Widget | Notes |
|---|---|---|
| `#structure` | dropdown `w-structure` | `relabel`: short labels (Bundle / Helicoid / Braid / Weave); keep descriptions in help. `onchange="onStructure()"` keeps working via the dispatched `change`. |
| `#homoGrid` | dropdown `w-grid`, `compact` | Labels 32³ / 48³ / 64³. Lives inside a `<label>` that the help engine anchors on — keep the label wrapper. |

### Ranges (29) → `slider` (rows `.sc-group.sl[data-w-slider]`)

All push through the native input (`input` event fires the inline `oninput="onP(...)"`). Format = current decimals (3, or 0 for `beamN`).

| Id | Range / step | Detents / notes |
|---|---|---|
| `beamR` | 0.03–0.25 / 0.005 | — |
| `beamN` | 1–4 / 1 | integer; detents 1,2,3,4; format 0 dp |
| `beamD` | 0.10–0.60 / 0.005 | — |
| `bundleGap` | −0.80–0.50 / 0.005 | detent 0 |
| `blendK` | 0–0.08 / 0.002 | — |
| `twistRate` | 0–4 / 0.05 | detent 0 |
| `warpAmp` | 0–0.40 / 0.005 | row hidden when warp Off |
| `warpFreq` | 0.30–4.00 / 0.05 | row hidden when warp Off |
| `hPitch` | 0.3–4.0 / 0.05 | — |
| `hThick` | 0.01–0.15 / 0.005 | — |
| `hInner` | 0.05–0.80 / 0.01 | — |
| `hOuter` | 0.30–2.00 / 0.02 | — |
| `hBlend` | 0–0.08 / 0.002 | — |
| `hGap` | −0.80–1.00 / 0.02 | detent 0 |
| `bRadius` | 0.10–1.50 / 0.02 | — |
| `bPitch` | 0.3–4.0 / 0.05 | — |
| `bFiber` | 0.03–0.30 / 0.005 | — |
| `bBlend` | 0–0.08 / 0.002 | — |
| `bGap` | −0.80–1.00 / 0.02 | detent 0 |
| `wvR` | 0.02–0.25 / 0.005 | — |
| `wvP` | 0.20–1.50 / 0.01 | — |
| `wvA` | 0–0.40 / 0.005 | — |
| `wvBlend` | 0–0.06 / 0.002 | — |
| `wvZGap` | −0.30–0.60 / 0.01 | detent 0 |
| `zRampX` | −3.10–3.10 / 0.05 | detent 0 |
| `zRampY` | −3.10–3.10 / 0.05 | detent 0 |
| `zChkStep` | −3.10–3.10 / 0.05 | detent 0 |
| `sheetWidth` | 0.005–0.08 / 0.002 | row shown in Sheet only |
| `isoOffset` | −0.15–0.15 / 0.002 | detent 0 |

The `.sv` readouts (`beamRv` …) stay as hidden natives: `onP` writes them; the slider shows its own value.

### Segmented button groups (9 groups, 24 buttons) → `segmented` (clicks the native buttons)

Sync on `class` (`.active`).

| Host | Native buttons | Options | Section |
|---|---|---|---|
| `w-shape` | `btnShapeCircle/Rounded/Square` | Circle · Rounded · Square (drop `● ◍ ■`) | Structure |
| `w-twist` | `btnTwistU/C` | Uniform · Checkerboard | Twist |
| `w-warp` | `btnWarpOff/X/XY` | Off · X · XY | Twist |
| `w-frame` | `btnFrameW/F` | World · Fiber | Twist |
| `w-hstarts` | `btnHS1/2/3` | 1 · 2 · 3 | Structure |
| `w-bn` | `btnBN2/3/4` | 2 · 3 · 4 | Structure |
| `w-topo` | `btnTopoHalf/Sheet` | Half-solid · Sheet | Solid |
| clip | `clipPillX/Y/Z` | → `radio` (template `CONFIG.clip`, axis accents, allowNone); `get:()=>clipState.axis`, `set:n=>onClipPill(n)` semantics need a set-to-value wrapper (`onClipPill` toggles) | viewer |
| quality | `btnQLow/Med/High` | **remove** (adaptive quality, lesson 2); keep natives hidden until stage 4 lands | — |

### Toggles (4 buttons) → `toggle`

These are `.toggle-btn` / plain buttons with no `role=switch`. Stage 2 adds `role="switch" aria-checked` (kept in sync in the existing handlers) so the template's `switches` path works.

| Native | Widget | Label | Section |
|---|---|---|---|
| `btnHColHand` | `w-hcol` | Checkerboard column XY | Twist |
| `btnHZHand` | `w-hz` | Alternating layer Z | Twist |
| `btnBColHand` | `w-bcol` | Checkerboard column XY | Twist |
| `btnFlip` | `w-flip` | Flip side (drop `◐` glyph and its rotate) | Solid |

### Action buttons (10)

| Native | Widget / treatment |
|---|---|
| `runHomoBtn` | `progressButton` `w-run` (`CONFIG.run`); report via `F13UI.onAnalyze('running'/'new'/'stale')` |
| `exportBtn` | Recipe menu → Export JSON (`CONFIG.recipe.exportBtn`); native hidden. Its "Copied + downloaded ✓" text already matches the template's `/^Copied/` watch. |
| `.fh-mesh-btn` Mesh (`openInMesh`) | Recipe menu → Mesh |
| `.fh-mesh-btn` Lab (`openInLab`) | Recipe menu → Lab |
| `.f13ldq-main` / `.f13ldq-caret` / `#f13ldq-fallback` | Recipe menu → Queue / Choose queue… (client still mounts them into `.fh-right`; hidden) |
| `.f13v>button` view | native, restyled, drawn icon instead of `◐`; 44 px |
| `.f13v-r` reset | native, restyled (template resyncs toggles after it) |
| `.f13h-trigger` | native (help engine), drawn icon, 44 px |
| `.f13h-close` | native (help engine), drawn icon |
| `.f13h-i` ×35 | native (help engine) — count must stay 35 |

### Number inputs (2) — stay native

| Native | Reason |
|---|---|
| `homoEs`, `homoNu` | typed material constants (TPMS kept `Es`/`nu` native too). Restyle: tokens, 15 px, 44 px tall, `inputmode="decimal"`, wider than 56 px. |

### Checkboxes (5) → `toggle` (`CONFIG.viewMenu:true`)

Shadows, Occlusion, Warm / cool light, Shade cut faces, Lime edges — label-first menu toggles.

## 5. Render cost

- **Build:** `_buildFrag(quality)` concatenates one GLSL ES 3.00 shader for all four structures; `ITERS` (110 / 220 / 380) is the only baked value. Every geometry, mode and clip value is a uniform set in `render()` (1260).
- **Compile call sites:**
  1. `Raymarcher.setParams` → `_submitShader` (1259): first call, quality change, **and every call while `this.prog` is null** (F5).
  2. `rebuildStiffnessShader` (1426), once per homogenization run (glyph constants baked; fine).
- **Slider ticks:** no recompile (measured 0 over a 20-tick drag) — lesson 1 holds except for F5.
- **Tiers:** manual Low/Med/High (`ITERS` 110/220/380), all at full canvas resolution (DPR capped at 1.5). Only interactive reduction: soft shadows off while dragging (`uF13Interact`, view block).
- **Loop:** rAF runs forever, draws only when `_dirty`/compiling (0 draws/s idle, but 60 wakeups/s). Stiffness glyph: separate endless rAF, draws every frame (F7). Axis canvas reallocated every render (F8).
- **Cost (swiftshader, 360×287 canvas, ms/frame, noisy):** bundle low 1252 · med 1828 · high 1459; braid low 956 · high 803. `ITERS` barely matters; the per-hit shading (48-step shadow, AO, 6+6 gradient evals, each a 3×3 neighbour union) dominates. 1440 timing not captured (context lost that run). Compare ratios on GPU in stage 4.

**Changes needed (render-quality.md):**
1. Make `ITERS` a uniform step limit (loop bound 400, `if(i>=uSteps)break`); quality never recompiles. Fix F5: compile only when the generated source changes, never because `prog` is still pending.
2. Two tiers: interactive (~160 steps, shadows/AO off as now, offscreen FBO at adaptive 0.4–0.85, `blitFramebuffer` LINEAR); rest after 150 ms (384 steps, shadows on, full res, DPR ≤ 2) in strips. Canvas size fixed; `antialias:false`.
3. On-demand loop (`requestRender`/`kick`, sleeps idle). Drop the 50 ms debounce (F9).
4. Stiffness glyph: draw on change, ≤ 30 fps spin only while on screen and the main view is still, off under Reduce Motion (TPMS pattern).
5. Axis canvas: size only on resize; redraw labels with the main frame (or into the same pass).
6. `preserveDrawingBuffer:true` is used by `F13LD_snapshot` (2160) to read `#glCanvas`. If the render worker drops it, the snapshot must render a frame first.
7. WebGL2 only (drop the dead WebGL1 fallback) and add a `webglcontextrestored` path (F10).
8. Remove the quality buttons and update the help "quality" card text (no (i) on it, count unaffected).

## 6. Recipe import

- **TOOL:** `id:'bundle-builder'`, `name:'F13LD.bundle'`, `major:1` (`meta.version` `1.1.0`), `looksLike:r=>r.surface&&r.surface.type==='ihb'`, `label:()=>structure name` (bundle / helicoid / braid / weave).
- **plan(r):** read `surface.structure` (fallback `meta.preset`, then `'bundle'`); topology `half-solid` → topo 0 flip 0, `void` → topo 0 flip 1, `sheet` → topo 1 (flip 0); `iso_offset`; `sheet_width` when sheet. Geometry keys are shared across structures and must be routed by structure:

| Key | Bundle | Helicoid | Braid | Weave |
|---|---|---|---|---|
| `column_gap` | `bundleGap` | `hGap` | `bGap` | — |
| `pitch` | — | `hPitch` | `bPitch` | — |
| `fiber_radius` | — | — | `bFiber` | `wvR` |
| `blend_k` | `blendK` | — (`helicoid_blend` → `hBlend`) | `bBlend` | `wvBlend` |

  Bundle-only: `beam_radius`, `beam_shape` (circle/rounded/square), `beams_per_side` (1–4 int), `beam_spacing`, `twist_rate`, `twist_mode` (0/1), `warp_mode` (0/1/2), `warp_amp`, `warp_freq`, `warp_frame` (0/1). Helicoid: `thickness`, `inner_radius`, `outer_radius`, `starts` (1–3), `col_handed`, `z_handed`. Braid: `strand_count` (2–4), `braid_radius`, `braid_col_handed`. Weave: `weave_pitch`, `weave_amplitude`, `weave_layer_gap`. All structures except weave: `z_ramp_x`, `z_ramp_y`, `z_checker_step`.
  Null = not set (keep default). Clamp to each slider's min/max (and snap to step) with a warning; enums out of set → default + warning.
- **Ignore:** `scale_xyz` (derived), `timestamp`, `homogenization` (not restored), unknown keys.
- **apply(P):** structure (`#structure.value` + `onStructure`) → modes via setters `onShape`, `onTwistMode`, `onWarpMode`, `onWarpFrame`, `onHStarts`, `onBraidN`, `onTopoMode` → toggles `onHColHand`, `onHZHand`, `onBColHand`, `onTopoFlip` (toggle-only: call when state differs) → sliders via native `.value` + `input` event (runs `onP`, updates readout) → one rebuild. Every setter calls `regenerate()`; stub it during the batch and call once (or rely on the existing debounce until stage 4 removes it).
- **Clear homogenization on import:** reset `lastHomogResult`, hide `#homoResults`, restore `#exportBtn` label, mark the Analyze badge stale/cleared. Otherwise the next export carries the previous design's results (F11).
- **Round-trip:** export nulls the inactive structures' keys, so the test compares only the active structure; random states must cover all four structures, both topologies, flip, warp modes and the z-step. Ignore `timestamp` and `homogenization`.
- **Setters that exist:** all of the above. Missing: set-to-value for the four toggles and for `onClipPill` (toggles); a small `setToggle(name,v)` helper in stage 5.

## 7. Help system

- 41 cards, **35 (i)** placed. No (i) by design for `es`, `nu`, `run`, `quality`, `export` (no anchor); `clip` has an anchor that resolves to nothing (F20). Keep 35 unless Matt wants the clip (i) fixed (then 36 — a decision).
- Engine hooks that must survive:
  - `ROW_SEL = '.sc-group'` with a child `<label>` (the (i) goes after/inside it).
  - `{sec:'#structure'}`: the nearest preceding `.sec-lbl` sibling of `#structure`'s wrapper (the "structure" heading).
  - `#homoGrid` inside a wrapping `<label>` (fallback path).
  - `.fh-right` (trigger inserted first), `.f13ld-header` (drawer alignment), card ids `#f13h-card-<id>`.
  - Anchor ids: `clipPillX`, `structure`, `btnShapeCircle`, `beamR`, `beamN`, `beamD`, `bundleGap`, `blendK`, `twistRate`, `btnTwistU`, `btnWarpOff`, `hPitch`, `hThick`, `hInner`, `hOuter`, `btnHS1`, `btnHColHand`, `hBlend`, `hGap`, `btnBN2`, `bRadius`, `bPitch`, `bFiber`, `bBlend`, `btnBColHand`, `bGap`, `wvR`, `wvP`, `wvA`, `wvBlend`, `wvZGap`, `zRampX`, `btnTopoHalf`, `sheetWidth`, `isoOffset`, `homoGrid`.
- `#flipRow` is not a `.sc-group`; making it one in stage 2 does not add an (i) (no card anchors it).
- "show" buttons (`CONFIG.helpTargets`): card id → selector above. Per-structure controls are hidden for other structures, so `revealControl` returns false for them; the show button needs either a structure switch or a "Helicoid only" note (decision).
- Card text to update: `quality` (adaptive now), `export` (mentions `⊕ ⬡ ▦` glyphs → Recipe menu).

## 8. Accent colour — to confirm with Matt

| Colour | Hex | OKLCH | Used for |
|---|---|---|---|
| cyan | `#22d3ee` | 0.797 0.134 211.5 | sliders, quality, flip, view checkboxes; also clip axis **X** |
| violet | `#7c6ff7` | 0.623 0.196 283.4 | active segment, run button, toggles on |
| help violet | `#a78bfa` | 0.709 0.159 293.5 | help theme accent |
| family purple | `#7B4F9E` | 0.511 0.128 308.2 | tool name, model albedo, tool icon |
| green | `#18f370` | 0.843 0.232 149.3 | section labels |
| magenta | `#e879f9` | 0.748 0.207 322.2 | clip axis **Y**, flip on |

The brief's guess ("cyan, amber") does not match: there is no amber except clip axis Z `#facc15`.

**Recommended:** lilac **`oklch(0.78 0.14 300)`** (≈ `#c5a2ff`) — the family purple's hue lifted to accent lightness, near the kit's violet (0.80 0.15 290), close to the existing violet buttons and help theme, and clear of clip axis X (cyan). 22° from clip axis Y magenta; the radio's per-axis accents keep their own colours.
Alternative: keep cyan `oklch(0.80 0.134 211.5)` — continuity with today's sliders, but it equals clip axis X and is the likely noise/grain accent.

## 9. Risks

- **Per-structure blocks inside sections.** Twist is empty for Weave; the rail/tab bar items are fixed in the template. The kit's `sections` widget has no `setItems` or per-item hide (API: `get/set/setBadge/setDisabled/redraw/destroy`, `web/widgets/sections.js` 329–345). Hiding the tab for Weave means a kit change in liquid-ui (or destroy/recreate on structure change).
- **Homogenization staleness** (F11) affects export compare: "byte-identical to main" holds only if stage 5 does not change when the homog block is attached. Any fix is a behaviour change Matt should approve.
- **Swiftshader context loss** during verification (frames 0.8–1.8 s at phone size). Long verifier runs may need a smaller viewport or retries; the tool has no restore path today.
- **Snapshot** for the queue reads `#glCanvas` with `preserveDrawingBuffer` — the stage 4 FBO/blit path must keep it working.
- **Toggles without `role=switch`** — stage 2 must add aria state in the existing handlers or the template's switch sync won't fire.
- **Help anchors** are mostly `.sc-group` + `<label>`; the slider wiring moves the label into the widget (template handles it) — verify the 35 (i) after stage 3.
- **Shared blocks** `F13LD-SHADE v1` / `F13LD-VIEW v1` must stay byte-identical across tools; stage 4 changes (shadow/AO tiering) go through uniforms, not edits to the block.
- **Uber-shader cost:** one shader with four structures; branch cost is per pixel. If stage 4 timing disappoints, per-structure shader variants (compile on structure change only) are the next step — a structure change, not a slider tick.

## Decisions for Matt

- Accent: lilac `oklch(0.78 0.14 300)` (recommended) or keep cyan?
- Sections: Structure · Twist · Tiling · Solid · Analyze — OK? For Weave, hide the Twist tab, or keep it with a one-line "no twist controls for Weave"?
- Remove the Low/Med/High quality buttons in favour of automatic quality (as in TPMS)?
- Homogenization in exports: keep attaching the last run even after edits (today), or drop/mark it stale once the design changes?
- On import, clear the old homogenization results (recommended)?
- Fix the missing help (i) on the section plane (35 → 36), or keep the count at 35?
- Help "show" for a control of another structure: switch structure, or just say "Helicoid only"?
- Warp amplitude/frequency/frame: hide when warp is Off (recommended) instead of greying out?
