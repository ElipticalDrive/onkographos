# 3D Gaussian Splatting

Also called: 3DGS, Tetrahedron Splatting, dGS Conditional Slicing, direct Gaussian Splatting conditional slicing

## Overview
3D Gaussian Splatting (3DGS) is a radiance-field optimization method that represents a 3D scene as a collection of anisotropic 3D Gaussian primitives, each defined by a mean position, covariance matrix, opacity, and spherical harmonic coefficients for view-dependent color [S1]. It solves the problem of high-fidelity novel-view synthesis by replacing implicit neural networks with explicit, differentiable volumetric primitives that can be rasterized directly into 2D images, achieving real-time rendering speeds [S1]. It sets itself apart from its siblings, such as [Neural Radiance Fields](4972-neural-radiance-fields.md) and [Direct Voxel Grid Optimization](4694-direct-voxel-grid-optimization.md), by using explicit Gaussian kernels that allow for analytic integration during volume rendering and efficient gradient propagation, avoiding the need for dense ray sampling through neural networks [G].

## How it works
**Inputs:** A set of calibrated images with known camera poses.
**Outputs:** A `GaussianSet` containing optimized parameters for $N$ Gaussian primitives.

**Data Structures:**
The scene is represented by a `GaussianSet` [S1], which stores the following learnable parameters for each Gaussian $i$:
1.  **Mean ($\boldsymbol{\mu}_i$):** A 3D vector $(x, y, z)$ representing the center of the Gaussian.
2.  **Rotation ($q_i$):** A unit quaternion $(w, x, y, z)$ representing orientation.
3.  **Scale ($s_i$):** A 3D vector $(s_x, s_y, s_z)$ representing the extent along the principal axes.
4.  **Opacity ($\alpha_i$):** A scalar in $[0, 1]$ representing the density/alpha value.
5.  **Spherical Harmonics (SH) Coefficients ($\mathbf{c}_i$):** A vector of coefficients representing view-dependent color. Typically, SH of degree 3 (9 coefficients per RGB channel) is used [S1].

**Covariance Matrix:**
The 3D covariance matrix $\Sigma_i$ is derived from the scale and rotation parameters to ensure positive definiteness [G]:
$$
\Sigma_i = R_i S_i S_i^T R_i^T
$$
where $S_i = \text{diag}(s_{i,x}, s_{i,y}, s_{i,z})$ and $R_i$ is the rotation matrix constructed from the quaternion $q_i$.

**Rendering Pipeline:**
The rendering process projects the 3D Gaussians onto the 2D image plane and composites them using alpha blending.

1.  **3D to 2D Projection:**
    For each Gaussian $i$, compute the 2D covariance matrix $\Sigma'_i$ in screen space using the Jacobian of the projection transformation $J$ and the view transformation $W$ [G]:
    $$
    \Sigma'_i = J W \Sigma_i W^T J^T
    $$
    The 2D mean $\boldsymbol{\mu}'_i$ is computed by projecting the 3D mean $\boldsymbol{\mu}_i$.

2.  **Sorting:**
    Gaussians are sorted by depth along the camera ray to ensure correct front-to-back alpha blending.

