# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific hyperparameters, data structures, and optimization strategies required to implement the 3D Gaussian Splatting (3DGS) training pipeline efficiently. It addresses the gap between the theoretical formulation of differentiable volume rendering and the practical constraints of GPU memory, precision, and convergence stability. Key distinctions from standard implementations include the use of half-precision rasterization kernels, specific background color tuning, and corrected scene extent calculations to prevent bias in adaptive density control.

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Initial point cloud from Structure-from-Motion (SfM) [S3].
    *   Camera poses and intrinsics for $N_{\text{cam}}$ views [S3].
    *   Training images (randomly sampled per iteration) [S2].
    *   Loss weights: $\lambda_1$ (scale loss), $\lambda_v$ (velocity/regularization loss) [S4, unofficial].
*   **Outputs:**
    *   Optimized Gaussian parameters: Position $\mathbf{x}$, Covariance $\Sigma$ (or scale $\mathbf{s}$ and rotation $\mathbf{q}$), Opacity $\alpha$, Spherical Harmonics coefficients $\mathbf{c}_{SH}$, and potentially deformation fields $\mathbf{v}$ [S4, unofficial].
    *   Rendered images $\mathbf{I}_{rendered}$ and corresponding gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ [S2].

### Data Structures
The implementation relies on the `gsplat` library [S2] with custom CUDA kernels based on the 3DGUT formulation [S1].

1.  **Gaussian Attributes:**
    *   **Position:** 3D float vector $\mathbf{x} \in \mathbb{R}^3$.
    *   **Covariance:** Parameterized as scale $\mathbf{s} \in \mathbb{R}^3$ and rotation $\mathbf{q} \in \mathbb{R}^4$ (quaternion). The covariance matrix $\Sigma$ is reconstructed as $\Sigma = R S S^T R^T$, where $R$ is the rotation matrix from $\mathbf{q}$ and $S$ is the diagonal matrix from $\mathbf{s}$ [G].
    *   **Opacity:** Scalar $\alpha \in [0, 1]$.
    *   **Color:** Spherical Harmonics (SH) coefficients. Default degree is 3, but degree 1 or 0 is used for efficiency [S2][S6, unofficial].
2.  **Memory Layout:**
    *   Feature vectors (SH coefficients, position, etc.) are fetched in **half-precision (FP16)** during rasterization to reduce register pressure in CUDA kernels [S1].
    *   The neural network components (if any, e.g., for refinement) use tiny-cuda-nn's JIT-compiled cooperative vector MLPs, also in FP16 [S1].

### Parameters and Configuration

#### Optimization Settings
*   **Optimizer:** Adam [S2].
*   **Learning Rate:** $1 \times 10^{-3}$ [S2].
*   **Loss Function:**
    $$ \mathcal{L} = \mathcal{L}_{color} + \lambda_1 \mathcal{L}_{scales} + \lambda_v \mathcal{L}_{other} $$
    Where $\mathcal{L}_{color}$ is the standard L1/SSIM loss, $\mathcal{L}_{scales}$ regularizes Gaussian scales, and $\lambda_1 = 0.1$, $\lambda_v = 0.2$ [S4, unofficial].
*   **Stability:** An automatic scaling factor is applied to loss values to improve half-precision training stability [S1].

#### Scene and Volume Setup
*   **Scene Extent:**
    *   **Default (Biased):** $e_{\text{scene}} = 1.1 \cdot \max_{i=1}^{N_{\text{cam}}} \|\bar{C} - C_i\|_2$, where $\bar{C} = \frac{1}{N_{\text{cam}}} \sum C_i$ [S3]. This is prone to bias if cameras are clustered [S3].
    *   **Corrected:** $e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2$, where $p_i$ are SfM points [S3]. This is recommended to avoid incorrect splitting/pruning [S3].
*   **Volume Dimensions (Example: Urban/StreetGaussian):**
    *   Height: 12.8m, Width: 32m, Length: 64m [S2].
    *   Voxel Size: 0.1m [S2].
    *   Resulting Grid: $128 \times 320 \times 640$ [S2].
    *   Waymo Zero-shot Range: (50m, 20m, 128m) [S2].

