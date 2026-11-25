# F13LD.bundle — UI refresh log

| Stage | Date | Branch | Commit | Checks | Open decisions |
|---|---|---|---|---|---|
| 1 · audit | 2026-11-25 | `ui-refresh` | none yet (base `d833c3c`, v0.4.0) | Audit + 11 screenshots (`docs/ui-audit/`). No h-scroll at 390/700/1024/1440. 0 compiles per slider drag. 76 controls + 35 help (i) mapped. | Accent (lilac `oklch(0.78 0.14 300)` vs cyan); section map and Weave's empty Twist tab; remove quality buttons; homogenization in exports after edits; clear homog on import; clip help (i) 35→36; help "show" for other-structure controls; hide warp rows when Off. |
| 2 · layout shell | 2026-11-25 | `ui-refresh` | `ace941c` (v0.4.0, no bump) | 0 page errors desktop + phone. No h-scroll, no text under 11 px, all targets 44 px at 390/700/1024/1440 (5 sections x 4 structures each). Export JSON identical to `main` for 7/7 states (all four structures, warp off/X/XY, sheet, flip). Help (i) 36 (35 + section plane); help "show" 64/64 land on the control (desktop + phone, all 4 structures), 92 hidden-control cases give a "<Structure> only" message. Homogenization cleared on design and structure change (verified). 12 screenshots `docs/ui-audit/stage2-*.png`. | Header wraps at 800 px (TPMS: 560) because Export sits beside Queue, Mesh, Lab until the Recipe menu (stage 3/5). Clip (i) lives in the native `#clipOverlay` label: stage 3 must move label + (i) into `#w-clip`. Stage 3 wire replaces `F13UI.revealControl`: keep the structure / solid-mode note (`F13UI.helpTargets`). Fallback rail and tab bar carry the new `strands` / `twist` icons for the kit `ICONS` map. Quality buttons and shader pill untouched (stage 4). |
| 3 · widgets | 2026-11-25 | `ui-refresh` | `e0b6529` (v0.4.0, no bump) | Kit `f13ld-wave1-kit` c558bea inlined (`script#liquid-ui`), template style + wire, `CONFIG` filled. Mounted: 29 sliders, 2 dropdowns, 7 segmented, 4 toggles (3 handedness + flip), 5 view-menu toggles, 1 clip radio, 1 progress button (Run), rail + tab bar (kit sections, drawn `strands`/`twist`/`cell`/`volume` icons). Widget to app and app to widget: 112 checks + 28 (every mapped control, all pass). Export JSON identical to `main` for 7/7 states driven through widgets (4 structures, warp off/X/XY, sheet, void). 0 page errors desktop + phone; no h-scroll at 390/700/1024/1440 (5 sections x 4 structures each); 0 visible native selects/checkboxes/ranges. Stall stress (10 rounds, 300-1500 ms stalls, outside releases): worst fill 7.2% vs 2.5% at rest, dt bounded, no non-finite uniforms; Reduce Motion: bead scale 1.00-1.05, no overshoot. No WebGL: native controls usable. Help (i) 36 (clip (i) now in `#w-clip`); help "show" 64/64 land on the control (desktop + phone), 92 other-structure cases give the "only" note. 16 screenshots `docs/ui-audit/stage3-*.png`. | Header at 390 px: the version label is hidden (Recipe menu needs the room). Twist tab on Weave is dimmed, not disabled (kit sections has no per-item disable): it opens the note. Swiftshader loses the app's WebGL context under load ("GPU reset", pre-existing F10, stage 4). Quality buttons untouched (stage 4). |

## Decisions (Matt, 2026-11-25)

Shared, all wave 1 tools:
- **Quality:** automatic, as in TPMS. Remove the manual quality control (buttons or slider); its help (i) goes with it.
- **Stale analysis:** clear homogenization / stiffness results when the design changes, and on import. Exports carry results only for the current design.
- **Missing help cards:** add them; the (i) count rises by that number (record the new count).
- **Field sliders that re-bake:** show a low-res preview bake while dragging, full bake on release. Keep it only if it measures faster to first change than today and does not slow the full bake.
- **Accent:** derived from the tool colour in the F13LD brand guidelines v2.0 (hue kept, lifted to L 0.80 for the dark UI). Final value pending Matt.

This tool:
- **Accent (proposed):** purple `oklch(0.80 0.13 308)` ≈ `#d5a6fe`, from brand stroke `#7B4F9E`.
- **Quality buttons:** removed (automatic).
- **Missing section-plane (i):** add it; 35 → 36.
- **Stale homogenization:** cleared on design change and on import.

- Sections Structure · Twist · Tiling · Solid · Analyze. Weave: Twist section disabled with a one-line note. Help "show" for another structure says which structure it belongs to (no structure switch). Warp sliders hidden when warp is Off.

Stage 1 approved by Matt 2026-11-25 ("use your picks").
