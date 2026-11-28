# Weighted Disparity Alignment


## Overview
Weighted Disparity Alignment is a supervision technique for NeRF training that aligns a monocular depth estimator's output with the NeRF's rendered geometry, specifically designed to handle masked regions in inpainting scenarios [S1]. It differs from standard affine alignment by incorporating spatially varying weights and additional tilt parameters to enforce tighter geometric consistency near mask boundaries, where misalignment is most detrimental [S1].

## How it works
**Inputs:**
*   Input RGB image $I_r$ and binary mask $M_r$ for the reference view [S1].
*   Monocular depth estimator $D(\cdot)$ (frozen) [S1].
*   NeRF model parameters $\theta$ [S1].
*   Hyperparameter $N_{depth}$: number of iterations to train on unmasked pixels before applying depth loss [S1].

**Outputs:**
*   Aligned and smoothed disparity map $D_r^{smooth}$ used for supervising the masked region of the NeRF [S1].

**Data Structures:**
*   $D_e$: Uncalibrated disparity map of the source view, computed as $D_e = D(I_r)$ [S1].
*   $H, V$: $H \times W$ matrices where for pixel $p=(p_x, p_y)$, $H(p)=p_x$ and $V(p)=p_y$ [S1].
*   $\mathbf{1}_{HW}$: $H \times W$ matrix of ones [S1].
*   $w(p)$: Weight for pixel $p$, defined as the inverse of the Euclidean distance between $p$ and the mask's center-of-mass [S1].

**Steps:**
1.  **Initial Unmasked Training:** Train the NeRF on unmasked pixels ($p \in I_r(1-M_r)$) using standard color reconstruction loss for $N_{depth}$ iterations [S1]. This allows the geometry of the visible scene to converge before introducing depth constraints on the masked area [S1].
2.  **Compute Uncalibrated Disparity:** Apply the frozen monocular depth estimator to the reference image: $D_e = D(I_r)$ [S1].
3.  **Weighted Alignment Optimization:** Solve for alignment parameters $\{a_0, a_1, a_2, a_3\}$ by minimizing the weighted squared error between the aligned predicted inverse depth and the NeRF's rendered disparity $\bar{D}_r$ on unmasked pixels [S1]:
    $$ \min_{a_0, a_1, a_2, a_3} \sum_{p \in I_r(1-M_r)} w(p) \left( D_r(p) - \bar{D}_r(p) \right)^2 $$
    where the aligned predicted inverse depth is defined as:
    $$ D_r = a_0 D_e + a_1 \mathbf{1}_{HW} + a_2 H + a_3 V $$
    The terms $a_2 H$ and $a_3 V$ introduce axis-aligned "tilts" to improve fitting quality near edges [S1].
4.  **Smoothing:** Perform an additional optimization step to correct $D_r$ and encourage smoothness around the mask boundary, yielding $D_r^{smooth}$ [S1]. (Specific smoothing details are in the supplementary material of [S1]).
5.  **Depth Loss Application:** Supervise the masked region of the reference view using the depth consistency loss:
    $$ L_{depth} = \mathbb{E}_{r' \in R_{masked}} \left( D_r^{smooth}(r') - \bar{D}_r(r') \right)^2 $$
    where $R_{masked}$ is the set of rays corresponding to masked pixels [S1].

**Edge Cases and Pitfalls:**
*   **Mask Boundary Sensitivity:** Misalignments persist near mask edges, requiring the smoothing step [S1].
*   **Scale Ambiguity:** Monocular depth is non-metric and lacks absolute scale, requiring alignment [S1].

## Subtopics

### Disparity Check
A baseline method for pose verification that computes optical flow between estimated and ground truth images to determine mean disparity [S2]. It uses a folded normal distribution with a standard deviation of 4 pixels to compute a confidence level of correctness based on this disparity [S2]. It does not handle scale ambiguity and does not generalize well across varying scene sizes [S2].

## Operating characteristics
*   **Complexity:** The alignment step involves solving a linear least-squares problem for 4 parameters over unmasked pixels, which is computationally inexpensive compared to the NeRF forward pass [S1].
*   **Performance:** Provides tighter alignment around mask edges compared to standard affine alignment, improving inpainting quality [S1].
*   **Limitations:** Relies on the accuracy of the monocular depth estimator; if the MDE fails, the alignment may propagate errors [S1]. The smoothing step details are not fully specified in the main text [S1].

## Background
Weighted Disparity Alignment was introduced in "Reference-guided Controllable Inpainting of Neural Radiance Fields" (ICCV 2023) to address the challenge of supervising geometry in masked regions where ground truth color is unavailable [S1]. It extends standard disparity alignment by adding spatial weights and tilt parameters to handle the specific artifacts introduced by inpainting [S1].

## Sources
[S1] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
[S2] VERF: Runtime Monitoring of Pose Estimation With Neural Radiance Fields, https://doi.org/10.1109/lra.2023.3341765, fetched 2026-10-30
