# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific hyperparameters, optimization strategies, and engineering implementations required to train and render 3D Gaussian Splatting (3DGS) models effectively. It addresses the gap between the theoretical formulation of differentiable volume rendering and the practical execution on hardware, focusing on stability, speed, and convergence. It sets itself apart from sibling articles by providing concrete numerical values, code-level modifications, and hardware-specific optimizations (such as half-precision rasterization) rather than abstract algorithmic descriptions.

## How it works
The implementation relies on a differentiable rasterizer (typically `gsplat`) that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters.

### Inputs and Outputs
*   **Inputs:** Camera poses ($C_i$), intrinsics, training images, and an initial point cloud (SfM points).
*   **Outputs:** Rendered images, gradients for Gaussian attributes, and the final optimized Gaussian scene representation.

### Data Structures
*   **Gaussian Attributes:** Each Gaussian $i$ is defined by position $\mathbf{t}_i$, rotation $\mathbf{q}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and Spherical Harmonics (SH) coefficients $\mathbf{c}_{i, l, m}$.
*   **Volume Representation:** For large-scale scenes, Gaussians may be organized within a 3D sparse voxel grid (e.g., using `torchsparse`) to accelerate culling and sorting [S2].
*   **Feature Vectors:** During rasterization, feature vectors are fetched using half-precision (FP16) memory accesses to reduce register pressure in CUDA kernels [S1].

### Optimization Steps
1.  **Initialization:** Initialize Gaussians from Structure-from-Motion (SfM) points. Calculate scene extent $e_{\text{scene}}$ or corrected extent $e'_{\text{scene}}$ to determine initial splitting/pruning thresholds [S3].
2.  **Warm-up Phase:** Optimize static Gaussians for the initial 3000 iterations without deformation fields (if applicable) [S4, unofficial].
3.  **Joint Optimization:** Jointly optimize Gaussians and deformation fields (if dynamic) [S4, unofficial].
4.  **Supervision:** For each iteration, randomly select a single image from a random scene as supervision [S2].
5.  **Loss Calculation:** Compute the total loss $\mathcal{L}$, typically combining color loss ($\mathcal{L}_{\text{color}}$) and auxiliary losses (e.g., scale loss $\mathcal{L}_{\text{scales}}$) [S4, unofficial].
6.  **Gradient Update:** Use the Adam optimizer to update parameters [S2].

### Formulas and Parameters
*   **Optimizer:** Adam with learning rate $\eta = 1 \times 10^{-3}$ [S2].
*   **Total Loss:**
    $$ \mathcal{L} = \mathcal{L}_{\text{color}} + \lambda_1 \cdot \mathcal{L}_{\text{scales}} $$
    where $\lambda_1 = 0.1$ [S4, unofficial].
*   **Scene Extent (Corrected):** To avoid bias from camera distribution, use:
    $$ e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2 $$
    where $\bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i$ is the average camera position, $N_{\text{SfM}}$ is the number of SfM points, and $p_i$ is the $i$-th SfM point [S3].
