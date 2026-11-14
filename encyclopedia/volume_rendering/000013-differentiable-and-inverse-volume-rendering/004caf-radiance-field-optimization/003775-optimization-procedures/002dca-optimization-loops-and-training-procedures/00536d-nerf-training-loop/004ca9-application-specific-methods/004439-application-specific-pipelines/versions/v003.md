# Application-Specific Pipelines


## Overview
Application-Specific Pipelines adapt the general NeRF training loop to solve domain-specific inverse problems by modifying the neural field architecture, loss functions, and optimization strategy. Unlike standard NeRF, which optimizes a single MLP for photometric reconstruction, these pipelines decompose fields to isolate physical properties, enforce domain priors (geometric, semantic, or physical), and employ iterative multi-stage optimization to handle complex constraints [S1, 12477-rehearsalnerf.md, 12668-neural-sar-surface-reconstruction-via-inverse-rendering.md, 12755-360fusionnerf.md, 13429-portrait-nerf.md, 13482-mirror-nerf.md].

## Shared principles
The children of this family share the following mechanisms and mathematical foundations:

1.  **Specialized Neural Field Architectures**: Standard NeRF uses a single MLP mapping $(x, y, z, \theta, \phi) \to (\sigma, c)$. Application-specific methods often decompose this into multiple MLPs or modify the input/output space:
    *   **Decoupling**: Fields are split to isolate specific physical properties (e.g., static vs. dynamic geometry, illumination vs. radiance) [S1, 12477-rehearsalnerf.md].
    *   **Dimensionality Reduction**: Some methods map 2D coordinates to 3D properties (e.g., elevation and backscattering from $(x,y)$) to enforce surface constraints [S1, 12668-neural-sar-surface-reconstruction-via-inverse-rendering.md].
    *   **Canonical Warping**: Coordinates are transformed into a canonical space (e.g., via 3DMM) before querying the MLP to handle non-rigid or specific subject geometries [S1, 13429-portrait-nerf.md].

2.  **Domain-Specific Loss Functions**: The optimization objective $L$ is extended beyond photometric reconstruction error ($L_{photo}$) to include constraints derived from the application domain:
    *   **Geometric/Depth Supervision**: Using auxiliary depth maps or point clouds to constrain geometry [S1, 12755-360fusionnerf.md].
    *   **Semantic Consistency**: Using high-level features (e.g., CLIP) to ensure plausible content in unobserved regions [S1, 12755-360fusionnerf.md].
    *   **Physical Priors**: Enforcing symmetry, planarity, or specific illumination models (e.g., HSV decomposition) [S1, 13482-mirror-nerf.md, 12477-rehearsalnerf.md].
    *   **Statistical Models**: Using Negative Log-Likelihood (NLL) for speckle noise in SAR data instead of MSE [S1, 12668-neural-sar-surface-reconstruction-via-inverse-rendering.md].

3.  **Iterative or Multi-Stage Optimization**: Many methods do not optimize a single static field directly. Instead, they employ iterative loops where the output of one stage (e.g., super-resolved images, uncertainty maps, or rough meshes) informs the next stage of optimization [S1, 10371-cross-guided-optimization-of-radiance-fields-crop.md, 13429-portrait-nerf.md].

4.  **Volume Rendering Equation**: All methods ultimately rely on the standard volume rendering integral to synthesize images from the learned fields:
    $$ \hat{C}(r) = \sum_{i=1}^{N} T_i \alpha_i c_i = \sum_{i=1}^{N} T_i (1 - e^{-\sigma_i \delta_i}) c_i $$
    where $T_i = \prod_{j=1}^{i-1} e^{-\sigma_j \delta_j}$ is the accumulated transmittance, $\sigma_i$ is density, $c_i$ is color, and $\delta_i$ is the interval length [S1, 10371-cross-guided-optimization-of-radiance-fields-crop.md, 12755-360fusionnerf.md, 13482-mirror-nerf.md].

## Subtopics

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
Uses cross-guidance to optimize radiance fields.

### [Urban Radiance Fields](11973-urban-radiance-fields.md)
Optimizes radiance fields for urban environments.

### [RehearsalNeRF](12477-rehearsalnerf.md)
Decomposes fields to isolate static and dynamic geometry.

### [Neural SAR Surface Reconstruction via Inverse Rendering](12668-neural-sar-surface-reconstruction-via-inverse-rendering.md)
Uses dimensionality reduction and NLL loss for SAR data.

### [360FusionNeRF](12755-360fusionnerf.md)
Incorporates semantic consistency and geometric supervision.

### [Portrait-NeRF](13429-portrait-nerf.md)
Uses canonical warping for non-rigid subject geometries.

### [Mirror-NeRF](13482-mirror-nerf.md)
Enforces physical priors like symmetry and planarity.

### [DPPE (Dense Pose Estimation in a Plenoxels Environment)](9640-dppe-dense-pose-estimation-in-a-plenoxels-environment.md)
DPPE is a 6-DoF monocular RGB-only pose estimation method that recovers a camera's ground truth pose from a perturbed initial estimate by minimizing photometric error within a pre-trained Plenoxels environment. It distinguishes itself from analytical derivative-based methods (like [30]) and slow NeRF-based optimizers (like iNeRF) by leveraging the rapid rendering speed of Plenoxels to numerically approximate image gradients via central differencing, allowing full-image re-rendering at each optimization step.

### [GO-NeRF](11181-go-nerf.md)
GO-NeRF is a pipeline for generating text-prompt-controlled 3D virtual objects at user-specified locations within an existing 3D environment represented as a NeRF, preserving unchanged scene content and maintaining compatibility with various NeRF representations. It distinguishes itself from standard NeRF training by separating the optimization of a local object field from the global scene, using compositional rendering to handle occlusions and context-aware losses (saturation alignment, random background replacement) to mitigate artifacts like floaters and oversaturation common in diffusion-guided optimization.

#### Dynamically-informed iNeRF Estimator
- [S1] A state estimation method that uses an iNeRF initialized with a dynamics prior, optimized only via photometric loss, acting as a recursive Bayesian filter with infinite state covariance.
- [S1] Process loss set to 0.
- [S1] An iNeRF estimator initialized without a dynamics prior will very quickly diverge.

## Background
Originating from the need to apply NeRF beyond static scene reconstruction, these pipelines emerged as researchers sought to integrate NeRF with robotics navigation, remote sensing, and interactive content creation. Key developments include RehearsalNeRF for dynamic scenes [12477-rehearsalnerf.md], Neural SAR for inverse radar imaging [12668-neural-sar-surface-reconstruction-via-inverse-rendering.md], and GO-NeRF for text-guided 3D editing [11181-go-nerf.md]. They are used in robotics navigation [S1], urban planning [11973-urban-radiance-fields.md], and portrait animation [13429-portrait-nerf.md].

## Sources
[S1] Vision-Only Robot Navigation in a Neural Radiance World, https://doi.org/10.1109/lra.2022.3150497, fetched 2026-10-26
