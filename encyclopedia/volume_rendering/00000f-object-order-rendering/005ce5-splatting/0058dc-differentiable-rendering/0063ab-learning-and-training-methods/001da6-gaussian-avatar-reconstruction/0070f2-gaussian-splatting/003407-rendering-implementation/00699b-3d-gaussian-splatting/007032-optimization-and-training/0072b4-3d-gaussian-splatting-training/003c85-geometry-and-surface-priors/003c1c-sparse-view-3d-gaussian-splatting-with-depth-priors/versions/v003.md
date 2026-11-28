# Sparse-view 3D Gaussian Splatting with Depth Priors

Also called: SparseGS, CoherentGS, DNGaussian

## Overview
This method addresses the challenge of 3D reconstruction from sparse-view inputs by integrating depth priors estimated by pre-trained 2D networks into the 3D Gaussian Splatting framework [S1]. It distinguishes itself from sibling methods like [NeuSG](9172-neusg.md) and [GSDF](9205-gsdf.md) by explicitly leveraging monocular depth estimation to supervise rendered depth, removing Gaussians with incorrect depth values, and utilizing Score Distillation Sampling (SDS) to ensure fidelity in novel viewpoints [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse set of input images $\{I_i\}_{i=1}^N$ with known camera poses $\{R_i, T_i\}$ [S1, S2]. Optional sensor depths $\{D_i^{sen}\}$ which are generally incomplete and noisy [S2].
*   **Outputs:** A set of 3D Gaussians representing the scene, and synthesized novel view images [S1, S2].

**Data Structures:**
*   **Gaussian Set:** A collection of $N$ Gaussians, each defined by center $\mathbf{c}_i$, covariance matrix (or rotation quaternion and scale), opacity $\alpha_i$, and spherical harmonic coefficients for color [S1].
*   **Point Cloud (FWD variant):** For the FWD variant, input images are converted to 3D point clouds $P_i$ containing geometries and view-dependent semantics [S2].

**Algorithm Steps:**
1.  **Depth Estimation:** Estimate monocular depth $D_i$ for each input image $I_i$ using a pre-trained 2D depth network $d$ [S1, S2].
2.  **Feature Extraction:** Compute per-pixel feature vectors $F_i^0$ using a spatial feature encoder $f$ and view-dependent MLP $\psi$ [S2].
3.  **Point Cloud Construction (FWD variant):** Project features $F_i^0$ into 3D space using known camera intrinsics to form point cloud $P_i$ [S2].
4.  **Gaussian Initialization/Refinement:** Initialize or unpool Gaussians from Structure-from-Motion (SfM) points [S1].
5.  **Depth Consistency Loss:** Compute a consistency loss between the Gaussian attributes and the estimated depth prior. This involves rendering depth images from the current Gaussian set and comparing them to the estimated depth $D_i$ [S1].
6.  **Gaussian Pruning:** Remove Gaussians that have incorrect depth values relative to the estimated depth prior [S1].
7.  **Score Distillation Sampling (SDS):** Apply SDS loss to encourage rendered results from novel viewpoints to be more faithful to the underlying scene distribution [S1].
8.  **Rendering (FWD variant):** Render point clouds $P_i$ to target viewpoints using a differentiable point cloud renderer $\pi$ [S2].
9.  **Fusion and Refinement (FWD variant):** Fuse rendered results at the target view using a Transformer-based module $\mathcal{T}$ and apply a refinement module $\mathcal{R}$ [S2].
10. **Optimization:** Optimize the Gaussian attributes (and potentially the depth network features) end-to-end using a combination of photometric loss $\mathcal{L}_{rgb}$, depth consistency loss, and SDS loss [S1, S2].

**Formulas:**
*   **Total Loss (General SparseGS/CoherentGS/DNGaussian):**
    $$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{depth} \mathcal{L}_{depth} + \lambda_{sds} \mathcal{L}_{sds} $$
    Where $\mathcal{L}_{depth}$ enforces consistency with estimated depth, and $\mathcal{L}_{sds}$ is the Score Distillation Sampling loss [S1].
*   **Photometric Loss (FWD variant):**
    $$ \mathcal{L}_{rgb} = \| I_{rendered} - I_{target} \|_1 $$
    (Standard L1 photometric loss between rendered and target image) [S2].
*   **Perceptual Loss (FWD variant):** Added to photometric loss to improve visual quality [S2].

**Parameters:**
*   $\lambda_{depth}, \lambda_{sds}$: Weights for depth and SDS losses. Not given in the sources.
*   $N$: Number of input views [S2].
*   Feature dimension: 61-dimensions for spatial features, concatenated with RGB to form 64-dimensions [S2].

**Edge Cases and Pitfalls:**
*   **Sensor Depth Noise:** If sensor depths $\{D_i^{sen}\}$ are used, they are generally incomplete and noisy, requiring careful integration [S2].
*   **Depth Ambiguity:** Monocular depth estimation suffers from scaling/shifting ambiguity, which can lead to inconsistent estimations across views if multi-view cues are not applied [S2].
*   **Incorrect Gaussians:** Gaussians with incorrect depth values must be explicitly removed to prevent artifacts [S1].

## Subtopics

### FWD
**Definition:** A generalizable novel view synthesis method for sparse inputs that uses explicit depth and forward warping to achieve real-time speed and high quality [S2].
**Mechanism:** It replaces implicit MLP volumetric sampling with explicit point clouds, avoiding millions of MLP queries and enabling 130-1000× speedup compared to NeRF-based methods [S2]. It constructs point clouds from estimated depth and features, renders them using a differentiable renderer, and fuses results via a Transformer module [S2].
**Comparison:** Unlike classical IBR which requires dense inputs, FWD uses sparse inputs and learns depth jointly optimized for synthesis quality [S2]. It achieves competitive quality to SOTA methods at real-time speeds [S2].
**Usage:** Evaluated on ShapeNet and DTU datasets. Code available at https://github.com/Caoang327/fwd_code [S2].

## Operating characteristics
*   **Time Complexity:** The FWD variant achieves real-time rendering speeds, offering a 130-1000× speedup over NeRF-based methods [S2]. The standard SparseGS/CoherentGS/DNGaussian methods rely on differentiable rendering of Gaussians, which is generally faster than NeRF but slower than the explicit point cloud warping of FWD [S1, S2].
*   **Memory Complexity:** FWD uses explicit point clouds, which have compact memory usage and scalability to complex scenes [S2].
*   **Strengths:** High quality synthesis from sparse inputs; real-time speed (FWD); improved geometry reconstruction via depth priors (SparseGS/CoherentGS/DNGaussian) [S1, S2].
*   **Limitations:** Reliance on pre-trained 2D depth networks which may have inaccuracies; potential for scaling/shifting ambiguity in monocular depth [S1, S2].
*   **When to use:** When high-quality novel view synthesis is required from a sparse set of input images, and real-time performance is critical (FWD), or when faithful geometry reconstruction is prioritized over pure photometric fidelity (SparseGS/CoherentGS/DNGaussian) [S1, S2].

## Background
The method originates from the need to improve 3D reconstruction quality from sparse views, a common limitation of standard 3D Gaussian Splatting which often requires dense input views for stable optimization [S1]. SparseGS, CoherentGS, and DNGaussian were developed to introduce depth inputs estimated by pre-trained 2D networks to supervise the rendered depth images, thereby improving geometry [S1]. The FWD method was proposed to address the speed limitations of NeRF-based generalizable methods, offering real-time synthesis using explicit depth and forward warping [S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
