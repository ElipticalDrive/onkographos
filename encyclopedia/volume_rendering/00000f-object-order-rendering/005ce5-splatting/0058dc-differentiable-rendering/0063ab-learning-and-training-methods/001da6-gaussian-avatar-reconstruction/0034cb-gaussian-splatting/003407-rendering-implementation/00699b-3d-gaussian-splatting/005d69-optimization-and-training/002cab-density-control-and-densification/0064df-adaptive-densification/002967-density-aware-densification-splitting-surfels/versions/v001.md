# Density-Aware Densification (Splitting Surfels)

Also called: Adaptive Density Control, Screen-space gradient accumulation, Surfel Densification, Surfel Splitting

## Overview
Density-Aware Densification is a mechanism for surfel densification that uses a density-aware splitting criterion based on local inter-sample spacing and a screen-space gradient accumulation strategy based on T-matrix backward passes [S1, unofficial]. It maintains the partition-of-unity assumption by adjusting surfel density as geometry evolves during optimization, similar to 3DGS densification [S1, unofficial]. It replaces the ad-hoc projected 3D position gradient used in 2DGS with screen-space gradients from the T-matrix backward pass and replaces the global scale threshold used in 3DGS with a criterion that adapts to spatially varying overlap conditions [S1, unofficial].

## How it works
**Inputs:**
*   Current set of surfels, each with position $\mathbf{c}$, tangent scales $s_1, s_2$, normalized major tangent axis $\hat{\mathbf{t}}_{max}$, and material attributes.
*   Local mean neighbor distance $\bar{d}$ for each surfel.
*   Densification threshold $c_{thresh}$.

**Outputs:**
*   Updated set of surfels, where selected surfels are replaced by two children.

**Data Structures:**
*   Surfel: Contains position $\mathbf{c} \in \mathbb{R}^3$, tangent scales $s_1, s_2 \in \mathbb{R}$, normalized major tangent axis $\hat{\mathbf{t}}_{max} \in \mathbb{R}^3$, and material attributes (e.g., albedo, normal).

**Steps:**
1.  **Gradient Accumulation:** Compute per-pixel absolute gradient norms directly from the T-matrix backward pass. Accumulate these norms to determine the cumulative screen-space gradient magnitude for each surfel [S1, unofficial].
2.  **Criterion Evaluation:** For each surfel, calculate the densification criterion $\bar{c}$ (derived from the accumulated screen-space gradients). A surfel is marked for splitting if:
    $$ \bar{c} \ge c_{thresh} \quad \text{AND} \quad \max(s_1, s_2) > \bar{d} $$
    The first condition ensures the surfel is in a region requiring refinement based on image-space error. The second condition targets over-reconstructed primitives whose kernel support is too wide relative to the local point distribution [S1, unofficial].
3.  **Splitting:** For each marked surfel:
    *   Identify the major tangent axis $\hat{\mathbf{t}}_{max}$ corresponding to the maximum scale $\max(s_1, s_2)$.
    *   Create two children positioned at:
        $$ \mathbf{c}_{child1} = \mathbf{c} + \frac{s_{max}}{2} \hat{\mathbf{t}}_{max} $$
        $$ \mathbf{c}_{child2} = \mathbf{c} - \frac{s_{max}}{2} \hat{\mathbf{t}}_{max} $$
    *   Set the scale along the split axis ($s_{max}$) for both children to $0.7 \times s_{max}$.
    *   Preserve the orthogonal tangent scale for both children.
    *   Inherit all material attributes from the parent surfel [S1, unofficial].
4.  **Replacement:** Remove the parent surfel and add the two children to the surfel set [S1, unofficial].

**Parameters:**
*   $c_{thresh}$: Densification threshold. Value not given in the sources.
*   Split scale factor: 0.7. This yields enough mutual overlap between siblings to maintain signal continuity [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Over-splitting in smooth regions:** A global scale threshold (as used in 3DGS) over-splits in smooth regions where the local density already satisfies the Nyquist condition. The density-aware criterion avoids this by comparing against local inter-sample spacing [S1, unofficial].
*   **Under-splitting in fine-scale features:** A global threshold under-splits at fine-scale features where the local inter-sample spacing is much smaller than the scene-level threshold. The density-aware criterion adapts to this [S1, unofficial].
*   **Zero gradient surfels:** Surfels with zero cumulative screen-space gradient magnitude are never reached during the backward pass (occluded, outside frustums, or failed cutoff) and should be pruned [S1, unofficial].

## Operating characteristics
*   **Performance:** Full 3DSS configuration with adaptive density control achieves a PSNR-H of 24.32 and Albedo PSNR-L of 41.26, compared to 23.56 PSNR-H and 40.51 Albedo PSNR-L when adaptive density control is disabled [S1, unofficial].
*   **Strengths:** Adapts spatially without requiring scene-dependent parameter tuning. Increases model complexity gradually and keeps both children coplanar with the parent, preserving local surface coherence [S1, unofficial].
*   **Limitations:** Requires T-matrix backward pass for gradient computation. Without adaptive refinement, under-sampled regions retain coarse coverage that limits both geometric detail and material resolution [S1, unofficial].

## Background
Originated in the 3DSS (3D Surface Splatting for Inverse Rendering) paper [S1, unofficial]. It is designed to maintain the partition-of-unity assumption in coverage-based compositing models as geometry evolves during optimization [S1, unofficial].

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
