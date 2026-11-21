# Sparse-view 3D Gaussian Splatting with Depth Priors

Also called: SparseGS, CoherentGS, DNGaussian

## Overview
This method addresses the challenge of high-fidelity 3D reconstruction and novel view synthesis from sparse input views, where traditional 3D Gaussian Splatting (3DGS) suffers from geometric ambiguity and overfitting. It sets itself apart from standard 3DGS by integrating a pre-trained 2D depth estimation network to provide metric depth priors, which supervise the rendered depth and guide the optimization of Gaussian geometry [S1]. Unlike sibling methods that may rely on implicit SDFs or dense inputs, this approach explicitly aligns discrete Gaussians with estimated continuous depth surfaces and utilizes Score Distillation Sampling (SDS) to enforce photometric faithfulness in novel views [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sparse set of posed input images $\{I_i\}_{i=1}^N$ with known camera poses. A pre-trained 2D depth estimation network $d(\cdot)$ is required.
*   **Outputs:** A set of 3D Gaussians $\mathcal{G}$ (centers $\mu$, covariances $\Sigma$, opacities $\alpha$, spherical harmonics coefficients $\mathbf{c}$) that can be rendered to novel viewpoints.

**Data Structures:**
The core data structure is the standard 3D Gaussian representation:
*   $\mu \in \mathbb{R}^3$: Center position.
*   $\Sigma \in \mathbb{R}^{3 \times 3}$: Covariance matrix (often parameterized as rotation $\mathbf{R} \in SO(3)$ and scale $\mathbf{s} \in \mathbb{R}^3$).
*   $\alpha \in [0, 1]$: Opacity.
*   $\mathbf{c}$: Spherical Harmonics coefficients for view-dependent color.

**Numbered Steps:**
1.  **Initialization:** Initialize Gaussians using Structure-from-Motion (SfM) points. [S1]
2.  **Depth Estimation:** For each input view $I_i$, estimate a dense depth map $D_i^{est}$ using the pre-trained 2D network $d(I_i)$. [S1]
3.  **Rendering:** Render the current set of Gaussians $\mathcal{G}$ to obtain:
    *   Color image $C_{render}$.
    *   Depth image $D_{render}$ (accumulated depth weighted by opacity).
4.  **Depth Consistency Loss:** Compute a loss term $\mathcal{L}_{depth}$ between the rendered depth $D_{render}$ and the estimated depth $D_i^{est}$. This aligns the closest Gaussian to the pixel with the estimated surface. [S1]
5.  **Gaussian Pruning:** Remove Gaussians that exhibit incorrect depth values relative to the estimated prior. [S1]
6.  **SDS Regularization:** To further constrain geometry and appearance in unobserved regions, apply Score Distillation Sampling (SDS) loss [97] to encourage rendered results from novel viewpoints to be faithful to the prior distribution of the diffusion model used in the SDS process. [S1]
7.  **Optimization:** Minimize the combined loss $\mathcal{L} = \mathcal{L}_{photometric} + \lambda_1 \mathcal{L}_{depth} + \lambda_2 \mathcal{L}_{SDS}$ via gradient descent. [S1]

**Formulas:**
*   **Depth Loss:** $\mathcal{L}_{depth} = \| D_{render} - D_i^{est} \|_1$ (or similar L1/L2 norm). [S1]
*   **Total Loss:** $\mathcal{L} = \mathcal{L}_{photometric} + \lambda_{depth} \mathcal{L}_{depth} + \lambda_{SDS} \mathcal{L}_{SDS}$. [S1]

**Parameters:**
*   $\lambda_{depth}$: Weight for depth consistency. Not given in the sources.
*   $\lambda_{SDS}$: Weight for SDS regularization. Not given in the sources.
*   $N$: Number of sparse input views. [S1]

**Edge Cases and Pitfalls:**
*   **Depth Ambiguity:** Pre-trained 2D depth networks suffer from scaling and shifting ambiguities, which can lead to inconsistent estimations across views if not handled [S2] (though S1 focuses on the alignment, S2 highlights this general issue with depth priors).
*   **Incorrect Gaussians:** Gaussians initialized from SfM may be noisy or incorrect; the pruning step is critical to remove these before they corrupt the optimization [S1].
*   **Sparse View Ambiguity:** Without sufficient overlap, depth priors are the primary source of geometric constraint; failure of the depth estimator leads to poor reconstruction [S1].

## Subtopics

### FWD
**Definition:** FWD is a generalizable novel view synthesis method for sparse inputs that uses explicit depth and forward warping to achieve real-time speed and high quality [S2]. It replaces implicit MLP volumetric sampling with explicit point clouds, avoiding millions of MLP queries and enabling 130-1000× speedup compared to NeRF-based methods [S2].

**Steps:**
1.  Given sparse input images $\{I_i\}_{i=1}^N$ and poses $\{R_i, T_i\}$, estimate depth $D_i$ and features $F_i^0$ for each pixel using a depth network $d$ and spatial feature encoder $f$ [S2].
2.  Project features into 3D space using known camera intrinsics to form point cloud $P_i$ [S2].
3.  Render $P_i$ to the target view using a differentiable point cloud renderer $\pi$ [S2].
4.  Fuse rendered results from multiple views using a Transformer-based module $\mathcal{T}$ [S2].
5.  Apply a refinement module $\mathcal{R}$ to generate the final output [S2].
6.  Train end-to-end with photometric and perceptual losses [S2].

**Parameters:**
*   $N$: Number of input views. [S2]
*   $\{R_i, T_i\}$: Camera poses for input views. [S2]
*   $\{R_t, T_t\}$: Camera pose for target view. [S2]
*   $\{D_i^{sen}\}$: Optional sensor depths for input views (generally incomplete and noisy) [S2].

**Comparison:**
*   Unlike classical IBR, FWD uses sparse inputs and learns depth jointly optimized for synthesis, rather than requiring dense inputs and proxy geometry [S2].
*   It achieves real-time speeds (130-1000× faster than NeRF variants) while maintaining competitive perceptual quality [S2].

**Usage:**
*   Evaluated on ShapeNet and DTU datasets [S2].
*   Code available at https://github.com/Caoang327/fwd_code [S2].

## Operating characteristics
*   **Time Complexity:** Training time is significantly reduced compared to standard 3DGS due to the regularization provided by depth priors, which converges faster from sparse views. Inference is real-time, consistent with standard 3DGS rasterization [S1, S2].
*   **Memory Complexity:** Depends on the number of Gaussians. Pruning incorrect Gaussians reduces memory usage compared to unpruned sparse-view 3DGS [S1].
*   **Performance:** Outperforms previous NeRF-based sparse-view reconstruction methods [S1]. FWD variant achieves 130-1000× speedup over NeRF-based methods with better perceptual quality [S2].
*   **Strengths:** Enables high-quality reconstruction from very sparse views (e.g., 4-8 images) by leveraging strong 2D priors. Real-time rendering capability.
*   **Limitations:** Relies on the accuracy of the pre-trained 2D depth network. Performance degrades if depth estimates are highly inaccurate or inconsistent across views [S1, S2].
*   **When to use:** When input views are sparse and traditional 3DGS fails to reconstruct geometry. When real-time rendering is required.
*   **When not to use:** When dense input views are available (standard 3DGS may suffice). When high-precision metric depth is not required and relative geometry is acceptable.

## Background
The method originates from the need to overcome the limitations of 3D Gaussian Splatting in sparse-view scenarios, where the lack of multi-view constraints leads to poor geometry. SparseGS, CoherentGS, and DNGaussian were introduced to address this by incorporating depth priors from pre-trained 2D networks [S1]. The use of SDS loss further stabilizes the optimization in unobserved regions [S1]. FWD, while distinct in its use of explicit point clouds and forward warping, shares the goal of real-time sparse-view synthesis using depth [S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