*   **SH Degree:** Set to 1 for simplicity and speed, as higher-degree coefficients often remain unchanged during training [S2][S6, unofficial].
*   **Background Color:** Set RGB background value `bg` to a gray value in the range $0.3 \le \text{bg} \le 0.5$ (e.g., 0.4) to improve training speed by ~21.85% without degrading quality [S6, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
gaussians = initialize_from_sfm(points)
optimizer = Adam(params=gaussians.parameters(), lr=1e-3)
bg_color = torch.tensor([0.4, 0.4, 0.4]) # Gray background

# Training Loop
for iteration in range(num_iterations):
    # 1. Select supervision
    camera_idx = random.choice(cameras)
    gt_image = get_gt_image(camera_idx)
    
    # 2. Forward Pass (Rasterization)
    # Uses gsplat with half-precision fetches [S1]
    rendered_image = rasterize(gaussians, camera_idx, bg_color)
    
    # 3. Loss Calculation
    # L_color (e.g., L1 + SSIM)
    loss_color = compute_color_loss(rendered_image, gt_image)
    
    # L_scales (if dynamic) [S4, unofficial]
    loss_scales = compute_scale_loss(gaussians)
    
    # Total Loss
    lambda_1 = 0.1
    loss = loss_color + lambda_1 * loss_scales
    
    # 4. Backward Pass
    optimizer.zero_grad()
    loss.backward()
    
    # 5. Update
    optimizer.step()
    
    # 6. Densification/Pruning (Standard 3DGS logic)
    # Apply gradient threshold schedule [S3]
    if iteration < 3000:
        # Warm-up: optimize static Gaussians only [S4, unofficial]
        pass
    else:
        # Joint optimization [S4, unofficial]
        densify_and_prune(gaussians)
```

### Edge Cases and Pitfalls
*   **Scene Extent Bias:** Default scene extent calculation based on furthest camera distance can be biased if cameras are clustered (e.g., orbiting a small object), leading to incorrect splitting/pruning [S3]. Use the corrected SfM-based extent [S3].
*   **SH Degree:** Using SH degree > 0 for color is often useless as these coefficients remain unchanged during training, adding computational overhead [S6, unofficial].
*   **Background Color:** Extreme background values may negatively impact learning rate, training loss, and PSNR [S6, unofficial].
*   **Half-Precision Stability:** Apply an automatic scaling factor to loss values to improve half-precision training stability [S1].

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
Reduces parameter count by distilling high-fidelity third-order SH coefficients into a compact first-order representation.

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework for unbiased surface extraction.

### [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)
Replaces Gaussian radial basis function with an inverse multiquadric function for slower decay.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
Derives analytical Jacobians of SE(3) camera pose for efficient gradient-based camera tracking.

### [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)
Decouples visibility threshold from opacity to allow aggressive truncation and gradient flow for zero-opacity Gaussians.

### [ASCII-Based Data Compression](11820-ascii-based-data-compression.md)
Compresses data into ASCII strings to accelerate training I/O.

### [DAVIGS](12021-davigs.md)
Mitigates appearance variation-induced floaters by transforming rendered images using 3D consistent features.

### [Sensitivity-Aware Vector Clustering](12167-sensitivity-aware-vector-clustering.md)
Compresses SH coefficients and shape features using sensitivity-aware vector quantization.

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Aligns software strategies with hardware bottlenecks (Roofline model) for memory and compute optimization.

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes scenes to incorporate semantic language features alongside photometric rendering.

### Corrected Scene-Extent
A method to calculate scene extent based on the SfM point cloud rather than camera positions to avoid bias from camera distribution (e.g., orbiting a small object in a large environment) [S3]. The corrected extent is $e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2$ [S3]. The default extent $e_{\text{scene}} = 1.1 \cdot \max_{i=1}^{N_{\text{cam}}} \|\bar{C} - C_i\|_2$ causes bias if cameras are clustered, leading to incorrect splitting/pruning [S3].

### Optimization Strategy
Optimize static Gaussians for the initial 3000 iterations as a warm-up phase, then jointly optimize both the Gaussians and the deformation field [S4, unofficial]. The total loss is $\mathcal{L} = \mathcal{L}_{\text{color}} + \lambda_1 \cdot \mathcal{L}_{\text{scales}}$ with $\lambda_1 = 0.1$ [S4, unofficial].

### POG (Potentially Over-large Gaussians)
POG improves LPIPS slightly but degrades PSNR significantly compared to Hard Gaussians (HG) because it overgrows already learned areas, making them susceptible to redundant Gaussians [S5]. In Tanks&Temples, POG yields PSNR 26.26, SSIM 0.879, LPIPS 0.121, while HG yields PSNR 26.36, SSIM 0.880, LPIPS 0.126 [S5].

### SH Degree Zero Optimization
Setting the maximum degree of Spherical Harmonics (SH) coefficients to 0 removes superfluous operations and improves training speed by ~16.25-16.57% compared to the default degree 3, as higher-degree SH coefficients for color remain unchanged during training [S6, unofficial].

### Background Color Optimization
Changing the default white background (RGB 1.0) to a gray value (optimal range $0.3 \le \text{bg} \le 0.5$, e.g., 0.4) improves training speed by ~21.85% on the truck dataset while maintaining rendering quality [S6, unofficial].

### Selective Defocus Blurring
Predicts a per-Gaussian scale offset $\delta s_j$ using a compact MLP to selectively enlarge covariances, creating pixel-wise variable blur kernels that match spatially varying scene blurriness [S7]. If $\delta s_j$ is constant, it degrades to a uniform Gaussian blur [S7].

## Operating characteristics
*   **Time Complexity:** Training speed is significantly impacted by SH degree and background color. Setting SH degree to 0 improves speed by ~16.57% [S6, unofficial]. Changing background to gray improves speed by ~21.85% [S6, unofficial].
*   **Memory Complexity:** Half-precision memory fetches on feature vectors reduce register pressure during rasterization [S1]. Large-scale scenes may require 3D sparse convolution implementations (e.g., `torchsparse`) to manage memory [S2].
*   **Performance:** The method achieves high-fidelity rendering with real-time capabilities. For example, EVolSplat achieves 83.81 FPS on Waymo with 10.41 GB memory usage [S2].
*   **Strengths:** Half-precision rasterization improves stability and speed [S1]. Corrected scene extent prevents optimization bias [S3].
*   **Limitations:** Default scene extent calculation is biased by camera distribution [S3]. Higher SH degrees add computational cost without quality improvement in some cases [S6, unofficial].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1, S2, S3][S4, unofficial][S5][S6, unofficial][S7]. The implementation details described here are derived from various extensions and optimizations of the original method, including `gsplat` [S1, S2], 4D Gaussian Splatting [S4, unofficial], Hard Gaussian Splatting [S5], and various efficiency improvements [S6, unofficial][S7].

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Alias-free 4D Gaussian Splatting, https://arxiv.org/abs/2511.18367v1, fetched 2026-10-26 (unofficial)
[S5] Pushing Rendering Boundaries: Hard Gaussian Splatting, https://arxiv.org/abs/2412.04826v1, fetched 2026-10-26
[S6] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S7] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
