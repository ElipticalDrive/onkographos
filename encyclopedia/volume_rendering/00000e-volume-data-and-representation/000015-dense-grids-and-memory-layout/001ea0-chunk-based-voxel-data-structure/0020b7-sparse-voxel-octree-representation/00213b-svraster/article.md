# SVRaster

Also called: Sparse Voxels Rasterization

## Overview
SVRaster is a real-time, high-fidelity novel-view synthesis framework that renders adaptive sparse voxels using a custom rasterizer, avoiding neural networks, 3D Gaussians, and sparse-point priors [S1, S2]. It distinguishes itself from Gaussian splatting by partitioning 3D space into disjoint voxels and using ray direction-dependent Morton ordering to ensure correct depth sorting, thereby eliminating the popping artifacts common in Gaussian-based methods [S1, S2]. Unlike previous neural-free voxel models, it achieves over 4 dB PSNR improvement and more than 10x FPS speedup while maintaining compatibility with grid-based 3D processing techniques like Volume Fusion and Marching Cubes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of multi-view images; camera parameters (intrinsics, extrinsics); maximum octree level $L$; number of integration samples $K$.
*   **Output:** Rendered color images for novel views.

**Data Structures:**
Voxels are stored in a 1D array containing only leaf nodes of an octree-style spatial partition [S1, S2]. No parent-child pointers or linear octree structures are maintained [S1]. Each voxel is defined by:
1.  **Index:** $v = \{i, j, k\} \in [0, \dots, 2^{L-1}]^3$ [S1].
2.  **Level:** $l \in [1, L]$ (where $l=0$ is the unused root) [S1].
3.  **Geometry Parameters:** $v_{geo} \in \mathbb{R}^{2\times2\times2}$, representing density values at the 8 corners of the voxel [S1]. Adjacent voxals share these corner values to ensure continuous density fields [S1, S2].
4.  **Appearance Parameters:** Spherical Harmonic (SH) coefficients encoding view-dependent color [S1, S2].

**Parameters:**
*   $L$: Maximum level of detail. Set to 16 in the source, defining a maximum representational grid resolution of $(2^{16})^3 = 65536^3$ [S1, S2]. This is a limit, not a dense allocation [S2].
*   $K$: Number of evenly spaced samples inside the ray-voxel intersection segment for volume integration [S1, S2].
*   Octree Size $\mathbf{w}_s \in \mathbb{R}^3$ and Center $\mathbf{w}_c \in \mathbb{R}^3$ in world space [S1].

**Algorithm Steps:**

1.  **Voxel Size and Center Calculation:**
    For a voxel with index $v$ and level $l$, compute its size $\mathbf{v}_s$ and center $\mathbf{v}_c$ [S1]:
    $$ \mathbf{v}_s = \mathbf{w}_s \cdot 2^{-l} $$
    $$ \mathbf{v}_c = \mathbf{w}_c - 0.5 \cdot \mathbf{w}_s + \mathbf{v}_s \cdot (2v + 1) $$
    *(Note: The source formula $\mathbf{v}_c = \mathbf{w}_c - 0.5 \cdot \mathbf{w}_s + \mathbf{v}_s \cdot \mathbf{v}_c$ in the notes appears to contain a typo where $\mathbf{v}_c$ is on both sides. The standard octree center calculation for index $v$ at level $l$ relative to size $\mathbf{v}_s$ is $\mathbf{w}_c - 0.5\mathbf{w}_s + \mathbf{v}_s(2v+1)$. The source text says "$\mathbf{v}_s \cdot \mathbf{v}_c$" but defines $\mathbf{v}_c$ as derived from index. We interpret $\mathbf{v}_c$ in the source formula as the normalized index vector $2v+1$ or similar. Given the ambiguity, we stick to the explicit variables defined: $\mathbf{v}_s$ is size, $\mathbf{v}_c$ is center. The source equation (2) is: $\mathbf{v}_s = \mathbf{w}_s \cdot \mathbf{v}_s^{-1}$ (likely $2^{-l}$) and $\mathbf{v}_c = \mathbf{w}_c - 0.5 \cdot \mathbf{w}_s + \mathbf{v}_s \cdot \mathbf{v}_c$ (likely $\mathbf{v}_c$ on RHS is the integer index vector). To be implementable, we define the normalized position. Let $u = 2v + 1$. Then $\mathbf{v}_c = \mathbf{w}_c - 0.5 \mathbf{w}_s + \mathbf{v}_s u$.)*
    
    *Correction based on strict source adherence:* The source states: $\mathbf{v}_s = \mathbf{w}_s \cdot \mathbf{v}_s^{-1}$ and $\mathbf{v}_c = \mathbf{w}_c - 0.5 \cdot \mathbf{w}_s + \mathbf{v}_s \cdot \mathbf{v}_c$. This is recursive. However, standard interpretation of such notation in this context (and the variable definition "derived from index $v$") implies $\mathbf{v}_c$ on the RHS is the normalized coordinate. We will use the standard octree center formula which matches the intent: $\mathbf{v}_c = \mathbf{w}_c - \frac{1}{2}\mathbf{w}_s + \mathbf{v}_s (2v + 1)$ [G].

