# Deformable 3D Gaussian Splatting

Also called: Canonical Space and Deformation Field, Deformable-GS, Deformable3DGS (Yang et al.), Dynamic 3D Gaussian Splatting (Canonical Space + Deformation Field), Implicit Deformation Modeling, SurgicalGaussian, Yang et al

## Overview
Deformable 3D Gaussian Splatting is a method for dynamic scene reconstruction that represents a scene as a set of 3D Gaussians in a static **canonical space**, with temporal and spatial deformations modeled by an implicit deformation field implemented as a Multi-Layer Perceptron (MLP) [S1, S2][S6, unofficial]. It solves the problem of reconstructing time-varying volumetric scenes by decoupling the static geometry from the motion, allowing for real-time rendering of dynamic content [S3][S5, unofficial]. It sets itself apart from siblings like [GSGEN](8985-gsgen.md) or [Fourier Splatting Primitive](11094-fourier-splatting-primitive.md) by explicitly learning a continuous deformation field to map canonical attributes to observation-space attributes at any timestep, rather than storing explicit trajectories or using frequency-based parameterizations [S2][S6, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of $T$ frames, where each frame $i$ consists of an image $I_i$, a depth map $D_i$, and a binary mask $M_i$ (indicating occlusions or objects to exclude, such as surgical tools) [S4]. Camera poses $K_i$ (intrinsic) and $E_i$ (extrinsic) are required for initialization [S4].
*   **Outputs:** A rendered image $\hat{C}_i$ and depth $\hat{D}_i$ for any timestep $t \in [0, 1]$, and the optimized canonical Gaussian parameters and deformation field weights [S4].

**Data Structures:**
1.  **Canonical Gaussians ($G_c$):** A list of $N$ Gaussians. Each Gaussian $G_c$ stores:
    *   $\mu_c \in \mathbb{R}^3$: Spatial mean (position).
    *   $q_c \in \mathbb{R}^4$: Rotation quaternion.
    *   $s_c \in \mathbb{R}^3$: Scaling vector.
    *   $c \in \mathbb{R}^k$: View-dependent color (typically SH coefficients).
    *   $\alpha \in \mathbb{R}$: Opacity.
    *   *Note:* Color and opacity are typically static in the canonical space for this method [S3, S4].
2.  **Deformation Field ($F_\Theta$):** An MLP with parameters $\Theta$.
    *   Architecture: 8 hidden layers, width 256 [S4].
    *   Input: Concatenation of positional encoding of $\mu_c$ and normalized time $t$.
    *   Output: Offsets $\delta x \in \mathbb{R}^3$, $\delta s \in \mathbb{R}^3$, $\delta q \in \mathbb{R}^4$ [S2, S4].

**Numbered Steps:**

1.  **Initialization:**
    *   Initialize canonical positions $\mu_c$ and colors $c$ using the GIDM (Geometry Initialization from Depth Maps) strategy [S4].
    *   Project refined depth/mask data into 3D space to create a point cloud $P^*$ [S4].
    *   $P^* = \{ D_e K_i^{-1} (I \odot (1 - M)) \}$, where $M = \bigcap_{i=0}^T M_i$ is the intersection of tool masks (regions occluded in all frames) [S4].
    *   Initialize rotation $q_c$ and scale $s_c$ using standard 3DGS SfM initialization or random initialization [S4][S6, unofficial].

2.  **Deformation Prediction (Forward Mapping):**
    *   For a target timestep $t \in [0, 1]$ (normalized from frame index $i$ as $t = i/T$) [S4]:
    *   Apply positional encoding $\gamma(\cdot)$ to the canonical position $\mu_c$ and time $t$ [S2, S4].
    *   Compute offsets: $(\delta x, \delta s, \delta q) = F_\Theta([\gamma(\mu_c), \gamma(t)])$ [S4].
    *   To improve temporal smoothing, add linearly decaying Gaussian noise to the encoded time vector before inputting to the MLP [S2].

3.  **Attribute Transformation to Observation Space:**
    *   Compute deformed attributes for observation space:
        *   Position: $\mu_o = \mu_c + \delta x$ [S4].
        *   Scale: $s_o = s_c \cdot \exp(\delta s)$ [S4].
        *   Rotation: $q_o = q_c \cdot \delta q$ (quaternion multiplication) [S4].
        *   Color and Opacity remain unchanged: $c_o = c, \alpha_o = \alpha$ [S3, S4].

4.  **Rendering:**
    *   Project 3D Gaussians to 2D image plane.
    *   Compute 3D covariance $\Sigma = R S S^T R^T$ from $q_o$ and $s_o$ [S4].
    *   Compute 2D covariance $\Sigma'$ using the viewing transformation $V$ and Jacobian $J$ of the affine approximation of the projective transformation:
        $$ \Sigma' = J V \Sigma V^T J^T $$
        [S3, S4].
    *   Render color $\hat{C}(r)$ and depth $\hat{D}(r)$ for pixel $r$ by blending $N$ sorted Gaussians:
        $$ \hat{C}(r) = \sum_{i \in N} \alpha_i' \left( \prod_{j=1}^{i-1} (1 - \alpha_j') \right) c_i $$
        $$ \hat{D}(r) = \sum_{i \in N} \alpha_i' \left( \prod_{j=1}^{i-1} (1 - \alpha_j') \right) d_i $$
        where $\alpha_i' = \alpha_i G_i'(r)$ is the opacity weighted by the projected 2D Gaussian probability density at pixel $r$, and $d_i$ is the depth of the $i$-th Gaussian [S4].