#### SH Degree and Background
*   **SH Degree:** Set to 1 [S2] or 0 [S6, unofficial]. Setting to 0 removes superfluous operations, improving training speed by ~16.25–16.57% with negligible quality loss, as higher-degree color coefficients often remain unchanged during training [S6, unofficial].
*   **Background Color:**
    *   Default: White (1.0, 1.0, 1.0) [S6, unofficial].
    *   Optimized: Gray value $bg \in [0.3, 0.5]$ [S6, unofficial].
    *   Effect: Setting $bg=0.4$ reduces processing time by ~21.85% on the truck dataset without degrading PSNR [S6, unofficial].

### Algorithm Steps

1.  **Initialization:**
    *   Initialize Gaussians from SfM points [S3].
    *   Calculate scene extent using the corrected formula $e'_{\text{scene}}$ to ensure proper densification bounds [S3].
    *   Set SH degree to 0 or 1 for efficiency [S2][S6, unofficial].
    *   Set background color to $bg \in [0.3, 0.5]$ [S6, unofficial].

2.  **Training Loop (Iterative):**
    *   **Sampling:** Randomly select a single image from a random scene as supervision [S2].
    *   **Warm-up (Static):** For the first 3000 iterations, optimize only static Gaussians [S4, unofficial].
    *   **Joint Optimization:** After warm-up, jointly optimize Gaussians and deformation fields (if applicable) [S4, unofficial].
    *   **Rasterization:**
        *   Project 3D Gaussians to 2D screen space.
        *   Sort Gaussians by depth.
        *   Render image using FP16 CUDA kernels via `gsplat` [S1, S2].
        *   Apply automatic loss scaling for FP16 stability [S1].
    *   **Backpropagation:**
        *   Compute gradients of $\mathcal{L}$ w.r.t. Gaussian parameters.
        *   Update parameters using Adam optimizer ($lr = 10^{-3}$) [S2].
    *   **Densification/Pruning:**
        *   Use the corrected scene extent $e'_{\text{scene}}$ to guide splitting and cloning thresholds [S3].
        *   Avoid "Potentially Over-large Gaussians" (POG) which degrade PSNR; use Hard Gaussian growing (HG) if necessary [S5].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_3dgs(gaussians, cameras, images, config):
    """
    Config includes:
    - optimizer: Adam(lr=1e-3)
    - sh_degree: 0 or 1
    - bg_color: float in [0.3, 0.5]
    - lambda1: 0.1
    - lambda_v: 0.2
    - max_iterations: e.g., 30000
    - use_corrected_extent: True
    """
    
    # 1. Initialization
    if config.use_corrected_extent:
        # Calculate average camera position
        avg_cam_pos = mean([cam.position for cam in cameras])
        # Calculate extent based on SfM points
        scene_extent = mean([norm(avg_cam_pos - p) for p in gaussians.sfm_points])
    else:
        # Default biased extent
        avg_cam_pos = mean([cam.position for cam in cameras])
        scene_extent = 1.1 * max([norm(avg_cam_pos - cam.position) for cam in cameras])
        
    gaussians.set_scene_extent(scene_extent)
    gaussians.set_sh_degree(config.sh_degree)
    gaussians.set_background_color(config.bg_color)
    
    optimizer = Adam(gaussians.parameters(), lr=1e-3)
    
    for iteration in range(1, config.max_iterations + 1):
        # 2. Sampling
        # Randomly select one image and its camera pose
        idx = random_choice(len(images))
        image = images[idx]
        camera = cameras[idx]
        
        # 3. Forward Pass (Rasterization)
        # Use gsplat with FP16 kernels
        rendered_image = rasterize_gaussians(
            gaussians=gaussians,
            camera=camera,
            background_color=config.bg_color,
            precision='fp16'
        )
        
        # 4. Loss Computation
        # L_color is typically L1 or SSIM
        loss_color = compute_color_loss(rendered_image, image)
        
        # L_scales regularization
        loss_scales = compute_scale_loss(gaussians.scales)
        
        # Total Loss
        loss = loss_color + config.lambda1 * loss_scales + config.lambda_v * compute_velocity_loss(gaussians)
        
        # Apply automatic scaling for FP16 stability
        loss = apply_loss_scaling(loss)
        
        # 5. Backward Pass
        optimizer.zero_grad()
        loss.backward()
        
        # 6. Optimization Step
        optimizer.step()
        
        # 7. Adaptive Densification (Every N steps)
        if iteration % 100 == 0:
            # Use scene_extent for thresholding
            densify_gaussians(gaussians, scene_extent, iteration)
            prune_gaussians(gaussians, scene_extent)
            
        # 8. Transition to Joint Optimization after warm-up
        if iteration == 3000:
            enable_deformation_field_optimization(gaussians)
