# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific software engineering choices, hyperparameters, and numerical stability techniques required to implement the 3D Gaussian Splatting (3DGS) training pipeline efficiently. It addresses the gap between the theoretical differentiable rendering formulation and a production-ready CUDA implementation, focusing on precision management, library selection, and scene initialization corrections. It sets itself apart from general optimization articles by providing concrete values for loss weights, optimizer settings, and memory fetch strategies that directly impact training convergence and hardware utilization [S1, S2][S4, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Initial 3D Gaussian point cloud (from SfM).
*   Camera poses and intrinsics.
*   Reference images.
*   Hyperparameters: Learning rate, loss weights, SH degree, volume dimensions.

**Outputs:**
*   Optimized Gaussian parameters (position, covariance, opacity, color/SH coefficients).
*   Rendered images and loss metrics.

### Data Structures
*   **Gaussian Attributes:** Stored in contiguous memory for rasterization. Includes position ($\mathbf{x}$), covariance matrix ($\mathbf{\Sigma}$, decomposed into rotation $\mathbf{R}$ and scale $\mathbf{s}$), opacity ($\alpha$), and Spherical Harmonics coefficients.
*   **Feature Vectors:** During rasterization, feature vectors (colors/SH) are fetched from global memory. To reduce register pressure, these fetches are performed in half-precision (FP16) [S1].
*   **Volume Grid:** For volume-based variants, a 3D sparse tensor structure is used. Example dimensions: $128 \times 320 \times 640$ with voxel size $0.1\text{m}$, covering a $12.8\text{m} \times 32\text{m} \times 64\text{m}$ volume [S2].

### Numbered Steps
1.  **Initialization:** Initialize Gaussians from Structure-from-Motion (SfM) points. Calculate scene extent. If using the corrected method, compute $e'_{\text{scene}}$ based on SfM points rather than camera positions to avoid bias from clustered cameras [S3].
2.  **Forward Rasterization:**
    *   Project 3D Gaussians to 2D screen space.
    *   Sort Gaussians per tile/pixel by depth.
    *   Alpha-blend colors using the 3DGUT formulation [S1].
    *   Fetch feature vectors in FP16 to reduce register pressure [S1].
3.  **Loss Computation:**
    *   Compute color loss ($\mathcal{L}_{\text{color}}$) against reference images.
    *   Compute scale loss ($\mathcal{L}_{\text{scales}}$) if optimizing deformation or static Gaussians jointly [S4, unofficial].
    *   Total Loss: $\mathcal{L} = \mathcal{L}_{\text{color}} + \lambda_1 \cdot \mathcal{L}_{\text{scales}}$, where $\lambda_1 = 0.1$ [S4, unofficial].
    *   Apply an automatic scaling factor to loss values to improve half-precision training stability [S1].
4.  **Backward Pass:**
    *   Compute gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for all Gaussian parameters.
    *   Use custom CUDA kernels for forward and backward logic, implemented via the `gsplat` library [S1, S2].
5.  **Optimization Update:**
    *   Update parameters using the Adam optimizer with a learning rate of $1 \times 10^{-3}$ [S2].
    *   **Warm-up Phase:** Optimize static Gaussians for the first 3000 iterations [S4, unofficial].
    *   **Joint Optimization:** After warm-up, jointly optimize Gaussians and deformation fields (if applicable) [S4, unofficial].
    *   **Sampling:** Randomly select a single image from a random scene for supervision per iteration [S2].

### Formulas
**Corrected Scene Extent:**
To prevent bias from camera distribution (e.g., orbiting a small object), the scene extent $e'_{\text{scene}}$ is calculated using SfM points $p_i$ and average camera position $\bar{C}$:
$$ \bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i $$
$$ e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2 $$
Where $N_{\text{cam}}$ is the number of camera views, $N_{\text{SfM}}$ is the number of SfM points, and $C_i$ is the position of the $i$-th camera [S3].

**Total Loss:**
$$ \mathcal{L} = \mathcal{L}_{\text{color}} + \lambda_1 \cdot \mathcal{L}_{\text{scales}} $$
Where $\lambda_1 = 0.1$ [S4, unofficial].