3.  **Rasterization (Alpha Blending):**
    For each pixel, the color $\hat{C}(\mathbf{r})$ is computed by summing the contributions of overlapping Gaussians [S1]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i \in \mathcal{O}} c_i(\mathbf{v}) \alpha_i T_i
    $$
    where $\mathcal{O}$ is the set of overlapping Gaussians, $c_i(\mathbf{v})$ is the color evaluated from the SH coefficients for view direction $\mathbf{v}$, $\alpha_i$ is the opacity, and $T_i$ is the transmittance:
    $$
    T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    The opacity $\alpha_i$ at a pixel location $\mathbf{u}$ is determined by the 2D Gaussian density:
    $$
    \alpha_i(\mathbf{u}) = \alpha_i \exp\left(-\frac{1}{2} (\mathbf{u} - \boldsymbol{\mu}'_i)^T (\Sigma'_i)^{-1} (\mathbf{u} - \boldsymbol{\mu}'_i)\right)
    $$

**Optimization:**
The parameters are optimized using stochastic gradient descent (e.g., Adam) to minimize the difference between rendered images and ground truth images [S1]. The loss function typically includes an $L_1$ loss and a D-SSIM loss [S1].

**Edge Cases and Pitfalls:**
*   **Degenerate Gaussians:** If scale parameters become too small or negative, the covariance matrix may become singular. Regularization or clamping is required [G].
*   **Sorting Order:** Incorrect sorting leads to visual artifacts. Depth sorting must be consistent with the camera projection [G].
*   **Precision:** Floating-point precision issues can arise when computing the inverse of the 2D covariance matrix for small Gaussians [G].

## Subtopics

### [3DGS Initialization Policy](1398-3dgs-initialization-policy.md)
The 3DGS Initialization Policy defines the deterministic strategy for populating the initial set of 3D Gaussians before optimization begins. It solves the problem of establishing a valid, non-degenerate starting configuration for the radiance field by distributing primitives uniformly in space with uniform opacity. It sets apart from adaptive initialization methods by using a fixed, pre-computed opacity value derived from the total Gaussian count $N$, rather than learning initial opacities from data or using random initialization with learned values.

### [CoR-GS](1486-cor-gs.md)
CoR-GS is a method that improves sparse-view reconstruction in 3D Gaussian Splatting by defining two independent reconstructions and comparing them to discard inaccurate Gaussians.

### [GaussianSet Data Structure](2074-gaussianset-data-structure.md)
The `GaussianSet` is a container class representing a collection of 3D Gaussian primitives, storing their parameters (means, orientations, scales, opacities, and spherical harmonics coefficients) and providing methods to compute their world-space covariance matrices. It solves the problem of organizing learnable parameters into a structured format suitable for differentiable volume rendering pipelines.

### [Optical Flow Loss for 3D Gaussian Reconstruction](3303-optical-flow-loss-for-3d-gaussian-reconstruction.md)
This method improves the 3D shape reconstruction of static objects from video sequences by incorporating an optical flow loss term into the optimization objective. It leverages the temporal prior inherent in video data to resolve concavity/convexity ambiguities that are present when relying solely on silhouette and color loss terms. Unlike standard photometric approaches, this technique uses optical flow as a local estimator of surface correspondence, which is robust to long-term lighting changes and autoexposure variations common in user-captured videos.

### EffGS
EffGS is an acceleration framework for 3D Gaussian Splatting (3DGS) that improves training and rendering efficiency while maintaining high-fidelity reconstruction in both bounded and large-scale scenes [S1]. It sets itself apart from standard 3DGS and other acceleration methods by combining frequency-aware guidance, localized density control, and adaptive primitive scale modulation to prevent quality degradation in complex environments [S1].

### [MaskGaussian](4242-maskgaussian.md)
MaskGaussian is a differentiable pruning method for 3D Gaussian Splatting that models Gaussian existence probabilistically rather than deterministically, mitigating sub-optimized reconstruction caused by permanent removal of components. It extends Compact3DGS by introducing a "masked-rasterization" technique that allows gradients to flow to unsampled (masked-out) Gaussians, enabling dynamic assessment of their contribution to the evolving scene. Unlike methods that multiply masks with Gaussian attributes (opacity/scale) which zero out density and prevent gradient updates, MaskGaussian applies masks during the alpha-blending stage, ensuring all Gaussians participate in rasterization and receive meaningful gradients regardless of their current mask state.

### [Substitutive LOD with K-wise Moment-Matched Merging](4801-substitutive-lod-with-k-wise-moment-matched-merging.md)
Substitutive LOD with K-wise moment-matched merging is a substitutive level-of-detail technique that reduces the number of Gaussian splats by merging groups of $K$ splats into a single representative Gaussian. It addresses the problem of excessive memory footprint and rendering latency in large-scale 3D Gaussian Splatting scenes, particularly for mobile or web-based visualization. Unlike methods that prune or compress existing Gaussians without replacement, this approach maintains visual fidelity by using moment-matching to ensure the merged Gaussian statistically approximates the merged group, guided by specific cost metrics. It sets itself apart from sibling methods like [CoR-GS](1486-cor-gs.md) or [EffGS](3365-effgs.md) by focusing on a structured, cost-aware hierarchical reduction rather than simple density-based culling or vector quantization.

### [Gaussian Conditional Probability Formulation](4937-gaussian-conditional-probability-formulation.md)
This method treats the unnormalized 4D Gaussian primitive as a special probability distribution over spatial coordinates $(u, v)$ and time $t$, enabling the factorization of the joint distribution into a temporal marginal and a conditional spatial component. This formulation allows for the efficient computation of view-specific or time-specific Gaussians by conditioning on non-spatial dimensions, which is foundational for dynamic scene representation in 4D Gaussian Splatting. It sets apart from static 3D Gaussian Splatting by explicitly modeling temporal dependencies through conditional covariance and mean adjustments.

### [Radiative and Tomographic Gaussians](4971-radiative-and-tomographic-gaussians.md)
This family adapts 3D Gaussian Splatting (3DGS) for X-ray imaging and tomographic reconstruction by replacing view-dependent appearance models with view-independent radiative or density fields [3040-radiative-gaussian-splatting.md], [4525-x-gaussian.md]. Unlike standard 3DGS which renders RGB images using spherical harmonics, these methods model isotropic X-ray attenuation or radiative intensity, enabling the reconstruction of volumetric density from 2D projections or the synthesis of novel X-ray views [3040-radiative-gaussian-splatting.md], [4525-x-gaussian.md].

### [Density Control and Densification](5337-density-control-and-densification.md)
Density Control and Densification is the mechanism in 3D Gaussian Splatting that dynamically adjusts the number, position, and scale of Gaussian primitives during optimization to improve reconstruction quality while maintaining a fixed budget [3034-adaptive-densification.md]. It addresses the limitation of standard gradient descent, which can only move, stretch, or fade existing primitives but cannot create new Gaussians in empty regions or cleanly remove useless ones [3034-adaptive-densification.md]. It differs from its siblings by relying on a specific set of thresholds for gradient magnitude and Gaussian size to classify primitives into "under-reconstructed" (clone) or "over-reconstructed" (split) categories, whereas alternatives may use loss accumulation [3298-loss-accumulation-guided-lag-densification.md] or attention mechanisms [1488-ots-satn-densification.md].

### [Domain-Specific Adaptations](5621-domain-specific-adaptations.md)
Domain-Specific Adaptations modify the standard 3D Gaussian Splatting framework to handle non-standard input modalities, such as volumetric medical imaging (CT/MR), fluorescence microscopy, or 3D ultrasound, where traditional multi-view RGB photometric supervision is unavailable or inappropriate [G]. Unlike standard 3DGS which optimizes for novel-view synthesis from calibrated camera poses, these methods optimize Gaussian parameters against target volumes, fixed-pool microscopy constraints, or 3D intersection targets, often replacing rasterization-based rendering with direct voxel-space supervision or specialized distance metrics [G].

### [Dynamic 3D Gaussian Splatting](5703-dynamic-3d-gaussian-splatting.md)
Dynamic 3D Gaussian Splatting extends static 3D Gaussian Splatting to handle time-varying scenes by maintaining a persistent set of Gaussians whose positions and orientations evolve over time. It solves the problem of temporal inconsistency and flickering inherent in per-frame independent training. Unlike deformation-field approaches that predict offsets from a canonical space, this method explicitly tracks the time-varying parameters $(\boldsymbol{\mu}_t, q_t)$ of each Gaussian using local-rigid and As-Rigid-As-Possible (ARAP) or isometry constraints.

### [Epipolar 3D Gaussian Splatting](6070-epipolar-3d-gaussian-splatting.md)
A method that incorporates an epipolar transformer to generalize features between two views, pairing this with a probabilistic prediction of Gaussians aligned along camera-to-pixel rays.

### [3D Gaussian Splatting with Transformer-based Parameterization](6072-3d-gaussian-splatting-with-transformer-based-parameterizatio.md)
A method that tokenizes two to four images and their camera parameters, concatenates them, and passes them through a transformer block whose output tokens are decoded into Gaussian parameters.

### [Primitive Geometry and Distribution](6228-primitive-geometry-and-distribution.md)
Primitive Geometry and Distribution defines the mathematical shape, parameterization, and spatial distribution of the basis functions used to represent a 3D scene in Gaussian Splatting [G]. It solves the problem of how to discretize continuous radiance fields into a set of learnable, differentiable volumetric or surface elements that can be efficiently rasterized [G]. What sets this family apart from its siblings is the specific choice of the Gaussian distribution as the primary kernel, characterized by a mean position and a covariance matrix, which allows for analytic integration during volume rendering and efficient gradient propagation [G].

### [Quantile Rendering](7396-quantile-rendering.md)
A sparse, transmittance-guided sampling strategy that selects only the most representative Gaussians along each ray for efficient rendering of high-dimensional features, avoiding the computational overhead of densely sampling all intersecting Gaussians.

### [4D Gaussian Splatting](7992-4d-gaussian-splatting.md)
4D Gaussian Splatting (4DGS) is a method for modeling dynamic scenes by extending 3D Gaussian Splatting with a coherent 4D Gaussian representation that captures spatiotemporal deformations of position, rotation, scale, and appearance. Unlike approaches that treat space and time as independent dimensions (e.g., adding a separate 1D temporal Gaussian to 3D Gaussians), 4DGS formulates a unified 4D ellipsoid, allowing it to better capture underlying scene motion and inter-frame information exchange. It builds on [3D Gaussian Splatting](1795-3d-gaussian-splatting.md) by introducing a temporal dimension to the Gaussian primitives and a corresponding 4D spherindrical harmonic basis for view-dependent color.

### [GauFRe](7994-gaufre.md)
A method that applies exponential and normalization operations to the scaling and rotation respectively after adding delta values predicted by an MLP, ensuring convenient and reasonable optimization.

### [Surface and Mesh Extraction](8498-surface-and-mesh-extraction.md)

### [Rendering Pipelines and Extensions](8587-rendering-pipelines-and-extensions.md)
This family encompasses the differentiable rendering mechanisms and optimization extensions that convert 3D Gaussian primitives into 2D images. It solves the problem of efficiently rasterizing soft, view-dependent ellipsoids while supporting inverse rendering tasks such as lighting estimation and physical parameter optimization [G]. What sets these methods apart from standard rasterization is the use of alpha blending or weighted-sum compositing over sorted or spatially hashed primitives, often augmented with volume-based occlusion or explicit geometric intersection for sharp features [G].

### SAGA
A 2D promptable segmentation method lifted to 3D [S4, unofficial].

### Negative Component Splatting
Negative Component Splatting is a technique using negative density components in a mixture model to enhance representational power and parameter efficiency, allowing subtraction of color/density to capture complex topologies (e.g., holes) with fewer components [S5]. The rendering equation uses signed weights $w \in \mathbb{R}$ (instead of $w > 0$) in the standard alpha-compositing sum:
$$
C(u) = \sum_{i=1}^N c_i o_i T_i^{2D}(u) \prod_{j=1}^{i-1} (1 - o_j T_j^{2D}(u))
$$
where $o_i$ is opacity and $T_i^{2D}$ is the 2D component function [S5]. Opacity $o$ is constrained by a tanh function during learning so that $o \in [-1, 1]$, allowing positive and negative components to dynamically change signs while being bounded [S5]. Using squared negative components introduces pairwise interactions between all $K$ components, increasing evaluation complexity to $O(n^2)$, which is significantly slower; therefore, the linear sum with signed weights is used instead [S5].

### Solid Simulation Decoupling
Solid Simulation Decoupling decouples solid simulation and rendering by using a separate set of sampled particles for simulation (via PBD) and interpolating deformations to trained Gaussian kernels during rendering [S6]. It uses Poisson disk sampling to place simulation particles within the surface mesh explicitly reconstructed using NeuS [S6]. It performs PBD on simulation particles to animate frames [S6]. It interpolates deformation gradients and displacements from simulation particles to trained Gaussian kernels using generalized moving least squares (GMLS) [S6]. It transforms solid Gaussian kernel from rest position $x_p$ to deformed position $x_{tp}$ [S6].

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N)$ per pixel, where $N$ is the number of Gaussians, but typically only a subset overlaps each pixel. Training time is significantly faster than neural radiance fields due to the explicit representation and efficient rasterization [S1].
*   **Memory Complexity:** Memory usage is $O(N \cdot D)$, where $D$ is the dimensionality of the SH coefficients. For degree 3 SH, this is substantial but manageable for moderate scene sizes [S1].
*   **Performance:** Achieves real-time rendering speeds (e.g., 166 FPS) on modern GPUs [S1].
*   **Strengths:** High visual fidelity, real-time rendering, explicit representation allows for easy editing and manipulation [S1].
*   **Limitations:** Large memory footprint for high-resolution scenes, potential for artifacts in regions with sparse input views [S1].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1]. It builds on the principles of differentiable volume rendering and explicit scene representation. The method has since been extended to dynamic scenes, large-scale environments, and various domain-specific applications [S1][S4, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S3] Direct Conditional Parameterization for N-Dimensional Splatting, https://arxiv.org/abs/2610.22778, fetched 2026-10-30 (unofficial)
[S4] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S5] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
[S6] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
