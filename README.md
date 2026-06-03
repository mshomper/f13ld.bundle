# F13LD.bundle

**Twisted fiber bundles, helicoids, braids, and weaves for the F13LD design suite.**

A browser-native designer for interwoven implicit scaffolds — twisted beam arrays, multi-start helicoids, helical braids, and over/under weaves — with a real-time GPU preview, fabric-tensor homogenisation, and one-click handoff to the mesher. No installation, no server, no build step.

**Live tool:** [mshomper.github.io/f13ld.bundle](https://mshomper.github.io/f13ld.bundle/)

---

## What it does

- Designs four families of interwoven scaffold from a single implicit (SDF) engine
- Renders a real-time, GPU-raymarched 3D preview (WebGL) plus a slice/section viewer
- Reports homogenised elastic anisotropy via an MIL-HS fabric-tensor estimate over the `[-π, π]³` domain
- Exports a full-fidelity JSON recipe consumable by every other F13LD tool
- One-click **⬡ Open in F13LD.mesh** to mesh and export a watertight 3MF
- Runs entirely in the browser — single HTML file, no install, no build

---

## Structures

| Structure | Description |
|---|---|
| **Bundle** | A twisted array of beams — configurable cross-section (circle / rounded / square), count per side, spacing, and column gap — twisted about Z. Optional warp bends the bundle along its axis. |
| **Helicoid** | Multi-start helicoid sheets defined by pitch, thickness, inner/outer radius, and start count. **Alternating** reverses the twist chirality every other period as a smooth spring/perversion (no seam); **Checkerboard** flips handedness per column. |
| **Braid** | N helical strands wound into a rigid braid — strand count, braid radius, fiber radius, and column gap. **Checkerboard** alternates handedness per column. |
| **Weave** | An over/under woven warp + weft — period, strand radius, crossing amplitude, and vertical gap. |

---

## Controls

### Topology

Every structure passes through a shared topology stage:

- **Sheet / half-solid / solid** surface mode
- **Iso offset** — shifts the isosurface to thicken or thin the structure
- **Sheet width** — shell thickness in sheet mode

### Twist & warp

- **Twist rate** about Z (and a per-column twist mode)
- **Warp** — amplitude, frequency, and frame, to bend the structure along its axis

### Handedness

- **Checkerboard** — flips handedness on alternating columns (bundle/helicoid/braid)
- **Alternating** (helicoid only) — smoothly reverses the twist chirality every other period along Z, producing a spring-like double-back with no seam

### Tiling aids

- **Z-ramp (X/Y)** and **checker step** — vertical offsets that, when set to self-tile, repeat seamlessly through the mesher

### Homogenisation

A collapsible **homogenization (MIL-HS)** panel estimates the structure's directional stiffness using a Mean-Intercept-Length fabric tensor with Hashin–Shtrikman bounds, computed over the periodic `[-π, π]³` domain.

---

## Getting the result into the mesher

Click **⬡ Open in F13LD.mesh** in the header. The current parameters are serialised to a JSON recipe (`surface.type: "ihb"`, `family: "bundle"`), URL-encoded as a `?r=` query parameter, and opened in [F13LD.mesh](https://mshomper.github.io/f13ld.mesh) pre-loaded — no download or copy-paste. The recipe carries the full parameter set for a faithful rebuild plus the homogenisation block, and the resulting URL is bookmarkable and shareable.

In the mesher, the Bundle field is re-scaled to world units, tiled through your domain (or imported shape) via GL_REPEAT, and meshed to a watertight 3MF.

---

## Coordinate system

The SDF is evaluated over a `[-π, π]³` design domain. One structural cell — `nx·d + gap` (bundle), `2·hOuter + hGap` (helicoid), `2·(bRadius + fiber) + bGap` (braid), or `wvP` (weave) in plane — maps to a fixed world span when imported, so the in-plane pitch and the Z-twist period both tile correctly. The `scene()` field is negative-inside with iso / sheet / solid topology folded in, matching the convention shared across the F13LD suite.

---

## Architecture

Single HTML file. No build step, no npm, no server.

- **Real-time preview** is a WebGL 1.0 fragment-shader raymarcher of the implicit field, with a separate CPU section/slice viewer for cross-sections.
- **One implicit engine** drives all four structures; the same `scene()` math is ported verbatim into F13LD.mesh so the preview and the meshed output agree.
- **Homogenisation** runs on the CPU over the periodic domain and surfaces directional moduli alongside the recipe export.

---

## F13LD suite

| Tool | Description |
|---|---|
| [f13ld.tpms-builder](https://mshomper.github.io/f13ld.tpms) | Periodic TPMS surface design with PI-TPMS and FFT-CG homogenisation |
| [f13ld.beam](https://mshomper.github.io/f13ld.beam) | Strut/beam lattice design — multiple unit-cell topologies, 3MF Beam Lattice export |
| **f13ld.bundle** | This tool — twisted fiber bundles, helicoids, braids, and weaves |
| [f13ld.noise](https://mshomper.github.io/f13ld.noise) | Stochastic noise scaffold explorer |
| [f13ld.grain](https://mshomper.github.io/f13ld.grain) | Spinodoid / Gaussian / hyperuniform anisotropic grain fields |
| [f13ld.mesh](https://mshomper.github.io/f13ld.mesh) | Implicit scaffold → watertight 3MF exporter |

All tools share a common JSON recipe schema. A Bundle recipe can be ingested directly by [f13ld.mesh](https://mshomper.github.io/f13ld.mesh).

---

## Credits

Built by [Not a Robot Engineering](https://notarobot-eng.com) as part of the F13LD computational materials design suite.