2.  **Morton Code Mapping:**
    Map the voxel grid index $v$ to a Morton code using bit-interleaving operations [S1]. This is typically done in low-level CUDA for performance [S1].

3.  **Adaptive Allocation (Training Phase):**
    *   **Pruning:** Remove voxels whose maximum contribution across training views is below a threshold [S2].
    *   **Subdivision:** Subdivide voxels with high loss-related priority [S2].
    *   **Suppression:** Suppress subdivision if training views sample the voxel too sparsely [S2].

4.  **Rasterization Preprocessing:**
    *   Project voxel corners into the image space [S2].
    *   Assign each voxel to the image tiles it covers [S2].
    *   Preprocess density, color, and normal data for each voxel [S2].

5.  **Sorting:**
    *   Sort voxels per tile using **ray direction-dependent Morton ordering** [S1, S2].
    *   The Morton order is remapped based on the signs of the viewing ray's direction components. There are 8 such sign cases in 3D [S2].
    *   This ensures voxels are rendered in the correct depth order, preventing smaller voxels from being incorrectly occluded by larger ones [S2].

6.  **Density and Alpha Computation:**
    *   **Density Field:** Modeled as trilinearly varying inside the voxel, interpolated from the 8 shared corner values $v_{geo}$ [S1, S2].
    *   **Activation:** Apply exponential-linear activation to ensure non-negative density [S1]:
        $$ \text{exp\_linear}(x) = \begin{cases} x & \text{if } x > 1.1 \\ \exp\left(\frac{x}{1.1} - 1 + \ln 1.1\right) & \text{otherwise} \end{cases} $$
    *   **Alpha Value:** Evenly sample $K$ points inside the ray-voxel intersection segment [S1, S2]. Compute volume integration to determine the alpha contribution $\alpha_i$ of the voxel [S1].
    *   **Color:** Approximate the color field as constant within the voxel for efficiency, using the SH coefficients evaluated at the ray direction [S1, S2].

