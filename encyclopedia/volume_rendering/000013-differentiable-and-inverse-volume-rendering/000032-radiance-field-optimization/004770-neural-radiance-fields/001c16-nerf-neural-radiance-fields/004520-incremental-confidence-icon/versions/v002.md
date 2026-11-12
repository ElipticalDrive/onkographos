# Incremental CONfidence (ICON)


## Overview
ICON is an optimization procedure for training Neural Radiance Fields (NeRFs) from streaming 2D video frames that eliminates the requirement for initial camera pose estimation via Structure-from-Motion (SfM) [S1]. It distinguishes itself from sibling methods like [NeRF (Neural Radiance Fields)](1185-nerf-neural-radiance-fields.md) and [BARF](link) by using an incremental registration approach with a "Neural Confidence Field" to dynamically reweight gradients, allowing the model to interpolate between learning the NeRF when poses are good and learning poses when the NeRF is good [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of monocular RGB video frames $\{I_i\}$, where $i=1, \dots, N$.
*   **Output:** Optimized NeRF parameters $\Theta$, camera poses $\{P_i\}$, and a Neural Confidence Field.

**Data Structures:**
1.  **NeRF MLP ($f_\Theta$):** A multilayer perceptron parameterized by $\Theta$.
    *   **Input:** 3D point $\mathbf{x} \in \mathbb{R}^3$ and viewing direction $\mathbf{d} \in \mathbb{R}^3$.
    *   **Output:** Color $\mathbf{c} \in \mathbb{R}^3$ and volume density $\sigma \in \mathbb{R}$.
2.  **Neural Confidence Field ($\zeta$):** A separate MLP (or shared head) mapping 3D location to a scalar confidence value $\zeta \in [0, 1]$.
    *   **Input:** 3D point $\mathbf{x} \in \mathbb{R}^3$.
    *   **Output:** Confidence $\zeta(\mathbf{x}) \in \mathbb{R}$.
3.  **Camera Poses ($P_i$):** 6-DoF transformation matrices for each frame $i$.

**Numbered Steps:**

1.  **Initialization:**
    *   Initialize the NeRF parameters $\Theta$ and the Confidence Field parameters.
    *   Initialize the first frame's pose $P_1$ (e.g., identity or known reference).
    *   Set the previous pose $P_{prev} = P_1$.

2.  **Incremental Frame Registration:**
    For each new incoming video frame $I_i$ (where $i > 1$):
    a.  **Pose Initialization:** Initialize the current frame's pose $P_i$ using the previous frame's pose $P_{prev}$, leveraging a motion-smoothness prior.
    b.  **Ray Sampling:** For each pixel in $I_i$, cast a ray through the camera center defined by $P_i$. Sample points along the ray.
    c.  **NeRF Rendering:** Query the NeRF MLP $f_\Theta$ at sampled points $(\mathbf{x}, \mathbf{d})$ to obtain color $\mathbf{c}$ and density $\sigma$. Render the predicted image $\hat{I}_i$ using volume rendering integration.
    d.  **Confidence Estimation:**
        i.  Compute the photometric error between the predicted image $\hat{I}_i$ and the input frame $I_i$.
        ii. Map this error to a "Pose Confidence" score for the current frame.
        iii. Query the Neural Confidence Field $\zeta$ for sampled 3D points to obtain "Ray Confidence" values.
    e.  **Gradient Reweighting:**
        i.  **Learning NeRF:** If the current pose confidence is high, update NeRF parameters $\Theta$ using the photometric loss, weighted by the Ray Confidence $\zeta$. This prioritizes 3D locations with high confidence.
        ii. **Learning Pose:** If the NeRF is considered "good" (high Ray Confidence in the scene), update the pose $P_i$ using 3D-2D reprojection errors, weighted by the Pose Confidence.
    f.  **Update:** Set $P_{prev} = P_i$ and store the optimized pose and updated NeRF/Confidence Field.

**Formulas:**
*   **NeRF Rendering:** The color $\hat{C}(r)$ for a ray $r$ is computed as:
    $$ \hat{C}(r) = \sum_{i=1}^{N} T_i \left( 1 - \exp(-\sigma_i \delta_i) \right) c_i $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the accumulated transmittance, $\sigma_i$ is the density, $c_i$ is the color, and $\delta_i$ is the distance between samples [S1].
*   **Photometric Loss:**
    $$ L_p = \| I_i - \hat{I}_i \|_2 $$
    The total loss for updating NeRF parameters $\Theta$ is minimized as:
    $$ \Theta^* = \arg \min_\Theta L_p (I_i | \hat{I}_i) $$
    [S1]
*   **Confidence Integration:** The optimization interpolates between regimes. High confidence in pose leads to NeRF updates; high confidence in 3D structure (encoded by $\zeta$) leads to pose updates [S1].

**Parameters:**
*   **Motion Smoothness Prior:** Used to initialize $P_i$ from $P_{prev}$. No specific numerical value is given, but it assumes continuous camera motion [S1].
*   **Failure Threshold:** Joint pose and NeRF optimization methods (like BARF) fail when pose noise exceeds approximately 20 degrees of rotation error [S1].

**Edge Cases and Pitfalls:**
*   **Fragmentation:** If initialized with identity poses for diverse viewpoints, joint optimization often collapses into fragmentation, where each frame creates its own mutually invisible 3D representation [S1].
*   **Bas-Relief Ambiguity:** The method must handle the classical bas-relief ambiguity, which can lead to overlapping registration or incorrect depth estimation [S1].
*   **Complex Trajectories:** Methods relying on initial pose refinement often fail on complex trajectories (e.g., full 360-degree flyarounds) if the initial noise is too high [S1].

## Operating characteristics
*   **Time Complexity:** Incremental, processing frames one at a time. Complexity depends on the number of samples per ray and the depth of the NeRF MLP.
*   **Memory Complexity:** Requires storing the NeRF MLP, the Confidence Field MLP, and the history of poses.
*   **Strengths:** Removes the need for SfM preprocessing; robust to large initial pose errors in incremental settings; handles challenging scenarios where camera poses are difficult to obtain [S1].
*   **Limitations:** Relies on the assumption of smooth camera motion; may struggle with abrupt motion or large occlusions [S1].
*   **Performance:** Achieves superior performance on CO3D and HO3D datasets compared to methods using SfM poses [S1].

## Background
ICON was introduced by Weiyao Wang and Matt Feiszli at Meta in 2024 [S1]. It addresses the chicken-and-egg problem in joint pose and 3D reconstruction, where previous methods required decent initial poses from SfM or additional signals like GANs, SLAM, or depth [S1]. It draws inspiration from incremental SfM and SLAM, adapting these concepts to the differentiable rendering setting of NeRFs [S1].

## Sources
[S1] ICON: Incremental CONfidence for Joint Pose and Radiance Field Optimization, https://doi.org/10.1109/cvpr52733.2024.00517, fetched 2026-10-26
