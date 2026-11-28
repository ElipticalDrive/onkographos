# NeRF with Learned Background

Also called: Learned Background NeRF

## Overview
A NeRF variant that combines primary scene rendering with a separate learned background rendering pass, blending the results based on the primary scene's opacity [S1, unofficial]. It addresses the limitation of constant background colors by allowing the background to be modeled as a distinct volumetric entity, while still supporting a fallback constant color for rays that do not intersect the background geometry [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Ray origins and directions (`rays`), primary scene model parameters, background model parameters (if separate), and a boolean flag `learned_background`.
- **Outputs:** A dictionary `comp_rgb` containing the final composited RGB color, `opacity` (accumulated alpha), and sample counts.

**Data Structures:**
- `out`: Dictionary from primary scene forward pass containing `comp_rgb`, `opacity`, `num_samples`, `rays_valid`.
- `out_bg`: Dictionary from background forward pass containing `comp_rgb`, `opacity`, `num_samples`.
- `self.background_color`: A constant RGB vector used as a fallback.

**Steps:**
1. **Primary Rendering:** Execute the primary scene forward pass (e.g., `forward_`) to obtain `out`. This yields the primary color `out['comp_rgb']` and primary opacity `out['opacity']` [S1, unofficial].
2. **Background Selection:** Check the configuration flag `self.config.learned_background` [S1, unofficial].
   - If `true`: Execute the background forward pass `self.forward_bg_(rays)` to obtain `out_bg` [S1, unofficial].
   - If `false`: Construct a dummy `out_bg` where `comp_rgb` is set to `self.background_color`, and `opacity` and `num_samples` are zero [S1, unofficial].
3. **Compositing:** Combine the primary and background results. The final `comp_rgb` is calculated as:
   $$ \text{comp\_rgb} = \text{out}['\text{comp\_rgb}'] + \text{out\_bg}['\text{comp\_rgb}'] \times (1.0 - \text{out}['\text{opacity}]) $$
   This assumes the background is rendered behind the primary scene, with the background contribution weighted by the transparency of the primary scene [S1, unofficial].
4. **Aggregation:** Sum or combine `num_samples` and `rays_valid` from both `out` and `out_bg` to reflect the total computational cost and valid ray count [S1, unofficial].

**Parameters:**
- `self.background_color`: A constant color vector. Used when `learned_background` is false or for rays that fail to intersect the background geometry [S1, unofficial].

**Pitfalls:**
- If `learned_background` is enabled but the background model is not properly trained, artifacts may appear in the background region.
- The blending formula assumes the primary scene's opacity correctly represents the transmittance of the foreground. If the primary scene's opacity is inaccurate, the background bleed will be incorrect [S1, unofficial].

## Subtopics

### Sky and Background Supervision
A supervision strategy for textureless sky and background rays using semantic masks and a free-space penalty loss to prevent spherical shell reconstructions [S2]. It uses an L1 loss for rendered color ($L_{COLOR}$) and an eikonal term for SDF regularization ($L_{REG}$) [S2]. A free-space penalty loss ($L_{MASK}$) is applied with a small weight to remove sky while keeping foreground structures intact, as semantic masks for the background are imperfect and may contain foreground structures [S2]. Directly using a background NeRF for foreground-background separation on textureless sky (which lacks motion parallax) leads to reconstructions contained in spherical shells [S2].

## Operating characteristics
**Time and Memory Complexity:**
- Adds the computational cost of a second volume rendering pass (or a constant color lookup if disabled) to the primary scene rendering [S1, unofficial].
- Memory usage increases if a separate background model (e.g., MLP or grid) is stored [S1, unofficial].

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Scales with the complexity of the background model. If a constant color is used, scaling is negligible [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Allows for realistic, non-uniform backgrounds that can be learned from data [S1, unofficial].
- **Limitations:** Requires additional training data or supervision for the background. If not learned, it falls back to a constant color [S1, unofficial].

**When to use it and when not:**
- Use when the background is complex, textured, or varies across views [S1, unofficial].
- Do not use when the background is uniform or when computational resources are extremely limited, as a constant color fallback is available [S1, unofficial].

## Background
Originated as a variant in NeRF-based reconstruction pipelines to handle backgrounds that are not part of the primary scene geometry [S1, unofficial]. It builds on [NeRF Background Rendering](19907-nerf-background-rendering.md) by allowing the background to be learned rather than just using a separate occupancy grid with a constant color [S1, unofficial].

## Sources
[S1] city-super/GSDF, https://github.com/city-super/GSDF, fetched 2026-10-27 (unofficial)
[S2] Neural 3D Reconstruction in the Wild, https://doi.org/10.1145/3528233.3530718, fetched 2026-10-29
