# Pixel-GS
Also called: Pixel-based densification, Projected area gradient scaling

## Overview
Pixel-GS is a densification strategy for 3D Gaussian Splatting that modifies the gradient-based splitting criteria to account for the spatial footprint of each Gaussian primitive [S1]. It scales the positional gradient by the number of pixels a Gaussian influences, ensuring that primitives causing error across many pixels are prioritized for refinement [S2, unofficial]. Additionally, it applies a distance-dependent scaling factor to the gradient to suppress the creation of "floaters" in background regions [S1]. This approach yields state-of-the-art rendering quality compared to standard 3DGS, though it typically results in a higher number of Gaussian primitives [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The current set of 3D Gaussians $\mathcal{G}_i$, the accumulated gradient of the loss with respect to the 2D mean position $\nabla_{\mu_{2D}}$, the projected area of each Gaussian in pixel space $A_i$, and the depth $d_i$ of each Gaussian.
*   **Output:** A binary mask or score list determining which Gaussians are split, cloned, or pruned during the densification step.

**Data Structures:**
*   **Gradient Buffer:** Stores $\nabla_{\mu_{2D}}$ for each Gaussian.
*   **Pixel Count Buffer:** Stores $N_{pix, i}$, the number of pixels covered by Gaussian $i$ across all training viewpoints.
*   **Depth Buffer:** Stores $d_i$, the depth of the Gaussian center relative to the camera.

**Method Steps:**
1.  **Accumulate Pixel Coverage:** For each Gaussian $\mathcal{G}_i$, compute the total number of pixels $N_{pix, i}$ it influences across all input viewpoints during the forward pass [S1].
2.  **Compute Base Gradient:** Calculate the gradient norm $\|\nabla_{\mu_{2D}}\|$ for each Gaussian from the differentiable rendering pipeline [Parent].
3.  **Scale by Projected Area:** Multiply the gradient norm by the pixel coverage count $N_{pix, i}$ to obtain the area-scaled gradient $G_{area}$ [S2, unofficial]:
    $$ G_{area, i} = \|\nabla_{\mu_{2D}, i}\| \cdot N_{pix, i} $$
    This ensures that a Gaussian affecting a large number of pixels contributes more to the densification score than one affecting few pixels, even if the per-pixel error is similar [S2, unofficial].
4.  **Apply Depth Scaling:** Scale the gradient by a factor dependent on the depth $d_i$ to suppress densification in distant regions [S1]. Let $S(d_i)$ be the depth scaling function. The final densification score $S_i$ is:
    $$ S_i = G_{area, i} \cdot S(d_i) $$
    The specific form of $S(d_i)$ is not explicitly defined in the provided sources, but it is intended to reduce the likelihood of splitting Gaussians in the background to prevent floaters [S1].
5.  **Thresholding:** Compare $S_i$ against a threshold. If $S_i$ exceeds the threshold, trigger split or clone operations according to the parent [Densification Strategy](6234-densification-strategy.md) principles.

**Parameters:**
*   **Depth Scaling Factor:** Not given in the sources.
*   **Thresholds:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **High Primitive Count:** The method typically results in a higher number of Gaussian primitives compared to standard 3DGS, which may increase memory usage [S1].
*   **Background Floaters:** Without the depth scaling, the method might still generate artifacts in low-texture background regions; the depth factor is critical for suppression [S1].

## Subtopics

### [PixelGS Densification Criteria](9689-pixelgs-densification-criteria.md)
Modifies the gradient-based densification strategy to account for the spatial footprint of each Gaussian primitive. It scales the positional gradient by the number of pixels a Gaussian influences and applies a depth-dependent scaling factor to encourage densification in background regions while suppressing it in the foreground. This approach yields state-of-the-art rendering quality compared to standard 3DGS, though it typically results in a higher number of Gaussian primitives.

## Operating characteristics
*   **Time Complexity:** Increases with the number of viewpoints due to the accumulation of pixel coverage counts across all views [S1].
*   **Memory Complexity:** Higher than standard 3DGS due to the increased number of Gaussian primitives required to achieve state-of-the-art quality [S1].
*   **Performance:** Yields state-of-the-art rendering quality compared to standard 3DGS [S1].
*   **Strengths:** Effective at refining regions with high pixel-level error; reduces floaters through depth scaling.
*   **Limitations:** Results in a higher number of Gaussian primitives, potentially impacting memory and rendering speed [S1].

## Background
Pixel-GS was introduced as part of the evolution of 3D Gaussian Splatting densification strategies. It is cited in recent surveys of 3D Gaussian Splatting advancements [S1]. The method addresses the limitation of standard gradient-based densification by considering the spatial extent of the error in pixel space [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
