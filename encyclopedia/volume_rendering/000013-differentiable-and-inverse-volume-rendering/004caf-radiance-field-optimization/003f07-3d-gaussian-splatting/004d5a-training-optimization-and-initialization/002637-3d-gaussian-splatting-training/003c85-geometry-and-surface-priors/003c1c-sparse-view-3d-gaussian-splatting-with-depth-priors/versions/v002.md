# Sparse-view 3D Gaussian Splatting with Depth Priors

Also called: SparseGS, CoherentGS, DNGaussian

## Overview
This method addresses the challenge of 3D reconstruction from sparse-view inputs by integrating pre-trained 2D depth estimation networks as geometric priors into the 3D Gaussian Splatting framework [S1]. It distinguishes itself from sibling methods like [NeuSG](9172-neusg.md) and [GSDF](9205-gsdf.md) by relying on explicit depth maps from a fixed 2D network rather than learning an implicit Signed Distance Function (SDF) or optimizing depth jointly with the scene representation during training [S1]. The approach enhances geometric fidelity by removing Gaussians with inconsistent depth values and utilizing Score Distillation Sampling (SDS) to ensure novel view synthesis remains faithful to the prior [S1].

## How it works
The method couples the discrete 3D Gaussian representation with a continuous geometric prior derived from a pre-trained 2D depth estimation network.

**Inputs and Outputs:**
*   **Inputs:** A sparse set of posed input images $\{I_i\}_{i=1}^N$ and their corresponding camera poses $\{R_i, T_i\}$. A pre-trained 2D depth estimation network $d(\cdot)$ is required.
*   **Outputs:** Optimized 3D Gaussian parameters (position $\mathbf{c}_i$, scale, rotation, opacity, color) and rendered novel views.

**Data Structures:**
*   **3D Gaussians:** Each Gaussian $i$ is defined by position $\mathbf{c}_i \in \mathbb{R}^3$, covariance matrix $\Sigma_i$ (or rotation quaternion and scale vector), opacity $\alpha_i$, and spherical harmonic coefficients for color.
*   **Depth Prior:** The pre-trained network $d(I_i)$ outputs a dense depth map $D_i^{est}$ for each input image.

