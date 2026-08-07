# Agent vision sensor and spatial state

*Design overview: a point-sampled, ray-cast retinal sensor and the agent's minimal
point-and-heading state — the connective tissue between `engine.md`'s renderer and
`neural.md`'s cognitive architecture*

## 1. Sensor-to-action pipeline overview

- Environment (engine.md scene: geometry, materials, lighting)
  → Retina point field (this module — N points, ray direction = heading + fixed per-point offset)
  → Ray-scene intersection (engine.md BVH, Phase 3)
  → Surface sample at hit (engine.md beauty/AOV pipeline: tiered colour AOV — §3, Depth (Z), Normals)
  → Packed colour-D-normal signal (this module)
  → Neural circuit (neural.md Retina stage onward → Visual cortex → Neocortex → Basal ganglia)
  → Motor output (neural.md)
  → Position/heading update (this module)
  → Environment (closes the loop)

This module defines only the sensor between an existing renderer and existing cognition: a field of points rigidly attached to the agent, each casting a ray and reading back whatever `engine.md`'s intersection/shading pipeline produces at the hit, packed into the signal `neural.md`'s Retina stage already assumes exists.

Distinct from `engine.md`'s Phase 1 debug camera, despite both being ray-cast viewpoints into the same scene: the debug camera is free-input human instrumentation outside the pipeline it observes; the retina is a mandatory stage driven by the agent's own motor output, and samples many points per step rather than one full-frame image. Both reuse the same BVH and shading code — neither duplicates it.

## 2. Component reference

| Feature | Mechanism | Role / why it matters | Depends on |
|---|---|---|---|
| Retina point-field topology | 2D grid of points in a plane orthogonal to the heading vector, bounded FOV | The agent reorients via heading, so a forward-facing bounded field is sufficient — no spherical coverage needed | — |
| Per-point ray direction | Heading vector rotated by each point's fixed offset within the field | Keeps the field rigidly attached to the agent — turning the heading turns every ray together | Agent spatial state (heading) |
| Ray-scene intersection | Reuses `engine.md`'s BVH unmodified (Phase 3) — one ray per retina point per step | Avoids a second intersection code path; the retina is a consumer of the engine's scene, not a second renderer. Requires the engine to have reached Phase 3 — no unaccelerated fallback | `engine.md` BVH (Phase 3) |
| Surface colour sampling | Reads back the active fidelity tier's AOV(s) at the ray-hit point (§3) | One shading pipeline underlies every tier; which AOV(s) are read is a render-cost/richness knob (§3), not a fixed choice | `engine.md` AOVs per §3's tier table |
| Depth sampling | Reads `engine.md`'s Depth (Z) AOV at the hit point | Agent-relative distance-to-surface per point | `engine.md` Depth (Z) AOV |
| Normal sampling | Reads `engine.md`'s Normals (shading) AOV — normal-mapped shading normal, not pre-perturbation geometric normal | Surface orientation per point, e.g. edge/silhouette cues for `neural.md`'s visual-cortex stage | `engine.md` Normals AOV |
| Signal packing | Per-point tiered colour channels (1–3, depending on the active fidelity tier — §3) + depth + normal concatenated into a fixed-length vector; full field flattened in a fixed point order, one frame per step — no buffering in this module | Defines the literal interface handed onward; `neural.md` §3's 2–4 frame buffer remains the sole buffering point. Depth and normal channels are fixed regardless of tier — only the colour portion varies | `neural.md` Retina stage |
| Agent spatial state | A single 3D position vector plus heading (yaw + pitch, no roll) | Deliberately minimal: everything the retina field needs to be placed/oriented, nothing more | — |
| Per-step state update | Position/heading overwritten each simulation step by `neural.md`'s Motor output stage | Closes the loop concretely: `neural.md` says motor output updates the environment; this defines *what* it updates | `neural.md` Motor output |
| Retina field count | Single retinal field centred on the agent's head | A stereo/multi-eye pair would be redundant with the direct Depth AOV read | — |
| Ray count per point | One ray per point, no jitter | Anti-aliasing, if ever needed, delegates to `engine.md`'s Phase 5 adaptive sampling rather than duplicating it here | `engine.md` Phase 5 (deferred) |

## 3. Retina fidelity tiers