```

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
First-order Spherical Harmonics (SH) Distillation reduces the parameter count and computational cost of 3D Gaussian Splatting by distilling high-fidelity third-order SH coefficients from a pretrained teacher model into a compact first-order representation for a student model. Unlike standard degree reduction which simply truncates coefficients, this method uses knowledge distillation to preserve rendering quality despite the aggressive reduction from 16 coefficients per color channel to 4. It sets itself apart from sibling methods like [RoofGS](9830-roofgs.md) or [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md) by specifically targeting the appearance representation's dimensionality rather than the geometric or sorting pipeline.

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
The Continuous Attenuation Model reformulates the rasterization-based image formation of 3D Gaussian Splatting into a volumetric ray-marching framework by mapping Gaussian primitives to a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. This equivalence allows the application of theoretical results from Other-at-Volume (OaV) for surface extraction, ensuring that transmittance isosurfaces remain unbiased and ellipsoidal. It distinguishes itself from prior attenuation approximations like those in GGGS [57] by enforcing geometric constraints that prevent surface erosion and multi-view inconsistency.

### [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)
Inverse Multiquadric Splatting is a radiance-field optimization technique that replaces the standard Gaussian radial basis function with an inverse multiquadric function to alter the decay profile of splats during rasterization. It differs from standard 3D Gaussian Splatting by using a slower-decaying tail, which requires a specific correction factor for the covariance matrix to maintain visual fidelity.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
This method derives analytical Jacobians of the SE(3) camera pose with respect to 3D Gaussian primitives to enable efficient gradient-based camera tracking in SLAM systems. It solves the problem of high computational overhead associated with automatic differentiation during rasterization, ensuring minimal parameterization using Lie algebra. Unlike prior ray-intersection methods that assume fewer Gaussians, this approach is applicable to the EWA splatting and 3D Gaussian Splatting (3DGS) rendering pipeline.

### [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)
Opacity-Independent Gaussian Truncation is a modification to the fragment culling mechanism in 3D Gaussian Splatting that decouples the visibility threshold from the Gaussian's opacity parameter. It solves the problem where the original implementation prevents consistent truncation at fewer than ~3.03 standard deviations because low-opacity Gaussians are discarded before they can receive gradients, and enforces an upper bound on the transparency threshold due to opacity reset clipping. It sets itself apart from the default 3DGS truncation by evaluating the Gaussian spatial response independently of the opacity scalar, allowing for aggressive truncation (e.g., 1σ) and enabling gradient flow for zero-opacity Gaussians.

### [ASCII-Based Data Compression](11820-ascii-based-data-compression.md)
This method accelerates data transmission during the training of 3D Gaussian Splatting models by compressing floating-point and integer values into ASCII strings. It distinguishes itself from sibling methods by targeting the I/O bottleneck of training speed rather than rendering quality or memory footprint, achieving approximately 6% training speed improvement.

### DAVIGS
DAVIGS is a plug-and-play appearance modeling method for 3D Gaussian Splatting that mitigates appearance variation-induced floaters and color distortions by transforming rendered images at the image level using 3D consistent features [S1]. Unlike coupled methods that manipulate Gaussian properties (slowing rendering) or decoupled methods that only account for mild global variations, DAVIGS decouples global view-dependent appearance variations (e.g., daylight, ISPs) from local spatial changes (e.g., street lamps) by querying multi-resolution hash grids based on back-projected 3D coordinates [S1]. It builds 3D consistency across views implicitly while maintaining real-time rendering speeds by discarding the appearance module after optimization [S1].

### [Sensitivity-Aware Vector Clustering](12167-sensitivity-aware-vector-clustering.md)
Sensitivity-Aware Vector Clustering is a compression scheme for 3D Gaussian Splatting that reduces storage requirements by clustering Spherical Harmonics (SH) coefficients and Gaussian shape features (scale and rotation) into compact codebooks using sensitivity-aware vector quantization. It differs from standard post-training quantization by deriving a sensitivity measure for each scene parameter to guide the clustering process, prioritizing the preservation of parameters that significantly affect image quality.

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Sorting and Rendering Pipeline Optimization addresses the computational bottlenecks in 3D Gaussian Splatting (3DGS) rendering, specifically the high memory bandwidth requirements of depth sorting and the compute intensity of alpha blending. It sets itself apart from generic optimization by explicitly aligning software strategies with hardware bottlenecks identified via the Roofline model, applying memory-centric optimizations to memory-bound stages and compute-centric optimizations to compute-bound stages.

## Operating characteristics

*   **Time Complexity:** The rasterization step is $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is the average number of Gaussians per pixel. Sorting adds $O(M \log M)$ per pixel. The use of FP16 kernels and SH degree reduction (0 or 1) significantly reduces compute time. For example, setting SH degree to 0 improves training speed by ~16.25–16.57% [S6, unofficial], and optimizing background color can reduce processing time by ~21.85% [S6, unofficial].
*   **Memory Complexity:** Memory usage is dominated by the storage of Gaussian attributes ($O(N_{gaussians})$) and the depth-sorted lists. FP16 fetches reduce register pressure and memory bandwidth requirements [S1].
*   **Performance:**
    *   On Waymo dataset, EVolSplat (using these optimizations) achieves 83.81 FPS with 10.41 GB memory [S2].
    *   PSNR/SSIM/LPIPS metrics are comparable to standard 3DGS when using corrected scene extent and appropriate SH degrees [S2, S3][S6, unofficial].
*   **Strengths:**
    *   High training speed through FP16 rasterization and SH simplification [S1][S6, unofficial].
    *   Robust densification with corrected scene extent prevents bias in non-uniform camera distributions [S3].
    *   Stable convergence with Adam optimizer and loss scaling [S1, S2].
*   **Limitations:**
    *   SH degree 0 may lose high-frequency view-dependent details if the scene has strong specularities [S6, unofficial].
    *   Corrected scene extent requires access to the full SfM point cloud, which may not be available in all pipelines [S3].
    *   POG (Potentially Over-large Gaussians) can degrade PSNR if not handled via HG (Hard Gaussian) growing [S5].

## Background

3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1, S2]. The method represents scenes as a set of 3D anisotropic Gaussians and renders them using a differentiable rasterizer. The implementation details described here stem from subsequent optimizations and variations aimed at improving training efficiency, stability, and robustness. Key contributions include the `gsplat` library [S2], the 3DGUT formulation [S1], and various heuristic improvements like background color tuning [S6, unofficial] and scene extent correction [S3]. The method is widely used in real-time novel view synthesis, SLAM, and dynamic scene reconstruction [S2][S4, unofficial].

## Sources

[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Alias-free 4D Gaussian Splatting, https://arxiv.org/abs/2511.18367v1, fetched 2026-10-26 (unofficial)
[S5] Pushing Rendering Boundaries: Hard Gaussian Splatting, https://arxiv.org/abs/2412.04826v1, fetched 2026-10-26
[S6] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