7.  **Compositing:**
    *   Composite voxels front-to-back using the standard alpha blending equation [S1, S2]:
        $$ C = \sum_{i=1}^N T_i \cdot \alpha_i \cdot c_i $$
        $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
        where $c_i \in [0,1]^3$ is the view-dependent color, $\alpha_i \in [0,1]$ is the alpha, and $T_i$ is the transmittance [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def exponential_linear(x):
    """
    Exponential-linear activation for non-negative density.
    """
    # x is a scalar or array
    if x > 1.1:
        return x
    else:
        return math.exp(x / 1.1 - 1 + math.log(1.1))

def trilinear_interpolate(corners, t):
    """
    Trilinearly interpolate density from 8 corners at parameter t (0 to 1).
    corners: array of shape (8,) representing density at corners.
    t: interpolation parameter along the ray segment inside the voxel.
    Note: The exact mapping of 't' to the 8 corners requires the ray direction
    and voxel geometry. This is a placeholder for the specific trilinear kernel.
    """
    # Implementation detail: Map t to (u,v,w) in [0,1]^3 based on ray direction
    # and voxel bounds. Then perform trilinear interpolation.
    # For the purpose of this article, we assume a helper exists or is implemented
    # based on the ray-voxel intersection segment.
    pass

def render_voxel_ray(ray_origin, ray_direction, voxel, K=16):
    """
    Compute alpha and color contribution of a single voxel for a given ray.
    """
    # 1. Compute intersection segment of ray and voxel
    # Returns t_near, t_far in world space
    t_near, t_far = compute_ray_voxel_intersection(ray_origin, ray_direction, voxel)
    
    if t_near is None:
        return 0.0, np.array([0.0, 0.0, 0.0])

    # 2. Sample K points inside the segment
    # Evenly sample K points
    t_samples = np.linspace(t_near, t_far, K)
    
    # 3. Compute density and color at samples
    # Density is trilinearly interpolated from 8 corners
    # Color is constant (SH evaluated at ray direction)
    
    # Get corner densities
    corner_densities = voxel.corners # Shape (8,)
    
    # Get SH coefficients for color
    sh_coeffs = voxel.sh_coeffs
    color = evaluate_sh(sh_coeffs, ray_direction)
    
    alpha_sum = 0.0
    density_sum = 0.0
    
    for t in t_samples:
        # Interpolate density at t
        # Note: t must be normalized to the voxel's local coordinate system
        # for trilinear interpolation.
        local_t = normalize_to_voxel(t, voxel)
        density = trilinear_interpolate(corner_densities, local_t)
        
        # Apply activation
        density = exponential_linear(density)
        
        # Accumulate for volume integration (simplified Riemann sum)
        # Delta t in world space
        dt = (t_far - t_near) / K
        alpha_sum += density * dt
        
    # Convert accumulated density to alpha (approximation)
    # Standard volume rendering alpha = 1 - exp(-density * thickness)
    # Or simply alpha = min(1.0, alpha_sum) depending on specific implementation
    # The source says "compute volume integration for its alpha value contribution"
    # We assume the result is mapped to [0,1]
    alpha = min(1.0, alpha_sum)
    
    return alpha, color

def render_tile(tile_voxels, ray_direction, K=16):
    """
    Render a tile by sorting and compositing voxels.
    """
    # 1. Sort voxels using ray direction-dependent Morton ordering
    # Morton code depends on voxel index
    # Order depends on signs of ray_direction components
    sorted_voxels = sort_by_morton_order(tile_voxels, ray_direction)
    
    # 2. Composite front-to-back
    transmittance = 1.0
    color = np.array([0.0, 0.0, 0.0])
    
    for voxel in sorted_voxels:
        alpha, c = render_voxel_ray(ray_origin, ray_direction, voxel, K)
        
        # Alpha composition
        contribution = transmittance * alpha * c
        color += contribution
        transmittance *= (1.0 - alpha)
        
        if transmittance < 1e-4:
            break
            
    return color

def compute_morton_order_sign_dependent(voxel_morton_code, ray_direction):
    """
    Remap Morton order based on ray direction signs.
    There are 8 sign cases.
    """
    sx = 1 if ray_direction.x >= 0 else -1
    sy = 1 if ray_direction.y >= 0 else -1
    sz = 1 if ray_direction.z >= 0 else -1
    
    # The specific remapping logic is described as "remaps Morton order... 
    # according to the signs". The exact permutation table is not provided 
    # in the text, but it ensures correct depth sorting.
    # [G] Implementation requires the specific permutation table from the paper's 
    # supplementary materials or code.
    return voxel_morton_code # Placeholder for the remapped key
```

## Subtopics

### Sparse Voxel Octree with Bricks
Uses brick-based compression for memory efficiency. SVRaster uses a flat 1D array of leaves, trading compression for simpler rasterization logic.

### Naive Octree Node Representation
Stores full octree nodes. SVRaster stores only leaves, reducing overhead.

### Dynamic Update Optimization
Focuses on updating the structure during training. SVRaster's adaptive allocation is a specific instance of this, focusing on pruning and subdivision based on loss.

## Operating characteristics
*   **Time Complexity:** Rendering is rasterization-based, allowing for real-time frame rates. The paper reports 121 FPS on a 3090 Ti for Mip-NeRF360 scenes, comparable to 3DGS (131 FPS) [S2].
*   **Memory Complexity:** Uses more GPU memory and produces larger models than 3DGS [S2]. The maximum representational resolution is $65536^3$, but actual memory usage depends on the number of active leaf voxels [S2].
*   **Performance:** On Mip-NeRF360, SVRaster achieves LPIPS 0.219 vs 0.257 for 3DGS, and PSNR 27.33 vs 27.45 dB [S2]. On Tanks&Temples, it shows brightness boundaries and floaters due to exposure variation, with worse PSNR and FPS than 3DGS, though LPIPS is better [S2].
*   **Strengths:** Avoids popping artifacts; compatible with grid-based 3D processing (Volume Fusion, Marching Cubes); neural-free; high fidelity.
*   **Limitations:** Sensitive to exposure variation in training data; higher memory usage than 3DGS; requires custom rasterizer.

## Background
SVRaster was proposed by NVIDIA Research (NVlabs) and presented at CVPR 2025 [S1]. It builds on the concept of sparse voxel octrees and Plenoxels, aiming to combine the efficiency of rasterization with the explicit nature of voxel representations, avoiding the limitations of Gaussian splatting (popping) and neural fields (training complexity) [S1]. The code is available at [github.com/NVlabs/svraster](https://github.com/NVlabs/svraster) [S1].

## Sources
[S1] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
[S2] Sparse Voxels Rasterization: Real-time High-fidelity ..., https://www.alphaxiv.org/abs/2412.04459, fetched 2026-10-29