5.  **Optimization (Loss Functions):**
    *   Minimize the total loss $L$ using Adam optimizer [S4]:
        $$ L = (L_{color} + \lambda_1 L_{ssim} + \lambda_2 L_{depth}) + (\lambda_3 L_{pos} + \lambda_4 L_{cov} + \lambda_5 L_{smooth}) $$
        [S4].
    *   **Reconstruction Losses:**
        *   Color: $L_{color} = \frac{1}{1-M_i} \| I_i - \hat{C}_i \|_2^2$ [S4].
        *   Depth: $L_{depth} = \frac{1}{1-M_i} \| D_i - \hat{D}_i \|_2^2$ [S4].
        *   SSIM: $L_{ssim} = 1 - SSIM(I_i, \hat{C}_i)$ [S4].
        *   *Note:* Masks $M_i$ are inverted to select soft tissues (regions not occluded by tools) [S4].
    *   **Regularization Losses:**
        *   Position Consistency ($L_{pos}$): For each Gaussian $G_i$, find $K=5$ nearest neighbors. Constrain the difference in Euclidean distances between canonical and observation spaces [S4]:
            $$ L_{pos} = \frac{1}{N} \sum_{i=1}^N \frac{1}{K} \sum_{k=1}^K \| \|x_o^{(i)} - x_o^{(k)}\|_2 - \|x_c^{(i)} - x_c^{(k)}\|_2 \|_2^2 $$
            [S4].
        *   Covariance Consistency ($L_{cov}$): Similar to $L_{pos}$ but for covariance matrices $\Sigma$ [S4]:
            $$ L_{cov} = \frac{1}{N} \sum_{i=1}^N \frac{1}{K} \sum_{k=1}^K \| \|\Sigma_o^{(i)} - \Sigma_o^{(k)}\|_2 - \|\Sigma_c^{(i)} - \Sigma_c^{(k)}\|_2 \|_2^2 $$
            [S4].
        *   Color Smoothness ($L_{smooth}$): Applied to regions $M^*$ (occluded in all frames) to prevent holes [S4]:
            $$ L_{smooth} = \frac{1}{n} \sum_{p,q} (\|C_{p,q} - C_{p-1,q}\|_2^2 + \|C_{p,q} - C_{p,q-1}\|_2^2) $$
            where $C = \hat{C}_i \odot M^*$, and $n$ is the number of pixels in $C$ [S4].

**Parameters:**
*   $\lambda_1 = 0.2$ (SSIM weight) [S4].
*   $\lambda_2 = 0.001$ (Depth weight) [S4].
*   $\lambda_3 = 1$ (Position regularization weight) [S4].
*   $\lambda_4 = 200$ (Covariance regularization weight) [S4].
*   $\lambda_5 = 0.02$ (Color smoothness weight) [S4].
*   Initial learning rate for deformation network: $1.5 \times 10^{-5}$ [S4].
*   Training iterations: 40,000 [S4].

**Edge Cases and Pitfalls:**
*   **Inaccurate Poses:** Discrete 3DGS amplifies pose errors compared to continuous NeRFs, especially during time interpolation [S2].
*   **Single-View Under-constraint:** Single-view inputs lead to noisy deformations. Addressed by $L_{pos}$ and $L_{cov}$ [S4].
*   **Occluded Regions:** Gaussians in areas occluded by instruments in all frames ($M^*$) are hard to optimize, causing holes. Addressed by $L_{smooth}$ [S4].

## Subtopics

### [Shape-Shifting Splats](10708-shape-shifting-splats.md)
Predicts all Gaussian parameters via shallow MLPs conditioned on a control parameter, replacing explicit storage with learned functions for context translation and stylization.

### [GSGEN](8985-gsgen.md)
Uses voxelization and sparse convolutions for geometry-aware deformation learning.

### [Fourier Splatting Primitive](11094-fourier-splatting-primitive.md)
Uses frequency-based parameterizations for dynamic attributes.

