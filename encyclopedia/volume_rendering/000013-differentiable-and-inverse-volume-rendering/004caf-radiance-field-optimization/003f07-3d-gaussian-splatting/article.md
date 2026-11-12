# 3D Gaussian Splatting

Also called: 3DGS, Tetrahedron Splatting, dGS Conditional Slicing, direct Gaussian Splatting conditional slicing

## Overview
3D Gaussian Splatting (3DGS) is a radiance-field optimization method that represents a scene as a collection of anisotropic 3D Gaussian primitives, each defined by a mean position, covariance matrix, opacity, and spherical harmonic coefficients for view-dependent color [S1]. It solves the trade-off between the high memory and slow rendering speeds of neural implicit representations (like NeRF) and the limited expressiveness of explicit mesh or voxel grids by enabling real-time differentiable rasterization of these Gaussians [S1]. Unlike its siblings in the Radiance-field optimization family, 3DGS avoids volumetric ray marching entirely, instead projecting 3D Gaussians to 2D screen space and rendering them via alpha compositing, which allows for minute-level training times and real-time novel view synthesis [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of calibrated images with known camera poses.
*   **Outputs:** A set of learnable 3D Gaussian primitives $\mathcal{G} = \{g_1, \dots, g_N\}$, where each primitive $g_i$ is defined by parameters optimized via gradient descent to minimize the difference between rendered and ground-truth images.

**Data Structures:**
Each Gaussian primitive $g_i$ stores the following fields [S1]:
1.  **Mean ($\mu_i \in \mathbb{R}^3$):** The 3D position of the Gaussian center.
2.  **Rotation ($R_i \in SO(3)$):** A quaternion or rotation matrix representing orientation.
3.  **Scale ($s_i \in \mathbb{R}^3$):** A vector of positive values representing the extent along the principal axes.
4.  **Opacity ($\alpha_i \in \mathbb{R}$):** A scalar value in $[0, 1]$ representing the density/alpha of the primitive.
5.  **Spherical Harmonics Coefficients ($c_{i,0}, \dots, c_{i,D}$):** A set of coefficients representing view-dependent color, where $D$ is the degree of the harmonics (typically $D=3$ for 9 coefficients per RGB channel).

The covariance matrix $\Sigma_i$ in 3D space is constructed from the scale and rotation parameters:
$$ \Sigma_i = R_i S_i S_i^T R_i^T $$
where $S_i = \text{diag}(s_i)$ [S1].

**Rendering Steps:**
The rendering process projects each 3D Gaussian to 2D and composites them [S1]:

1.  **3D to 2D Projection:** For each Gaussian $g_i$, compute its 2D mean $\mu'_i$ and covariance $\Sigma'_i$ in screen space using a first-order Taylor approximation of the perspective projection matrix $W$ and the camera extrinsic/intrinsic matrices [S1]:
    $$ \Sigma'_i = J W \Sigma_i W^T J^T $$
    where $J$ is the Jacobian of the affine approximation of the projective transformation. The 2D Gaussian is then defined as:
    $$ G_i(x, y) = \exp\left(-\frac{1}{2} \begin{bmatrix} x - \mu'_i \\ y - \mu'_i \end{bmatrix}^T (\Sigma'_i)^{-1} \begin{bmatrix} x - \mu'_i \\ y - \mu'_i \end{bmatrix}\right) $$

2.  **Sorting:** Sort all Gaussians that overlap with the current pixel or tile by their depth (distance from the camera) in front-to-back order [S1].

3.  **Alpha Compositing:** Render the image by accumulating color and opacity along each ray using standard alpha blending [S1]:
    $$ C = \sum_{i=1}^N c_i(\mathbf{v}_i) \alpha_i T_i $$
    $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j G_j(u, v)) $$
    where $c_i(\mathbf{v}_i)$ is the color of the $i$-th Gaussian evaluated at the viewing direction $\mathbf{v}_i$ using the spherical harmonics, $\alpha_i$ is the opacity, and $G_j(u, v)$ is the 2D Gaussian value at pixel coordinates $(u, v)$. $T_i$ is the accumulated transmittance.

4.  **Gradient Computation:** Compute the loss (e.g., L1 + D-SSIM) between the rendered image and the ground truth. Backpropagate gradients through the differentiable rendering operator to update $\mu_i, R_i, s_i, \alpha_i,$ and spherical harmonic coefficients [S1].

**Parameters:**
*   **Initial Opacity:** Typically initialized using a power-law function of the total Gaussian count to ensure stable training [S1].
*   **Learning Rate:** Adaptive learning rates are used for different parameters (e.g., higher for positions, lower for scales) [S1].

**Edge Cases and Pitfalls:**
*   **Densification Artifacts:** Without proper densification control, Gaussians may over-reconstruct or under-reconstruct regions, leading to noise or holes [S1].
*   **Precision:** The projection step involves matrix inversions and exponentials; numerical stability must be maintained, especially for very small or very large covariances [S1].
*   **Degenerate Input:** If camera poses are inaccurate, the optimization may fail to converge to a consistent 3D structure [S1].

## Subtopics

### [3DGS Initialization Policy](1398-3dgs-initialization-policy.md)
The 3DGS Initialization Policy defines the standard procedure for seeding 3D Gaussian primitives in 3D Gaussian Splatting (3DGS) and Order-Independent Splatting (OTS). It establishes a uniform initial scene representation by distributing Gaussians with random positions and a single computed opacity value derived from a power-law function of the total Gaussian count, differing from sibling methods like [CoR-GS](1486-cor-gs.md) or [Gaussian Splatting Volume Rendering](2063-gaussian-splatting-volume-rendering.md) that use geometric or learned initialization strategies.

### [CoR-GS](1486-cor-gs.md)
CoR-GS is a method designed to improve sparse-view reconstruction in 3D Gaussian Splatting by defining two independent reconstructions and comparing them to discard inaccurate Gaussians. It addresses the ambiguity and instability inherent in reconstructing scenes from minimal view overlap, where standard optimization often yields noisy or inconsistent geometry.

### [GaussianSet Data Structure](2074-gaussianset-data-structure.md)
The `GaussianSet` is a container class representing a collection of 3D Gaussian primitives, storing their geometric parameters (means, orientations, scales), opacity, and appearance (spherical harmonics coefficients), while providing methods to compute the resulting 3D covariance matrices. It solves the problem of organizing learnable parameters for differentiable volume rendering by maintaining a unified state that supports batched processing and efficient memory access. It differs from its sibling [GaussianList Data Structure and Initialization](2214-gaussianlist-data-structure-and-initialization.md) by defining the logical container and covariance computation logic, whereas the `GaussianList` explicitly defines the contiguous memory layout and initialization sequence required to compute covariances from raw parameters.

### [Optical Flow Loss for 3D Gaussian Reconstruction](3303-optical-flow-loss-for-3d-gaussian-reconstruction.md)
This method improves the 3D shape reconstruction of static objects from video sequences by incorporating an optical flow loss term into the optimization objective. It leverages the temporal prior inherent in video data to resolve concavity/convexity ambiguities that are present when relying solely on silhouette and color loss terms. Unlike standard photometric approaches, this technique uses optical flow as a local estimator of surface correspondence, which is robust to long-term lighting changes and autoexposure variations common in user-captured videos.

### EffGS
EffGS is an acceleration framework for 3D Gaussian Splatting that improves training and rendering efficiency while maintaining high-fidelity reconstruction in both bounded and large-scale scenes [S1]. It sets itself apart from standard 3DGS and other acceleration methods by combining frequency-aware guidance, localized density control, and adaptive primitive scale modulation to prevent quality degradation in complex environments [S1].

### [MaskGaussian](4242-maskgaussian.md)
MaskGaussian is a differentiable pruning method for 3D Gaussian Splatting that models Gaussians as probabilistic entities rather than deterministically removing them based on static importance scores. It solves the problem of sub-optimal reconstruction caused by early pruning by introducing **masked-rasterization**, which allows gradients to flow to "masked-out" (inactive) Gaussians, enabling them to be re-evaluated and potentially reactivated as the scene evolves. Unlike sibling methods like [Compact3DGS](1398-compact3dgs.md) which use deterministic pruning or multiply masks with opacity/scale (blocking gradients), MaskGaussian applies masks during the alpha-blending stage, decoupling Gaussian existence from its geometric attributes.

### [Substitutive LOD with K-wise Moment-Matched Merging](4801-substitutive-lod-with-k-wise-moment-matched-merging.md)
Substitutive LOD with K-wise moment-matched merging is a technique for reducing the number of 3D Gaussians in a scene by merging groups of $K$ splats into a single representative Gaussian, thereby boosting rendering performance and managing memory constraints. It addresses the problem of excessive Gaussian counts and associated aliasing artifacts by replacing small or redundant Gaussians with larger, merged ones. Unlike methods that modify Gaussians during reconstruction or prune based on static importance, this method operates on the final set of Gaussians using cost metrics to guide the merging process, allowing for runtime LOD selection. It sets itself apart from siblings like [CoR-GS](1486-cor-gs.md) and [EffGS](3365-effgs.md) by focusing on substitutive merging via moment matching rather than density control or vector quantization.

### [Gaussian Conditional Probability Formulation](4937-gaussian-conditional-probability-formulation.md)
This method treats the unnormalized 4D Gaussian primitive as a special probability distribution over spatial coordinates $(u, v)$ and time $t$, enabling the factorization of the joint distribution into a temporal marginal and a conditional spatial component. This factorization allows for efficient rendering and optimization by decoupling temporal dynamics from spatial geometry, distinguishing it from static 3D Gaussian Splatting which lacks this probabilistic temporal modeling.

### [Radiative and Tomographic Gaussians](4971-radiative-and-tomographic-gaussians.md)
Radiative and Tomographic Gaussians adapt 3D Gaussian Splatting (3DGS) for X-ray imaging by replacing view-dependent color and opacity with view-independent scalar density or radiation intensity [3040-radiative-gaussian-splatting.md]. This modification solves the problem of reconstructing volumetric data from 2D attenuation measurements, where radiation depends on the integrated density along the ray path rather than surface appearance [3040-radiative-gaussian-splatting.md]. Unlike standard 3DGS, which uses spherical harmonics for view-dependent appearance, these methods remove angular dependence to model isotropic density fields [3040-radiative-gaussian-splatting.md].

### [Density Control and Densification](5337-density-control-and-densification.md)
Density Control and Densification is the mechanism in 3D Gaussian Splatting that dynamically adjusts the number, position, and scale of Gaussian primitives during optimization to improve reconstruction quality while maintaining a fixed budget. It addresses the limitation of standard gradient descent, which can only move, stretch, or fade existing Gaussians but cannot create new ones in empty regions or cleanly remove useless ones. By inspecting view-space positional gradients at fixed intervals, it triggers cloning for under-reconstructed regions, splitting for over-reconstructed regions, and pruning for low-opacity primitives. It differs from its siblings by relying on a simple threshold-based heuristic on accumulated positional gradients and spatial extent, whereas alternatives like [OTS+SAtn Densification](1488-ots-satn-densification.md) use attention mechanisms, and [Loss Accumulation Guided (LAG) Densification](3298-loss-accumulation-guided-lag-densification.md) uses loss accumulation.

### [Domain-Specific Adaptations](5621-domain-specific-adaptations.md)
Domain-Specific Adaptations modify the standard 3D Gaussian Splatting (3DGS) framework to handle non-RGB, non-photometric data such as medical volumes, microscopy stacks, and ultrasound probe intersections. While standard 3DGS optimizes for novel-view synthesis of RGB scenes using calibrated camera poses and photometric losses, these adaptations replace the projection target with volumetric supervision, voxel-space rasterization, or 3D geometric intersections [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md], [5170-ultragauss-rendering.md]. They solve the problem of reconstructing and visualizing high-dimensional or physically distinct data fields where traditional camera-based rendering is inapplicable or insufficient.

### [Dynamic 3D Gaussian Splatting](5703-dynamic-3d-gaussian-splatting.md)
Dynamic 3D Gaussian Splatting enables real-time novel view synthesis of dynamic scenes by maintaining a persistent set of Gaussian primitives whose parameters evolve over time. Unlike per-frame baselines that suffer from storage explosion and temporal flicker, this method tracks the position and rotation of each Gaussian across frames using local-rigid and As-Rigid-As-Possible (ARAP) or isometry constraints. It distinguishes itself from deformation-field approaches (e.g., [4D Gaussian Splatting](7992-4d-gaussian-splatting.md)) by avoiding global deformation networks, instead relying on explicit temporal constraints on the primitive parameters themselves.

### [Epipolar 3D Gaussian Splatting](6070-epipolar-3d-gaussian-splatting.md)
Epipolar 3D Gaussian Splatting is a method for reconstructing 3D Gaussian Splatting (3DGS) scenes from image pairs by incorporating an epipolar transformer to generalize features between two views. It pairs this feature generalization with a probabilistic prediction of Gaussians aligned along camera-to-pixel rays. Unlike monocular approaches that estimate depth from a single view, this method leverages stereo geometry to resolve ambiguity, serving as the foundation for multi-view extensions like MVSplat and GS-LRM.

### [3D Gaussian Splatting with Transformer-based Parameterization](6072-3d-gaussian-splatting-with-transformer-based-parameterizatio.md)
This method enables feed-forward reconstruction of 3D Gaussian primitives from a small set of input images by leveraging a Transformer architecture to predict Gaussian parameters directly. It distinguishes itself from iterative optimization-based 3D Gaussian Splatting by eliminating the need for per-scene training, instead learning a direct mapping from image space to Gaussian space. Unlike pixel-aligned feed-forward methods such as MVSplat, which rely on cost volumes and U-Net decoders, this approach tokenizes images and camera parameters to predict Gaussians aligned along camera-to-pixel rays, offering a more direct parameterization.

### [Primitive Geometry and Distribution](6228-primitive-geometry-and-distribution.md)
Primitive Geometry and Distribution defines the mathematical representation of individual splats (primitives) in 3D Gaussian Splatting, specifying how position, orientation, scale, and density are parameterized and projected. It solves the problem of representing complex 3D scenes with a set of differentiable, analytically tractable basis functions that can be efficiently rasterized. It sets itself apart from its siblings by establishing the baseline 3D Gaussian ellipsoid model, whereas alternatives modify the distribution shape (e.g., Student's t, Generalized Exponential), dimensionality (2D disks), or conditioning (Direct Gaussian Splatting) to address specific limitations like geometry accuracy or computational cost.

### [Quantile Rendering](7396-quantile-rendering.md)
Quantile Rendering (Q-Render) is a sparse, transmittance-guided sampling strategy designed to efficiently render high-dimensional feature maps from 3D Gaussians. It solves the computational bottleneck of dense volume rendering, where accumulating high-dimensional features along every ray scales poorly with feature dimensionality. Unlike conventional volume rendering, which processes all rasterized Gaussians, Q-Render shifts the Riemann partition from the spatial domain to the transmittance domain, sampling only the most representative "quantile Gaussians" that dominate the ray's transmittance profile. This approach reduces complexity from $O(N C)$ to $O(N + K C)$, enabling real-time rendering of high-dimensional data (e.g., 512-D CLIP embeddings) while maintaining segmentation accuracy.

### [4D Gaussian Splatting](7992-4d-gaussian-splatting.md)
4D Gaussian Splatting (4DGS) extends 3D Gaussian Splatting to model dynamic scenes by treating space and time as a unified 4D continuum, enabling real-time rendering of arbitrary time slices without per-frame optimization. Unlike methods that treat time as an independent dimension or rely on MLP-based deformation fields, 4DGS uses native 4D Gaussian primitives with coherent spatiotemporal covariance, allowing for smooth interpolation of geometry and appearance between discrete keyframes.

### [GauFRe](7994-gaufre.md)
GauFRe is a method for dynamic 3D Gaussian Splatting that decouples static and dynamic scene components to improve optimization stability and reconstruction quality. It addresses the difficulty of jointly optimizing deformation fields and static geometry by initializing the point cloud into separate static and dynamic subsets, optimizing them independently, and rendering them together. Unlike methods that use positional encoding for time, GauFRe employs compact MLPs for different attributes and applies specific normalization operations to scaling and rotation parameters to ensure reasonable optimization.

### [Surface and Mesh Extraction](8498-surface-and-mesh-extraction.md)

### [Rendering Pipelines and Extensions](8587-rendering-pipelines-and-extensions.md)
This family encompasses the differentiable rendering architectures and material extensions that convert 3D Gaussian primitives into 2D images, enabling both standard novel view synthesis and advanced inverse rendering tasks [G]. It solves the problem of efficiently rasterizing soft, view-dependent primitives while supporting complex appearance models such as physically based rendering (PBR) and volume visualization [G]. What sets these methods apart from standard triangle-based rasterization is the use of analytic Gaussian projection and alpha compositing, often augmented with specific data structures for density control or material decoupling [G].

#### SAGA
A 2D promptable segmentation method lifted to 3D [S4, unofficial].

#### Negative Component Splatting
A technique using negative density components in a mixture model to enhance representational power and parameter efficiency, allowing subtraction of color/density to capture complex topologies (e.g., holes) with fewer components [S5]. The rendering equation uses signed weights $w \in \mathbb{R}$ (instead of $w > 0$) in the standard alpha-compositing sum: $C(u) = \sum_{i=1}^N c_i o_i T_i^{2D}(u) \prod_{j=1}^{i-1} (1 - o_j T_j^{2D}(u))$, where $o_i$ is opacity and $T_i^{2D}$ is the 2D component function [S5]. Opacity $o$ is constrained by a tanh function during learning so that $o \in [-1, 1]$, allowing positive and negative components to dynamically change signs while being bounded [S5]. Using squared negative components (Eq. 5: $P(x) = (\sum w_i T_i(x))^2$) introduces pairwise interactions between all $K$ components, increasing evaluation complexity to $O(n^2)$, which is significantly slower. Therefore, the linear sum (Eq. 3) with signed weights is used instead [S5].

#### Solid Simulation Decoupling
Decouples solid simulation and rendering by using a separate set of sampled particles for simulation (via PBD) and interpolating deformations to trained Gaussian kernels during rendering [S6]. Use Poisson disk sampling to place simulation particles within the surface mesh explicitly reconstructed using NeuS [S6]. Perform PBD on simulation particles to animate frames [S6]. Interpolate deformation gradients and displacements from simulation particles to trained Gaussian kernels using generalized moving least squares (GMLS) [S6]. Transform solid Gaussian kernel from rest position $x_p$ to deformed position $x_{tp}$ [S6].

## Operating characteristics
*   **Time Complexity:** Training time is significantly faster than NeRF-based methods, often taking minutes to hours compared to hours to days, due to the efficient rasterization pipeline and lack of volumetric sampling [S1]. Rendering is real-time (60+ FPS) on modern GPUs because it relies on standard GPU rasterization primitives [S1].
*   **Memory Complexity:** Memory usage scales linearly with the number of Gaussians $N$. Each Gaussian stores position, rotation, scale, opacity, and spherical harmonics coefficients. The number of Gaussians can grow large (millions) to capture fine details, leading to high memory consumption [S1].
*   **Measured Performance:** Achieves high-fidelity novel view synthesis with PSNR and SSIM metrics competitive with or exceeding NeRF methods on standard benchmarks (e.g., Mip-NeRF 360, NeRF Synthetic) [S3, unofficial].
*   **Scaling:** Scales well to large scenes with appropriate data structures (e.g., tile-based rasterization, hierarchical structures) [S1].
*   **Strengths:** Real-time rendering, fast training, high visual fidelity, explicit representation allows for easy editing and manipulation [S1].
*   **Limitations:** High memory usage for large scenes, potential for artifacts in regions with insufficient view coverage, and sensitivity to camera pose accuracy [S1].
*   **When to use:** When real-time rendering and fast training are priorities, and the scene can be represented well by Gaussian primitives.
*   **When not to use:** When memory is extremely constrained, or when the scene has complex, thin structures that are difficult to represent with ellipsoids (though variants like Tetrahedron Splatting address this) [S2].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1]. It emerged from the need to overcome the computational bottlenecks of neural radiance fields (NeRFs) while maintaining high visual quality. The method builds on the idea of explicit scene representations, using Gaussian primitives as basis functions for volume rendering. It has since become a popular baseline for novel view synthesis and has inspired numerous variants and applications in dynamic scenes, human avatars, and domain-specific tasks [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S3] Direct Conditional Parameterization for N-Dimensional Splatting, https://arxiv.org/abs/2610.22778, fetched 2026-10-30 (unofficial)
[S4] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S5] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
[S6] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
