# X-Gaussian

Also called: Radiative Gaussian Splatting for X-Ray

## Overview
X-Gaussian is a 3D Gaussian Splatting (3DGS) framework adapted for X-ray imaging, designed for efficient X-ray novel view synthesis (NVS) and sparse-view CT reconstruction [S1]. It replaces the standard RGB radiance field with a radiative model that excludes view-direction dependence, rendering radiation intensity directly [S1]. Unlike its sibling [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md) which may retain angular dependencies or general radiative properties, X-Gaussian specifically models the projection nature of X-rays, enabling it to generate high-quality projections for downstream learning-free CT reconstruction algorithms like FDK, SART, and ASD-POCS [S1]. It achieves superior performance over NeRF-based methods (e.g., NAF) with significantly faster inference and training times [S1].

## How it works
X-Gaussian optimizes a 3D Gaussian point cloud to represent X-ray attenuation/radiation intensity. The method differs from standard 3DGS by removing view-directional color dependence and using a Differentiable Radiative Rasterization (DRR) kernel [S1].

### Inputs and Outputs
*   **Inputs**: A set of $N$ X-ray projection images with known camera intrinsics ($K$) and extrinsics ($c2w$). The images represent radiation intensity (or log-transformed intensity, depending on preprocessing, though the source implies direct intensity fitting in the radiative model context [S1]).
*   **Outputs**: A set of 3D Gaussians $\{G_i\}$ representing the scene's radiative properties, and rendered novel-view X-ray projections [S1].

### Data Structures
The scene is represented by a set of $M$ 3D Gaussians. Each Gaussian $G_i$ is defined by:
1.  **Mean** $\boldsymbol{\mu}_i \in \mathbb{R}^3$: The 3D position.
2.  **Covariance** $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Controls the shape and orientation. Parameterized as $\boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$, where $\mathbf{R}_i$ is a rotation matrix and $\mathbf{S}_i$ is a scaling matrix [S1].
3.  **Radiation Intensity** $I_i \in \mathbb{R}$: The scalar radiation value associated with the Gaussian. Unlike RGB 3DGS which stores spherical harmonics coefficients for color, X-Gaussian stores a single scalar intensity value because the model excludes view-direction dependence [S1].

### Initialization (ACUI)
X-Gaussian uses an **ACUI** (Automatic Camera and Uniform Initialization) strategy, avoiding Structure-from-Motion (SfM) [S1]:
1.  Compute intrinsic matrix $K$ and extrinsic matrix $c2w$ from the X-ray scanner parameters [S1].
2.  Define a cuboid bounding box enclosing the scanned object.
3.  Uniformly sample center points $\boldsymbol{\mu}_i$ for the 3D Gaussians within this cuboid [S1].

### Rendering (Differentiable Radiative Rasterization)
The rendering process computes the radiation intensity $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ passing through pixel $(u,v)$:

1.  **Sorting**: Gaussians are sorted by depth along the ray $\mathbf{r}$ [S1].
2.  **Projection**: For each Gaussian $G_i$, compute its 2D covariance in screen space $\boldsymbol{\Sigma}'_i$ using the Jacobian of the projection matrix [S1].
3.  **Alpha Calculation**: Compute the alpha value $\alpha_i$ for the Gaussian at the ray intersection:
    $$
    \alpha_i = \exp\left(-\frac{1}{2} \mathbf{d}_i^T \boldsymbol{\Sigma}'^{-1}_i \mathbf{d}_i\right)
    $$
    where $\mathbf{d}_i$ is the 2D offset from the Gaussian center to the ray origin in screen space [S1].
4.  **Blending**: The final radiation intensity is computed via alpha compositing:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i \in \mathcal{O}} T_i \alpha_i I_i, \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\mathcal{O}$ is the set of sorted Gaussians, $I_i$ is the radiation intensity of Gaussian $i$, and $T_i$ is the transmittance [S1].

### Optimization
The model is optimized by minimizing the photometric loss $\mathcal{L}$ between the rendered radiation intensity $\hat{C}(\mathbf{r})$ and the ground-truth X-ray projection $C(\mathbf{r})$:

$$
\mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \left\| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \right\|_2^2
$$