**Optimization Objective:**
The total loss $\mathcal{L}$ combines standard photometric rendering loss with geometric consistency terms derived from the depth prior [S1]:
$$ \mathcal{L} = \mathcal{L}_{rgb} + \lambda_{depth} \mathcal{L}_{depth} + \lambda_{sdf} \mathcal{L}_{sdf} + \lambda_{normal} \mathcal{L}_{normal} + \lambda_{sds} \mathcal{L}_{sds} $$
Where:
*   $\mathcal{L}_{rgb}$ is the standard photometric loss between rendered and ground-truth images.
*   $\mathcal{L}_{depth}$ aligns rendered depth $D_{render}$ with estimated depth $D_i^{est}$, typically using an L1 norm: $\| D_{render} - D_i^{est} \|_1$ [S1].
*   $\mathcal{L}_{sdf}$ and $\mathcal{L}_{normal}$ enforce geometric consistency. Although the prior is a depth map, it can be converted to a local SDF approximation or used to derive normals. The SDF consistency encourages centers $\mathbf{c}_i$ to lie on the zero-level set: $\mathcal{L}_{sdf} = \frac{1}{N} \sum_{i=1}^{N} |f_\theta(\mathbf{c}_i)|$ [S1]. The normal consistency aligns the Gaussian normal $\mathbf{n}_i$ (derived from covariance eigenvectors) with the prior normal $\mathbf{m}_i$: $\mathcal{L}_{normal} = \frac{1}{N} \sum_{i=1}^{N} (1 - \mathbf{n}_i \cdot \mathbf{m}_i)$ [S1].
*   $\mathcal{L}_{sds}$ is the Score Distillation Sampling loss, encouraging rendered results from novel viewpoints to be faithful to the prior distribution [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize 3D Gaussians using Structure-from-Motion (SfM) points or by unpooling existing Gaussians from a coarse representation [S1].
2.  **Depth Estimation:** For each input image $I_i$, compute the estimated depth map $D_i^{est} = d(I_i)$ using the pre-trained 2D network [S1].
3.  **Gaussian Pruning:** Remove Gaussians that have incorrect depth values relative to $D_i^{est}$. This step cleans the initial geometry before joint optimization [S1].
4.  **Joint Optimization:** Optimize Gaussian parameters and potentially the prior network parameters (if fine-tuned) using gradient descent. The optimization minimizes the combined loss function defined above [S1].
5.  **Regularization:** Apply SDS loss to regularize the geometry and appearance in unobserved regions, ensuring consistency across novel viewpoints [S1].

**Edge Cases and Pitfalls:**
*   **Depth Ambiguity:** Pre-trained 2D depth networks suffer from scaling and shifting ambiguities, which can lead to inconsistent estimations across views if not handled [S2].
*   **Degenerate Input:** Sparse views may result in large unobserved regions where the depth prior is weak; SDS helps mitigate this but requires careful tuning of $\lambda_{sds}$ [S1].

## Subtopics

### FWD
**Definition:** FWD is a generalizable novel view synthesis method for sparse inputs that uses explicit depth and forward warping to achieve real-time speed and high quality [S2]. Unlike the sparse-view 3D Gaussian Splatting methods which optimize per-scene Gaussians, FWD is a generalizable model that does not require per-scene optimization [S2].

**Mechanism:**
1.  Given sparse input images $\{I_i\}_{i=1}^N$ and poses $\{R_i, T_i\}$, estimate depth $D_i$ and features $F_i^0$ for each pixel [S2].
2.  Project features into 3D space using known camera intrinsics to form point cloud $P_i$ [S2].
3.  Render $P_i$ to the target view using a differentiable point cloud renderer $\pi$ [S2].
4.  Apply a Transformer-based fusion module and a refinement module $R$ for the final output [S2].
5.  Train end-to-end with photometric and perceptual losses [S2].

**Parameters:**
*   $N$: Number of input views [S2].
*   $\{R_i, T_i\}$: Camera poses for input views [S2].
*   $\{R_t, T_t\}$: Camera pose for target view [S2].
*   $\{D_i^{sen}\}$: Optional sensor depths for input views (generally incomplete and noisy) [S2].

**Comparison:** FWD replaces implicit MLP volumetric sampling with explicit point clouds, avoiding millions of MLP queries and enabling a 130-1000× speedup compared to NeRF-based methods [S2]. It uses sparse inputs and learns depth jointly optimized for synthesis, unlike classical Image-Based Rendering (IBR) which requires dense inputs [S2].

**Usage:** Evaluated on ShapeNet and DTU datasets. Code available at https://github.com/Caoang327/fwd_code [S2].

## Operating characteristics
*   **Time Complexity:** Per-scene optimization is required for SparseGS/CoherentGS/DNGaussian, making it slower than generalizable methods like FWD during inference but faster than NeRF during training [S1, S2]. FWD achieves real-time speed (130-1000× speedup over NeRF) [S2].
*   **Memory Complexity:** Depends on the number of Gaussians or points. SparseGS methods typically require less memory than dense voxel-based NeRFs [S1].
*   **Strengths:** Faithful geometry reconstruction from sparse views; real-time rendering capability (for FWD); leverages strong 2D priors [S1, S2].
*   **Limitations:** Reliance on the quality of the pre-trained 2D depth network; potential for scaling ambiguities in depth estimation; requires per-scene optimization for Gaussian-based methods [S1, S2].
*   **When to use:** When high-quality 3D reconstruction is needed from very few input views (e.g., 4-8 images) and real-time rendering is not the primary constraint (for Gaussian methods). Use FWD when real-time generalizable synthesis is required [S1, S2].

## Background
The method originates from the need to improve 3D Gaussian Splatting in scenarios with limited input data. Traditional Gaussian Splatting requires dense views for stable optimization [S1]. SparseGS, CoherentGS, and DNGaussian were introduced to address this by incorporating depth priors from pre-trained 2D networks [S1]. FWD was proposed as a generalizable alternative using explicit depth and forward warping for real-time synthesis [S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