### Parameters
*   **Optimizer:** Adam [S2].
*   **Learning Rate:** $1 \times 10^{-3}$ [S2].
*   **SH Degree:** 1 [S2].
*   **Loss Weight ($\lambda_1$):** 0.1 [S4, unofficial].
*   **Loss Weight ($\lambda_v$):** 0.2 (used in specific deformation contexts) [S4, unofficial].
*   **Volume Dimensions (Urban):** Height 12.8m, Width 32m, Length 64m; Voxel size 0.1m [S2].
*   **Waymo Zero-shot Range:** (50m, 20m, 128m) [S2].
*   **Warm-up Iterations:** 3000 [S4, unofficial].

### Edge Cases and Pitfalls
*   **Scene Extent Bias:** The default 3DGS scene extent calculation ($e_{\text{scene}} = 1.1 \cdot \max_{i} \|\bar{C} - C_i\|_2$) is proportional to the furthest camera distance. If cameras are clustered (e.g., orbiting a small object in a large environment), this results in an incorrectly small scene extent, causing inappropriate Gaussian splitting/pruning [S3]. Use the corrected SfM-based extent [S3].
*   **Half-Precision Stability:** Training in FP16 requires an automatic scaling factor for loss values to prevent gradient underflow/overflow [S1].
*   **Register Pressure:** Fetching feature vectors in full precision during rasterization increases register pressure, potentially reducing occupancy. Use FP16 fetches [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_3dgs(gaussians, cameras, images, config):
    """
    Main training loop for 3D Gaussian Splatting.
    """
    optimizer = Adam(gaussians.parameters(), lr=config.learning_rate)
    
    # Calculate scene extent using SfM points to avoid camera bias
    avg_camera_pos = mean([cam.position for cam in cameras])
    scene_extent = mean([norm(avg_camera_pos - p) for p in gaussians.initial_points])
    
    for iteration in range(config.max_iterations):
        # Randomly select a single image and camera for supervision
        idx = random.randint(0, len(images) - 1)
        image = images[idx]
        camera = cameras[idx]
        
        # Forward pass: Rasterize Gaussians
        # Uses gsplat with custom CUDA kernels
        # Feature vectors fetched in FP16 to reduce register pressure
        rendered_image = rasterize(gaussians, camera, scene_extent)
        
        # Compute Loss
        # L = L_color + lambda1 * L_scales
        loss_color = l1_loss(rendered_image, image)
        loss_scales = compute_scale_loss(gaussians)
        loss = loss_color + config.lambda1 * loss_scales
        
        # Apply scaling factor for FP16 stability
        loss = loss * config.loss_scale_factor
        
        # Backward pass
        optimizer.zero_grad()
        loss.backward()
        
        # Update parameters
        optimizer.step()
        
        # Densification/Pruning logic (e.g., gradient threshold scheduling)
        if iteration < 3000:
            # Warm-up: optimize static Gaussians only
            pass
        else:
            # Joint optimization of Gaussians and deformation field
            update_deformation_field(gaussians)
            
        # Adaptive density control (splitting/pruning)
        densify_gaussians(gaussians, iteration)
```

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
First-order Spherical Harmonics (SH) Distillation reduces the parameter count and computational cost of 3D Gaussian Splatting by distilling high-fidelity third-order SH coefficients from a pretrained teacher model into a compact first-order representation for a student model. Unlike standard degree reduction which simply truncates coefficients, this method uses knowledge distillation to preserve rendering quality despite the aggressive reduction from 16 coefficients per color channel to 4. It sets itself apart from sibling methods like [RoofGS](9830-roofgs.md) or [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md) by specifically targeting the appearance representation's dimensionality rather than the geometric or sorting pipeline.

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
The Continuous Attenuation Model reformulates the rasterization-based image formation of 3D Gaussian Splatting into a volumetric ray-marching framework by mapping Gaussian primitives to a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. This equivalence allows the application of theoretical results from Other-at-Volume (OaV) for surface extraction, ensuring that transmittance isosurfaces remain unbiased and ellipsoidal. It distinguishes itself from prior attenuation approximations like those in GGGS [57] by enforcing geometric constraints that prevent surface erosion and multi-view inconsistency.

### RoofGS
RoofGS is an end-to-end acceleration method for 3D Gaussian Splatting (3DGS) that optimizes the rendering pipeline by matching specific software techniques to the dominant hardware constraints of each stage [S1]. It distinguishes itself from sibling methods by applying a global Roofline analysis to guide a stage-specific optimization principle: bandwidth-limited stages prioritize reductions in global memory traffic, whereas compute-intensive stages prioritize lower instruction cost and increased instruction-level parallelism (ILP) [S1].

### [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)
Inverse Multiquadric Splatting is a radiance-field optimization technique that replaces the standard Gaussian radial basis function with an inverse multiquadric function to alter the decay profile of splats during rasterization. It differs from standard 3D Gaussian Splatting by using a slower-decaying tail, which requires a specific correction factor for the covariance matrix to maintain visual fidelity.

### [Tile-Based Comparison-Free Sorting](9991-tile-based-comparison-free-sorting.md)
Tile-Based Comparison-Free Sorting restricts depth sorting operations to individual rendering tiles to avoid global synchronization, reduce hardware complexity, and achieve fixed latency per output. Unlike full-scene sorting which causes excess traffic and latency spikes, this method uses $\langle \text{tile-id}, \text{depth} \rangle$ keys to order Gaussians front-to-back, matching the $\alpha$-blending requirement with deterministic, fixed per-output latency. It is specifically designed for hardware accelerators where variable latency and high memory overhead of radix sort are prohibitive.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
This method derives analytical Jacobians of the SE(3) camera pose with respect to 3D Gaussian primitives to enable efficient gradient-based camera tracking in SLAM systems. It solves the problem of high computational overhead associated with automatic differentiation during rasterization, ensuring minimal parameterization using Lie algebra. Unlike prior ray-intersection methods that assume fewer Gaussians, this approach is applicable to the EWA splatting and 3D Gaussian Splatting (3DGS) rendering pipeline.

### [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)
Opacity-Independent Gaussian Truncation is a modification to the fragment culling mechanism in 3D Gaussian Splatting that decouples the visibility threshold from the Gaussian's opacity parameter. It solves the problem where the original implementation prevents consistent truncation at fewer than ~3.03 standard deviations because low-opacity Gaussians are discarded before they can receive gradients, and enforces an upper bound on the transparency threshold due to opacity reset clipping. It sets itself apart from the default 3DGS truncation by evaluating the Gaussian spatial response independently of the opacity scalar, allowing for aggressive truncation (e.g., 1σ) and enabling gradient flow for zero-opacity Gaussians.

### Corrected Scene-Extent
A method to calculate scene extent based on the SfM point cloud rather than camera positions to avoid bias from camera distribution (e.g., orbiting a small object in a large environment) [S3]. The default 3DGS scene extent ($e_{\text{scene}}$) is proportional to the distance of the furthest camera from the average camera position, causing bias if cameras are clustered, leading to incorrect splitting/pruning [S3]. The corrected extent $e'_{\text{scene}}$ is calculated as $e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2$, where $N_{\text{SfM}}$ is the number of SfM points, $\bar{C}$ is the averaged camera position, and $p_i$ is the i-th SfM point [S3].

### Optimization Strategy
Optimize static Gaussians for the initial 3000 iterations as a warm-up phase, then jointly optimize both the Gaussians and the deformation field [S4, unofficial]. The total loss is $\mathcal{L} = \mathcal{L}_{\text{color}} + \lambda_1 \cdot \mathcal{L}_{\text{scales}}$, where $\lambda_1 = 0.1$ [S4, unofficial]. Another loss weight $\lambda_v = 0.2$ is used in specific contexts [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the rasterization and sorting steps. Tile-based sorting reduces global synchronization overhead [S1].
*   **Memory Complexity:** Depends on the number of Gaussians and the volume grid size. Half-precision fetches reduce register pressure and memory bandwidth usage [S1]. Sparse convolution implementations (e.g., `torchsparse`) help manage memory for large volumes [S2].
*   **Performance:** Using `gsplat` and custom CUDA kernels provides efficient forward and backward passes [S1, S2]. Half-precision training improves stability and speed when combined with loss scaling [S1].
*   **Strengths:** High fidelity, real-time rendering capability, efficient memory usage with sparse structures.
*   **Limitations:** Sensitive to scene extent initialization; requires careful hyperparameter tuning for half-precision stability.

## Background
3D Gaussian Splatting was introduced as a real-time radiance field representation method. The implementation details described here are based on various extensions and optimizations, including the `gsplat` library [S1, S2], corrected scene extent formulations [S3], and 4D Gaussian Splatting adaptations [S4, unofficial]. The use of `tiny-cuda-nn` for MLPs and `torchsparse` for 3D convolutions reflects the integration of neural components with primitive-based rendering [S1, S2].

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Alias-free 4D Gaussian Splatting, https://arxiv.org/abs/2511.18367v1, fetched 2026-10-26 (unofficial)