The retina's colour signal (§2's "Surface colour sampling" row) is not fixed to one AOV — it is
one of five selectable fidelity tiers, trading render cost against signal richness. Exactly one
tier is active for the whole retina field at a time, set via runtime config (analogous to
`engine.md`'s Phase 1 AOV selector, but consumed by the sensor rather than displayed for
debugging); tiers are not mixed per-point. Depth and normal sampling (§2) are unaffected by tier
selection — every tier still packs depth + normal per point; only the colour-channel portion of
the packed signal (§2's "Signal packing" row) varies.

| Tier | Signal | Engine AOV(s) read | Colour channels packed | Requires engine phase | Cost / role |
|---|---|---|---|---|---|
| 1 | Luminance | Luminance | 1 | 1 | Cheapest — brightness only, no colour, no edges |
| 2 | Sobel | Sobel / edge (`engine.md` §3) | 1–2 (gradient magnitude, optionally Gx/Gy) | 1 | Edge/gradient signal at luminance's cost tier; no colour information |
| 3 | Albedo | Albedo / base colour | 3 | 2 | Raw material colour, lighting-independent; cheaper than shading |
| 4 | Direct Lighting & Albedo | Beauty (RGB), direct-lighting-only | 3 | 3 | The tier this document originally specified as the sole option: full direct-lit shaded colour, never a flat albedo tap |
| 5 | Full GI | Beauty (RGB), post-GI | 3 | 6 | Richest — indirect lighting included; only reachable once `engine.md` Phase 6 lands |

## 4. Future considerations

Out of scope for §1–§2: the retina and spatial-state design above stands on its own as an abstract point + heading, with no dependency on any particular body or environment-avoidance behaviour. A rigged 3D biped model and a library of motion-capture clips already exist as assets, and a working 2D reflexive obstacle-avoidance prototype already exists separately; this section records how each is expected to attach later, without pulling their concerns into the core design now.

| Consideration | Note |
|---|---|
| Retina anchor | Today the retina point is free-floating. Once attached to the rig, it anchors to the head/eye bone's transform after animation is applied, rather than being set directly. |
| Motor output resolution | §2's "per-step state update" row has motor output write position/heading directly. With the biped, motor output more likely selects/blends mocap clips from the library (a discrete or blended clip-index action space), and position/heading become *derived* from the resulting animation rather than commanded directly. |
| Root motion | Whether mocap clips carry baked-in root motion (position/heading delta comes from the clip itself) or are in-place (movement applied externally, independent of the clip) — changes what "per-step update" even means once this lands. |
| Ground contact | Foot IK / foot-locking likely needed once locomotion is driven by discrete mocap clips rather than continuous physics-driven movement. |
| Action space change | A bigger change than a parameter tweak — the neural circuit's action space may need to shift from continuous position/heading deltas to clip selection/blend weights, with implications back in `neural.md`'s Motor output / Basal ganglia stages. Not resolved here, just flagged. |
| Reflexive obstacle avoidance | A working 2D prototype (a 360° ring of proximity sensors around heading, bearing-weighted into a signed steering correction) bypasses `neural.md`'s cognitive loop entirely as a reflex arc; extending it to 3D and deciding whether it stays a hand-tuned reflex or folds into the cognitive loop is future work. |

Deferred by design: when these are addressed, each will likely warrant its own document section, or its own document, rather than expanding this one further.

## 5. References

- González, Á. (2010). Measurement of areas on a sphere using Fibonacci and latitude–longitude lattices. Mathematical Geosciences, 42(1).
- Curcio, C.A. et al. (1990). Human photoreceptor topography. Journal of Comparative Neurology.
- Itti, L., Koch, C. (2001). Computational modelling of visual attention. Nature Reviews Neuroscience.
- Barlow, H.B., Levick, W.R. (1965). The mechanism of directionally selective units in rabbit's retina. Journal of Physiology.
- Shah, S. et al. (2018). AirSim: High-fidelity visual and physical simulation for autonomous vehicles. Field and Service Robotics.
- Koenig, N., Howard, A. (2004). Design and use paradigms for Gazebo, an open-source multi-robot simulator.
- See `engine.md` §5 for the rendering/ray-tracing literature this module's intersection and shading depend on but do not re-derive.