Gradients are backpropagated to update $\boldsymbol{\mu}_i$, $\boldsymbol{\Sigma}_i$, and $I_i$ using an optimizer (e.g., Adam) [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# X-Gaussian Rendering and Optimization

def initialize_gaussians(scanner_params):
    """
    Initialize 3D Gaussians using ACUI strategy.
    """
    K = compute_intrinsics(scanner_params)
    c2w = compute_extrinsics(scanner_params)
    
    # Define bounding cuboid enclosing the object
    cuboid = get_scanned_object_bounds(scanner_params)
    
    # Uniformly sample centers
    num_gaussians = 100000 # Not given in sources, placeholder for scale
    centers = uniform_sample(cuboid, num_gaussians)
    
    # Initialize other parameters
    # Covariance initialized to isotropic or based on initial density
    covariances = initialize_covariances(num_gaussians)
    # Radiation intensity initialized to mean intensity of input images
    intensities = torch.full((num_gaussians,), torch.mean(input_images))
    
    return centers, covariances, intensities

def render_radiative_projection(centers, covariances, intensities, K, c2w, ray_indices):
    """
    Differentiable Radiative Rasterization (DRR) CUDA kernel equivalent.
    Renders X-ray projection for a batch of rays.
    """
    # 1. Project 3D centers to 2D screen space
    # P = K @ c2w
    # x_2d = project(P, centers)
    
    # 2. Sort Gaussians by depth for each ray
    # sorted_indices = sort_by_depth(centers, c2w, ray_origins)
    
    # 3. Compute 2D covariance in screen space
    # J = Jacobian of projection
    # Sigma_2d = J @ Sigma_3d @ J.T
    
    # 4. Compute Alpha and Transmittance
    # For each ray, iterate through sorted Gaussians
    # alpha = exp(-0.5 * (x - mu)^T Sigma_2d^-1 (x - mu))
    # T = product(1 - alpha_prev)
    
    # 5. Composite Radiation Intensity
    # C = sum(T * alpha * I)
    
    # Note: The actual implementation is in CUDA C++ [S1]
    # This pseudocode represents the mathematical logic
    rendered_projections = compute_alpha_compositing(centers, covariances, intensities, K, c2w)
    return rendered_projections

def train_x_gaussian(input_images, scanner_params, num_iterations=20000):
    """
    Training loop for X-Gaussian.
    """
    centers, covariances, intensities = initialize_gaussians(scanner_params)
    
    for step in range(num_iterations):
        # 1. Ray Sampling
        ray_indices = sample_random_pixels(input_images)
        
        # 2. Forward Pass
        rendered = render_radiative_projection(
            centers, covariances, intensities, 
            scanner_params.K, scanner_params.c2w, ray_indices
        )
        
        # 3. Loss Computation
        # Ground truth images are X-ray projections
        ground_truth = get_pixel_values(input_images, ray_indices)
        loss = mse_loss(rendered, ground_truth)
        
        # 4. Backpropagation
        loss.backward()
        
        # 5. Update
        optimizer.step()
        optimizer.zero_grad()
        
        # 6. Scheduling
        # Adjust learning rates if necessary [S1]
        
    return centers, covariances, intensities
```

### Parameters and Typical Values
*   **Training Iterations**: Convergence is analyzed at 1000, 20000 iterations, and training times of 20s, 60s, 180s [S1].
*   **Optimizer**: Adam [S1].
*   **Learning Rate**: Not given in the sources.

### Edge Cases and Pitfalls
*   **Implementation Complexity**: The method requires a custom CUDA (C++) kernel for Differentiable Radiative Rasterization, making it harder to debug and less interpretable than PyTorch (Python) implementations [S1].
*   **View Direction Independence**: The model explicitly excludes view-direction dependence. If the data contains significant angular artifacts not modeled by simple projection, this assumption may limit accuracy [S1].
*   **Initialization**: Relies on accurate scanner parameters for ACUI. Incorrect intrinsic/extrinsic matrices will lead to misaligned Gaussians [S1].

## Subtopics

### Radiative Gaussian Splatting
A sibling method that may retain more general radiative properties or angular dependencies. Choose X-Gaussian for X-ray specific NVS and sparse-view CT reconstruction where view-independence is a valid assumption and speed is critical. [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md)

### Volumetric Primitives Radiance Field
A sibling method using volumetric primitives. X-Gaussian is faster and more efficient for sparse-view scenarios. [Volumetric Primitives Radiance Field](5900-volumetric-primitives-radiance-field.md)

## Operating characteristics
*   **Time Complexity**: Inference is significantly faster than NeRF-based methods. X-Gaussian is 73× faster in inference than NAF and 192× faster than TensoRF [S1]. Training time is less than 15% of that required by NAF [S1].
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**:
    *   Outperforms SOTA NeRF-based methods by over 6.5 dB in PSNR [S1].
    *   When used to generate novel views for sparse-view CT reconstruction (FDK, SART, ASD-POCS), X-Gaussian yields PSNR improvements of 15.19, 13.01, and 13.53 dB respectively, which are 1.32, 2.41, and 2.65 dB higher than using NAF [S1].
    *   Converges faster: Basic shape is formed by the 1000th iteration, whereas original 3DGS converges slowly and suffers from noisy point clouds [S1].
*   **Scaling**: The mean covariance of 3D Gaussians decreases as the number of training views increases, indicating a coarse-to-fine adaptation capable of representing fine-grained structures like small tumors [S1].
*   **Strengths**: Extreme efficiency (speed and training time), clear structural details (ribs, blood vessels) even in early training stages, cleaner backgrounds compared to NAF [S1].
*   **Limitations**: Higher implementation complexity due to CUDA dependency; less interpretable than Python-based methods [S1]. Requires foundational knowledge in computer graphics and 3D vision [S1].

## Background
X-Gaussian was proposed as the first 3DGS-based framework for X-ray novel view synthesis [S1]. It addresses the limitations of NeRF-based methods (like NAF) in terms of training time and inference speed, while providing better structural details for sparse-view CT reconstruction [S1]. The method builds on the principles of 3D Gaussian Splatting but adapts the radiative model to exclude view-direction dependence, suitable for X-ray imaging physics [S1].

## Sources
[S1] Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis, https://doi.org/10.1007/978-3-031-73232-4_16, fetched 2026-10-26
