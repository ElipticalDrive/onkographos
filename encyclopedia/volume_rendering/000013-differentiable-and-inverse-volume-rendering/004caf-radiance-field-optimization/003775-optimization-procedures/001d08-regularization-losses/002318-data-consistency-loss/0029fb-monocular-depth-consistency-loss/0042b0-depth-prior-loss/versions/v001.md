# Depth Prior Loss


## Overview
Depth Prior Loss is a regularization term that constrains the geometry of a radiance field by enforcing consistency between rendered depth and monocular depth priors [S1]. It differs from its parent [Monocular Depth Consistency Loss](4668-monocular-depth-consistency-loss.md) by serving as the specific geometric constraint mechanism within that framework, often combined with other losses like $L_{geo}$ to balance initial convergence speed with long-term scale accuracy [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered depth map $D_r$ for a given view; Monocular depth prior $D_m$ for the same view; Camera intrinsics and extrinsics (for reprojection if multi-view, though primarily defined per-view in this context).
*   **Outputs:** Scalar loss value $L_d$.

**Data Structures:**
*   $D_r$: 2D tensor of shape $(H, W)$, representing the expected depth along each ray.
*   $D_m$: 2D tensor of shape $(H, W)$, representing the predicted monocular depth.

**Steps:**
1.  **Alignment:** Compute scale and shift parameters ($s, t$) that minimize the least-squares error between $D_r$ and $D_m$. This is typically done per batch or per image to handle the scale ambiguity of monocular depth [S1, Parent].
    $$ s, t = \text{solve\_scale\_shift}(D_r, D_m) $$
2.  **Transformation:** Apply the scale and shift to the monocular prior to align it with the rendered depth scale:
    $$ \hat{D}_m = s \cdot D_m + t $$
3.  **Loss Calculation:** Compute the difference between the rendered depth and the aligned monocular prior. The specific formulation is a least-squares error:
    $$ L_d = \frac{1}{N} \sum_{i,j} \| D_r(i,j) - \hat{D}_m(i,j) \|^2 $$
    Where $N$ is the number of valid pixels.

**Parameters:**
*   Scale and shift computation method: Closed-form least-squares solution [Parent].

**Edge Cases and Pitfalls:**
*   **Scale Mismatch:** Using $L_d$ alone initially offers good performance but later suffers from higher error due to scale mismatches between the depth prior and voxel resolutions [S1].
*   **Complementarity:** $L_d$ should be combined with other geometric losses (e.g., $L_{geo}$) to achieve optimal results, as $L_d$ provides strong initial constraints while other losses refine depth progressively [S1].

## Subtopics

## Operating characteristics
*   **Strengths:** Provides strong initial geometric constraints, leading to fast convergence in early training stages [S1].
*   **Limitations:** Suffers from scale ambiguity issues over time if used in isolation, leading to increased error in later training stages [S1].
*   **Performance:** When combined with $L_{geo}$, it yields better PSNR, SSIM, and LPIPS scores than using either loss alone, with comparable model size and training time (e.g., ~10 minutes for FrugalNeRF) [S1].
*   **When to use:** Use as part of a combined regularization strategy (e.g., with $L_{geo}$) to leverage its fast initial convergence while mitigating its long-term scale drift. Avoid using it in isolation for final quality.

## Background
Depth Prior Loss is used in few-shot NeRF methods, specifically FrugalNeRF, to enable fast convergence without learned priors [S1]. It was introduced to address the challenge of achieving high-quality novel view synthesis with limited training views by leveraging monocular depth priors as a geometric constraint [S1].

## Sources
[S1] FrugalNeRF: Fast Convergence for Extreme Few-shot Novel View Synthesis without Learned Priors, https://doi.org/10.1109/cvpr52734.2025.01049, fetched 2026-10-29