### [Conditional Slicing Mechanism](11455-conditional-slicing-mechanism.md)
Uses conditional mechanisms for dynamic scene representation.

### [Structure-Aware 3D Gaussian Splatting (SAGS)](11908-structure-aware-3d-gaussian-splatting-sags.md)
Incorporates structural awareness into Gaussian deformation.

### [Mesh-embedded representation](12559-mesh-embedded-representation.md)
Embeds Gaussians into mesh structures for deformation.

### [Inverse Gaussian Splattling](13193-inverse-gaussian-splattling.md)
Uses inverse mapping for deformation.

### [Occupancy-Opacity Factorization](13440-occupancy-opacity-factorization.md)
Factorizes occupancy and opacity for dynamic scenes.

### [CAG-GS Semantic Mapping and Gaussian Generation](15260-cag-gs-semantic-mapping-and-gaussian-generation.md)
Integrates semantic mapping with Gaussian generation.

### [Geometry and orientation extension](15381-geometry-and-orientation-extension.md)
Extends geometry and orientation handling.

### [Voxel and Grid-Based Prediction](15382-voxel-and-grid-based-prediction.md)
Uses voxel/grid-based prediction for deformation.

### [Functional Gaussian Splatting Representation](15572-functional-gaussian-splatting-representation.md)
Represents Gaussians functionally.

### [InnerGS Volumetric Rendering](15748-innergs-volumetric-rendering.md)
Focuses on volumetric rendering techniques.

### [Hybrid and Multi-Representation](15831-hybrid-and-multi-representation.md)
Combines multiple representations.

### [Plenoptic Sampling Guided 3DGS Construction](16971-plenoptic-sampling-guided-3dgs-construction.md)
Uses plenoptic sampling for construction.

### [Geometry Guided Gaussian Points Generation](16972-geometry-guided-gaussian-points-generation.md)
Guides generation with geometry.

### [Optical and Attribute Enhancement](17107-optical-and-attribute-enhancement.md)
Enhances optical and attribute properties.

### [Sampling and Initialization Strategy](17201-sampling-and-initialization-strategy.md)
Focuses on sampling and initialization.

### [Wavelet-to-Gaussian Transition Bank](17351-wavelet-to-gaussian-transition-bank.md)
Transitions from wavelets to Gaussians.

### [EA-3DGS](20636-ea-3dgs.md)
Efficient Adaptive 3D Gaussian Splatting.

### [Hard Prediction Pipeline](21482-hard-prediction-pipeline.md)
Uses hard prediction for attributes.

### [SHARP Gaussian Generation with Hard Depth Anchor](21493-sharp-gaussian-generation-with-hard-depth-anchor.md)
Uses hard depth anchors for generation.

## Operating characteristics
*   **Time Complexity:** Rendering is real-time, comparable to static 3DGS, with overhead from the deformation MLP [S4].
*   **Memory Complexity:** The deformation field reduces memory from $O(N^3 T)$ to $O(R_s N^2 + R_s N + R_t T)$ if using low-rank decomposition, but standard MLP implementation stores weights independent of $N$ and $T$ [S3].
*   **Performance:** On RTX 3090, achieves $\ge 80$ FPS with $\le 4$ GB GPU memory [S4].
*   **Quality:** Outperforms NeRF-based methods (EndoNeRF, LerPlane, EndoSurf) in PSNR, SSIM, and LPIPS on surgical datasets [S4]. On D-NeRF, achieves PSNR 39.51, SSIM 0.990, LPIPS 0.012 [S2].
*   **Strengths:** Real-time rendering, high-fidelity reconstruction of dynamic scenes, ability to remove occlusions (e.g., surgical tools) [S4].
*   **Limitations:** Sensitive to inaccurate camera poses [S2]. Single-view inputs require strong regularization to avoid noisy deformations [S4].

## Background
Deformable 3D Gaussian Splatting was introduced by W. Xie et al. in the paper "SurgicalGaussian: Deformable 3D Gaussians for High-Fidelity Surgical Scene Reconstruction" [S4]. It builds on the original 3D Gaussian Splatting [8] and extends it to dynamic scenes by incorporating a deformation field, similar to dynamic NeRF approaches [S2][S6, unofficial]. It is primarily used for high-fidelity reconstruction of dynamic surgical scenes, robot-assisted surgery, and intelligent medical care [S4].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] VolSegGS: Segmentation and Tracking in Dynamic Volumetric Scenes via Deformable 3D Gaussians, https://doi.org/10.1109/tvcg.2025.3642516, fetched 2026-10-26
[S4] SurgicalGaussian: Deformable 3D Gaussians for High-Fidelity Surgical Scene Reconstruction, https://doi.org/10.1007/978-3-031-72089-5_58, fetched 2026-10-26
[S5] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S6] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
