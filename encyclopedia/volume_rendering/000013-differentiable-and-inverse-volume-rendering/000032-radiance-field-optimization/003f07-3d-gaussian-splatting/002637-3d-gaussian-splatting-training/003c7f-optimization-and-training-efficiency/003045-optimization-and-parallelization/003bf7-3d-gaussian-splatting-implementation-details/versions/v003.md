# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific hyperparameters, data structures, and optimization strategies required to implement 3D Gaussian Splatting (3DGS) training and rendering pipelines effectively. It addresses common implementation pitfalls such as scene extent bias, half-precision instability, and redundant SH computations. While the parent article [Optimization and Parallelization](6233-optimization-and-parallelization.md) covers the general differentiable rasterization and gradient flow, this article provides the concrete configuration values and code-level adjustments necessary for stable convergence and performance optimization.

## How it works

### Inputs and Outputs
*   **Inputs**: Initial SfM point cloud (position, normal, color), camera poses (extrinsics, intrinsics), and training image pairs.
*   **Outputs**: Optimized set of 3D Gaussians (position $\mathbf{x}$, covariance $\Sigma$, opacity $\alpha$, spherical harmonic coefficients $\mathbf{c}$, scale $\mathbf{s}$) and rendered images.

### Data Structures and Layouts
The core data structure is the Gaussian primitive. In standard implementations (e.g., `gsplat`), Gaussians are stored in a Structure of Arrays (SoA) layout for efficient vectorization during rasterization [S1].

1.  **Position**: `float3` (x, y, z).
2.  **Rotation**: `float4` (quaternion, normalized).
3.  **Scale**: `float3` (log-scale storage is common to ensure positivity).
4.  **Opacity**: `float` (sigmoid output).
5.  **SH Coefficients**: `float` array. For degree $d$, there are $(d+1)^2$ coefficients per RGB channel.
    *   Degree 0: 1 coeff/channel.
    *   Degree 1: 4 coeffs/channel.
    *   Degree 3: 16 coeffs/channel.
6.  **Covariance**: Derived from Rotation and Scale. $\Sigma = R S S^T R^T$.

**Memory Optimization**: To reduce register pressure in CUDA kernels, feature vectors (including SH coefficients) are fetched in half-precision (FP16) during both forward and backward passes [S1].

### Parameters and Configuration

#### Optimizer and Learning Rate
*   **Optimizer**: Adam [S2].
*   **Learning Rate**: $1 \times 10^{-3}$ for Gaussian parameters [S2].
*   **Loss Scaling**: An automatic scaling factor is applied to loss values to improve stability during half-precision (FP16) training [S1].

#### Spherical Harmonics (SH) Degree
*   **Standard**: Degree 3 is default in original 3DGS.
*   **Optimization**: Setting SH degree to 0 can improve training speed by ~16.25–16.57% with negligible quality loss, as higher-degree color coefficients often remain unchanged during training [S7, unofficial].
*   **Alternative**: Degree 1 is used in some urban scene implementations (e.g., EVolSplat) to balance fidelity and speed [S2].

#### Background Color
*   **Default**: White (RGB 1.0).
*   **Optimization**: Setting background to gray (RGB $bg$ where $0.3 \le bg \le 0.5$) improves training speed. Specifically, $bg=0.4$ reduces processing time by ~21.85% on certain datasets without degrading PSNR [S7, unofficial].

#### Scene Extent Calculation
The default 3DGS scene extent $e_{\text{scene}}$ is calculated based on camera positions:
$$ e_{\text{scene}} = 1.1 \cdot \max_{i=1}^{N_{\text{cam}}} \|\bar{C} - C_i\|_2 $$
where $\bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i$ [S3].

**Pitfall**: This formula biases the extent towards camera distribution. If cameras orbit a small object in a large environment, $e_{\text{scene}}$ becomes artificially small, causing incorrect splitting/pruning of Gaussians [S3].

