# VoxNeRF

Also called: VoxNeRF: Bridging Voxel Representation and Neural Radiance Fields for Efficient Neural Rendering

## Overview
VoxNeRF is a neural rendering method that integrates explicit scene geometry priors (such as point clouds or depth maps) with implicit radiance field representation to improve sampling efficiency and reconstruction accuracy [S1]. It addresses the ill-posed nature of recovering 3D geometry from 2D images by using a Sparse Voxel Octree (SVO) to model geometric uncertainty, which guides a hybrid sampling strategy [S1]. Unlike standard NeRF methods that sample uniformly or via stratified noise, VoxNeRF concentrates samples near the estimated surface while maintaining coverage of free space, reducing computational waste in empty regions [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of input images with known camera poses.
*   A scene geometry prior, which can be a mesh, point cloud, or depth map [S1].
*   Pseudo-depth labels $D$ derived from the geometry prior for regularization [S1].

**Outputs:**
*   Rendered RGB images for novel viewpoints.
*   Volume accumulated depth $\hat{D}$ [S1].

### Data Structures

**Sparse Voxel Octree (SVO)**
*   Represents the scene geometry prior.
*   Each voxel is an axis-aligned bounding box with side length $v$ [S1].
*   Voxel vertex coordinates are encoded using 8-bit Morton codes to map 3D coordinates to 1D representations, preserving spatial locality and reducing memory consumption [S1].
*   Used to determine ray-surface intersections [S1].

**Multi-resolution Hash Grids (MHG)**
*   Represents the radiance field.
*   Consists of several compact spatial hash tables indexed at different resolutions [S1].
*   Stores trainable feature vectors [S1].
*   Used to encode spatial features for the neural network [S1].

**Neural Network**
*   **Feature MLP:** Takes concatenated tri-linear interpolated outputs from MHG tables and view direction encoding $d_i$ as input. Outputs density $\sigma_i$ and a hidden code [S1].
*   **Color MLP:** A 3-layer MLP that takes the hidden code concatenated with view direction encoding $d_i$ as input. Outputs radiance $c_i$ [S1].

### Sampling Strategy
The sampling strategy is divided into an "important region" near the surface and a "remaining region" for the rest of the ray [S1].

1.  **Ray-Surface Intersection:** Find the first non-empty leaf voxel hit by the ray in the SVO. Let $z(p_i)$ be the hitting point of the ray and the voxel [S1].
2.  **Important Region:** Model the sampling region as a Gaussian distribution $\mathcal{N}(z(p_i), (\sqrt{3}v_i)^2)$, where $v_i$ is the voxel size [S1].
    *   The region length is $6\sqrt{3}v_i$, centered at $z(p_i)$ [S1].
    *   This captures 99.7% of the distribution, assuming the true surface lies within the cube defined by the voxel uncertainty [S1].
    *   Sample points uniformly within this region [S1].
3.  **Remaining Region:** Generate additional sampling points using uniform distributions for the rest of the ray:
    *   $U(0, z(p_i) - \sqrt{3}3v_i)$
    *   $U(z(p_i) + \sqrt{3}3v_i, \infty)$ [S1].

### Rendering and Loss

**Volume Rendering**
For each sample point $i$ with density $\sigma_i$ and emitted radiance $c_i$, and distance to the next adjacent sample $\delta_i$:
*   Transmittance $T_i$ and termination probability $\alpha_i$ are computed as:
    $$ \alpha_i = 1 - \exp(-\delta_i \sigma_i) $$
    *(Note: The source text contains a typo in Eq. 2 as `1 - exp(1 - δi σi)`, but standard volume rendering and the context of transmittance imply the standard form. However, strictly following the source text provided: $\alpha_i = 1 - \exp(1 - \delta_i \sigma_i)$ is cited. Given the standard definition of transmittance $T_i = \exp(-\sum \delta_j \sigma_j)$, the probability of termination is $\alpha_i = T_i (1 - \exp(-\delta_i \sigma_i))$. The source Eq. 2 is likely a typo for $\alpha_i = 1 - \exp(-\delta_i \sigma_i)$ or similar. We will use the standard interpretation for implementation correctness, noting the source discrepancy.)*
    *Correction based on standard VR and source context:* The source Eq. 2 is explicitly written as $\alpha_i = 1 - \exp(1 - \delta_i \sigma_i)$. This is mathematically suspect for a probability. Standard NeRF uses $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$. We assume the source meant $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [S1].

**Loss Function**
The total loss $L$ is defined as:
$$ L = L_c + \lambda_d (L_d + L_{reg}) $$
where $L_c$ is the color reconstruction loss (e.g., MSE between rendered and ground truth colors) [S1].

**Robust Depth Loss ($L_d$)**
Let $\hat{D}$ be the volume accumulated depth and $D$ be the pseudo depth. Let $\beta = 0.1$ [S1].
$$
L_d =
\begin{cases}
\frac{1}{2} |\hat{D} - D|^2 & \text{if } |\hat{D} - D| < \beta \\
\frac{1}{2} |\hat{D} - D| + \ln\left(\frac{|\hat{D} - D|}{\beta}\right) & \text{if } |\hat{D} - D| \ge \beta
\end{cases}
$$
This loss uses a squared error for small differences and a logarithmic function for large differences to reduce sensitivity to outliers [S1].

**Regularization Loss ($L_{reg}$)**
Penalizes gradients across different scales to enforce smoothness:
$$ L_{reg} = \sum_{s=1}^{S} \sum_{i,j} M_{i,j} (|G_x^{(s)}| + |G_y^{(s)}|) $$
where $G_x^{(s)}$ and $G_y^{(s)}$ are gradients at scale $s$, and $M_{i,j}$ is a mask [S1].

### Parameters
*   **Voxel Size ($v$):** Set to the maximal standard deviation $\sigma$ across all scene points in the geometry prior: $v = \max(\sigma_1, \sigma_2, \dots, \sigma_n)$ [S1].
*   **Batch Size:** 4,096 rays per batch [S1].
*   **Points per Ray:** 128 points in the important sampling region and 128 points in the remaining region [S1].
*   **Optimizer:** ADAM with learning rate 0.01 and exponential schedule [S1].
*   **Beta ($\beta$):** 0.1 [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def voxnerf_render(ray_origins, ray_directions, svos, mhgs, mlp_feature, mlp_color, params):
    """
    Renders a batch of rays using VoxNeRF.
    
    Args:
        ray_origins: (B, 3) Array of ray origins.
        ray_directions: (B, 3) Array of ray directions.
        svos: List of SVO structures for each scene (or shared).
        mhgs: Multi-resolution Hash Grids structure.
        mlp_feature: Feature MLP function.
        mlp_color: Color MLP function.
        params: Dictionary containing hyperparameters.
        
    Returns:
        colors: (B, 3) Rendered colors.
        depths: (B,) Accumulated depths.
    """
    B = ray_origins.shape[0]
    colors = np.zeros((B, 3))
    depths = np.zeros(B)
    
    # 1. Determine Sampling Points
    # Find intersection with SVO to get z(pi) and voxel size vi
    # This step requires SVO traversal logic not detailed in pseudocode but implied
    # For each ray, find the first non-empty leaf voxel.
    # Let z_pi be the intersection point.
    # Let v_i be the voxel size.
    
    # Important region: Gaussian centered at z_pi with std sqrt(3)*v_i
    # Region length: 6 * sqrt(3) * v_i
    # Sample 128 points uniformly in [z_pi - sqrt(3)*3*v_i, z_pi + sqrt(3)*3*v_i]
    # Note: Source says length 6*sqrt(3)*vi, which is +/- 3*sqrt(3)*vi from center.
    
    important_points = []
    remaining_points = []
    
    for i in range(B):
        # Placeholder for SVO intersection logic [G]
        # z_pi, v_i = find_svo_intersection(ray_origins[i], ray_directions[i], svos)
        
        # Generate 128 points in important region
        # Lower bound: z_pi - sqrt(3) * 3 * v_i
        # Upper bound: z_pi + sqrt(3) * 3 * v_i
        # Using numpy linspace for uniform sampling
        lower = z_pi[i] - np.sqrt(3) * 3 * v_i[i]
        upper = z_pi[i] + np.sqrt(3) * 3 * v_i[i]
        pts_imp = np.linspace(lower, upper, 128)
        important_points.append(pts_imp)
        
        # Generate 128 points in remaining region
        # U(0, z_pi - sqrt(3)*3*v_i) and U(z_pi + sqrt(3)*3*v_i, inf)
        # Since infinity is not practical, we assume a max ray length or split the 128 points
        # Source says "U(0, ...) and U(..., inf)". We split 128 points between these two intervals.
        # Let's assume 64 points for near, 64 for far.
        
        near_lower = 0.0
        near_upper = z_pi[i] - np.sqrt(3) * 3 * v_i[i]
        pts_near = np.linspace(near_lower, near_upper, 64) if near_upper > near_lower else np.array([])
        
        far_lower = z_pi[i] + np.sqrt(3) * 3 * v_i[i]
        # Max ray length is typically defined by scene bounds. Not given in sources. [G]
        # Assume a large constant MAX_DIST for implementation.
        MAX_DIST = 100.0 
        pts_far = np.linspace(far_lower, MAX_DIST, 64) if MAX_DIST > far_lower else np.array([])
        
        remaining_points.append(np.concatenate([pts_near, pts_far]))

    # 2. Query Network and Render
    for i in range(B):
        all_points = np.concatenate([important_points[i], remaining_points[i]])
        # Convert 1D depth to 3D positions
        points_3d = ray_origins[i] + all_points[:, None] * ray_directions[i]
        
        # Query MHG and MLPs
        # mhg_output = query_mhgs(mhgs, points_3d)
        # hidden_code, sigma = mlp_feature(mhg_output, view_dirs)
        # radiance = mlp_color(concat(hidden_code, view_dirs))
        
        # Volume Rendering
        # T_i = exp(-sum(sigma * delta))
        # alpha_i = 1 - exp(-sigma * delta)
        # Color = sum(alpha_i * T_i * c_i)
        # Depth = sum(alpha_i * T_i * z_i)
        
        # Standard VR loop [G]
        transmittance = 1.0
        accumulated_color = np.zeros(3)
        accumulated_depth = 0.0
        
        for j in range(len(all_points)):
            # delta is distance to next sample
            if j < len(all_points) - 1:
                delta = all_points[j+1] - all_points[j]
            else:
                delta = 0.0 # Or small epsilon
            
            # Query network for sigma and color at points_3d[j]
            # sigma_j, c_j = query_network(points_3d[j])
            
            alpha_j = 1.0 - np.exp(-sigma_j * delta)
            transmittance *= (1.0 - alpha_j)
            
            accumulated_color += alpha_j * transmittance * c_j
            accumulated_depth += alpha_j * transmittance * all_points[j]
            
        colors[i] = accumulated_color
        depths[i] = accumulated_depth

    return colors, depths

def compute_loss(rendered_colors, gt_colors, rendered_depths, gt_depths, params):
    """
    Computes the total loss for VoxNeRF.
    """
    # Color Loss Lc (MSE)
    Lc = np.mean((rendered_colors - gt_colors) ** 2)
    
    # Robust Depth Loss Ld
    beta = 0.1
    diff = np.abs(rendered_depths - gt_depths)
    
    Ld = np.where(diff < beta, 
                  0.5 * diff ** 2, 
                  0.5 * diff + np.log(diff / beta))
    Ld = np.mean(Ld)
    
    # Regularization Loss Lreg (Placeholder for gradient computation)
    # Lreg = sum of gradients across scales [S1]
    Lreg = 0.0 # Computed via image gradients [G]
    
    lambda_d = params.get('lambda_d', 1.0)
    L = Lc + lambda_d * (Ld + Lreg)
    
    return L
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of sampling points (256 per ray in the reported setup) and the cost of querying the Multi-resolution Hash Grids and MLPs. The SVO query is efficient due to Morton code indexing [S1].
*   **Memory Complexity:** Dominated by the MHG tables and the SVO structure. The SVO uses 8-bit Morton codes for compactness [S1]. The MHG stores trainable feature vectors in compact hash tables [S1].
*   **Performance:** Evaluated on ScanNet and ScanNet++ datasets. Trained for 30,000 iterations on a single 24GB NVIDIA RTX 4090 GPU [S1].
*   **Strengths:** Reduces unnecessary sampling in free space by focusing on the geometry prior. Handles noise in geometry priors via the Gaussian sampling model.
*   **Limitations:** Relies on the quality of the geometry prior. If the prior is inaccurate, the sampling region may miss the true surface or include too much empty space.
*   **When to use:** When a coarse geometry prior (point cloud, depth map) is available and efficient rendering is desired. Particularly useful in sparse-view settings [S1].

## Background
VoxNeRF was introduced to bridge the gap between voxel-based representations and neural radiance fields. It addresses the limitations of standard NeRF methods which often suffer from slow convergence and poor performance in textureless or ambiguous regions. By incorporating explicit geometry priors, it aims to provide a more robust and efficient rendering pipeline. The method was evaluated on real-world indoor datasets like ScanNet and ScanNet++ [S1].

## Sources
[S1] VoxNeRF: Bridging Voxel Representation and Neural Radiance Fields for ..., https://arxiv.org/abs/2311.05289, fetched 2026-10-29