**Correction**: Use the SfM point cloud to calculate extent:
$$ e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2 $$
where $p_i$ are SfM points and $N_{\text{SfM}}$ is the number of SfM points [S3].

### Training Steps and Strategies

1.  **Initialization**: Initialize Gaussians from SfM points. Apply corrected scene extent if cameras are clustered [S3].
2.  **Warm-up Phase**: Optimize static Gaussians for the first 3000 iterations [S4, unofficial].
3.  **Joint Optimization**: After warm-up, jointly optimize Gaussians and any deformation fields (for 4DGS) [S4, unofficial].
4.  **Supervision**: Randomly select a single image from a random scene as supervision per iteration (in multi-scene settings) [S2].
5.  **Loss Function**:
    $$ \mathcal{L} = \mathcal{L}_{\text{color}} + \lambda_1 \mathcal{L}_{\text{scales}} $$
    where $\lambda_1 = 0.1$ [S4, unofficial]. Additional weights like $\lambda_v = 0.2$ may apply to specific terms [S4, unofficial].

### Edge Cases and Pitfalls

*   **Potentially Over-large Gaussians (POG)**: Allowing Gaussians to grow excessively large can degrade PSNR by creating redundant Gaussians in already learned areas. Use Hard Gaussian growing (HGS) instead to target hard areas without overgrowth [S6].
*   **Half-Precision Instability**: FP16 rasterization requires loss scaling to prevent gradient underflow/overflow [S1].
*   **Scene Bias**: Using camera-based extent for scenes with clustered cameras leads to poor reconstruction [S3].

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
First-order Spherical Harmonics (SH) Distillation reduces the parameter count and computational cost of 3D Gaussian Splatting by distilling high-fidelity third-order SH coefficients from a pretrained teacher model into a compact first-order representation for a student model. Unlike standard degree reduction which simply truncates coefficients, this method uses knowledge distillation to preserve rendering quality despite the aggressive reduction from 16 coefficients per color channel to 4. It sets itself apart from sibling methods like [RoofGS](9830-roofgs.md) or [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md) by specifically targeting the appearance representation's dimensionality rather than the geometric or sorting pipeline.

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
The Continuous Attenuation Model reformulates the rasterization-based image formation of 3D Gaussian Splatting into a volumetric ray-marching framework by mapping Gaussian primitives to a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. This equivalence allows the application of theoretical results from Other-at-Volume (OaV) for surface extraction, ensuring that transmittance isosurfaces remain unbiased and ellipsoidal. It distinguishes itself from prior attenuation approximations like those in GGGS [57] by enforcing geometric constraints that prevent surface erosion and multi-view inconsistency.

### RoofGS
RoofGS is an end-to-end acceleration method for 3D Gaussian Splatting (3DGS) that optimizes the rendering pipeline by aligning software strategies with hardware bottlenecks identified via the Roofline model [S1]. It solves the problem of uncoordinated optimization in 3DGS, where different stages are constrained by different resources (memory bandwidth vs. compute throughput) [S1]. It sets itself apart from generic optimization by explicitly using the Roofline model to identify the "knee" point and applying memory-centric optimizations to stages below it and compute-centric optimizations to stages near it [S1].

### [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)
Inverse Multiquadric Splatting is a radiance-field optimization technique that replaces the standard Gaussian radial basis function with an inverse multiquadric function to alter the decay profile of splats during rasterization. It differs from standard 3D Gaussian Splatting by using a slower-decaying tail, which requires a specific correction factor for the covariance matrix to maintain visual fidelity.

### [Tile-Based Comparison-Free Sorting](9991-tile-based-comparison-free-sorting.md)
Tile-Based Comparison-Free Sorting restricts depth sorting operations to individual rendering tiles to avoid global synchronization, reduce hardware complexity, and achieve fixed latency per output. Unlike full-scene sorting which causes excess traffic and latency spikes, this method uses $\langle \text{tile-id}, \text{depth} \rangle$ keys to order Gaussians front-to-back, matching the $\alpha$-blending requirement with deterministic, fixed per-output latency. It is specifically designed for hardware accelerators where variable latency and high memory overhead of radix sort are prohibitive.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
This method derives analytical Jacobians of the SE(3) camera pose with respect to 3D Gaussian primitives to enable efficient gradient-based camera tracking in SLAM systems. It solves the problem of high computational overhead associated with automatic differentiation during rasterization, ensuring minimal parameterization using Lie algebra. Unlike prior ray-intersection methods that assume fewer Gaussians, this approach is applicable to the EWA splatting and 3D Gaussian Splatting (3DGS) rendering pipeline.

### [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)
Opacity-Independent Gaussian Truncation is a modification to the fragment culling mechanism in 3D Gaussian Splatting that decouples the visibility threshold from the Gaussian's opacity parameter. It solves the problem where the original implementation prevents consistent truncation at fewer than ~3.03 standard deviations because low-opacity Gaussians are discarded before they can receive gradients, and enforces an upper bound on the transparency threshold due to opacity reset clipping. It sets itself apart from the default 3DGS truncation by evaluating the Gaussian spatial response independently of the opacity scalar, allowing for aggressive truncation (e.g., 1σ) and enabling gradient flow for zero-opacity Gaussians.

### [ASCII-Based Data Compression](11820-ascii-based-data-compression.md)
This method accelerates data transmission during the training of 3D Gaussian Splatting models by compressing floating-point and integer values into ASCII strings. It distinguishes itself from sibling methods by targeting the I/O bottleneck of training speed rather than rendering quality or memory footprint, achieving approximately 6% training speed improvement.

### DAVIGS
DAVIGS is a plug-and-play appearance modeling method for 3D Gaussian Splatting that mitigates appearance variation-induced floaters and color distortions by transforming rendered images at the image level using 3D consistent features [S1]. Unlike coupled methods that manipulate Gaussian properties (slowing rendering) or decoupled methods that only account for mild global variations, DAVIGS decouples global view-dependent appearance variations (e.g., daylight, ISPs) from local spatial changes (e.g., street lamps) by querying multi-resolution hash grids based on back-projected 3D coordinates [S1]. It builds 3D consistency across views implicitly while maintaining real-time rendering speeds by discarding the appearance module after optimization [S1].

## Operating characteristics

*   **Memory**: Single GPU (e.g., A100) limits training to ~11.2M Gaussians [S5, unofficial]. Distributed training across HPC nodes is required for larger scenes (e.g., 106.7M Gaussians) [S5, unofficial].
*   **Speed**: SH Degree 0 optimization yields ~16.25% speedup [S7, unofficial]. Background color optimization (bg=0.4) yields ~21.85% speedup [S7, unofficial].
*   **Scaling**: Distributed training shows 3.1× speedup scaling from 4 to 8 nodes for large datasets (106.7M Gaussians) [S5, unofficial]. Moderate datasets show limited gains (1.4× from 2 to 4 nodes) [S5, unofficial].
*   **Quality**: POG (Potentially Over-large Gaussians) degrades PSNR significantly compared to HGS (Hard Gaussians) [S6]. Corrected scene extent prevents bias in splitting/pruning [S3].

## Background

3D Gaussian Splatting was introduced by Kerbl et al. [12] (cited in [S7, unofficial] as "original code of 3D Gaussian Splatting model [12]"). The method uses explicit 3D Gaussian primitives for real-time radiance field rendering. Subsequent implementations have focused on optimizing the training pipeline, including half-precision rasterization [S1], distributed training [S5, unofficial], and hyperparameter tuning for SH and background colors [S7, unofficial]. The `gsplat` library is a common implementation target [S1, S2].

## Sources

[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Alias-free 4D Gaussian Splatting, https://arxiv.org/abs/2511.18367v1, fetched 2026-10-26 (unofficial)
[S5] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S6] Pushing Rendering Boundaries: Hard Gaussian Splatting, https://arxiv.org/abs/2412.04826v1, fetched 2026-10-26
[S7] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
