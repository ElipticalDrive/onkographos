# FastNeRF

Also called: Factorized NeRF, Factorized Neural Radiance Fields

## Overview
FastNeRF is a neural rendering method that accelerates the inference of Neural Radiance Fields (NeRF) by factorizing the view-dependent color function into position-dependent and direction-dependent components, enabling efficient caching of intermediate radiance values [S6, unofficial]. It solves the problem of high computational cost in standard NeRF, which requires hundreds of MLP evaluations per pixel, by reducing inference to memory-bound cache lookups and dot products, achieving speeds over 3000 times faster than the original NeRF [S1][S6, unofficial]. It differs from its siblings, such as [C-NeRF](7275-c-nerf.md) and [VoxGRAF](12633-voxgraf.md), by explicitly separating the 5D radiance field into two independent low-dimensional manifolds (position and direction) to allow for compact, sparse caching of a "deep radiance map" rather than caching the full 5D function or using smaller local MLPs [S6, unofficial][S5].

## How it works
FastNeRF builds on [Voxel-based Representation](10852-voxel-based-representation.md) by replacing the single monolithic MLP with two factorized networks and caching their outputs [S6, unofficial].

### Inputs and Outputs
*   **Inputs:** A set of training images with known camera poses; a bounding box $V$ enclosing the scene.
*   **Outputs:** A trained position-dependent network $\mathcal{F}_{\text{pos}}$, a direction-dependent network $\mathcal{F}_{\text{dir}}$, and optionally, cached grids for inference [S6, unofficial].

### Data Structures
1.  **Position Cache:** A sparse 3D grid storing outputs of $\mathcal{F}_{\text{pos}}$ [S6, unofficial].
    *   **Fields:** Density $\sigma$ (scalar) and a deep radiance map components $(\mathbf{u}, \mathbf{v}, \mathbf{w})$ (each $D$-dimensional vectors) [S6, unofficial].
    *   **Layout:** Stored as dense 16-bit floating point values in a grid of size $k^3$ (or sparse subset thereof) [S6, unofficial].
2.  **Direction Cache:** A sparse 2D grid storing outputs of $\mathcal{F}_{\text{dir}}$ [S6, unofficial].
    *   **Fields:** Weight vector $\boldsymbol{\beta}$ ($D$-dimensional) [S6, unofficial].
    *   **Layout:** Stored as dense 16-bit floating point values in a grid of size $l^2$ (parameterized by spherical coordinates $\theta, \phi$) [S6, unofficial].
3.  **Collision Mesh:** A triangle mesh derived from the density volume $\sigma$ using Marching Cubes, used for hardware-accelerated ray tracing to skip empty space [S6, unofficial].

### Network Architecture
*   **$\mathcal{F}_{\text{pos}}$:** An 8-layer MLP with 384 hidden units (256 for LLFF scenes). Input: 3D position $\mathbf{p}$. Output: $\{\sigma, (\mathbf{u}, \mathbf{v}, \mathbf{w})\}$ [S6, unofficial].
*   **$\mathcal{F}_{\text{dir}}$:** A 4-layer MLP with 128 hidden units. Input: 2D viewing direction $\mathbf{d}$ (parameterized as a unit vector or spherical coordinates). Output: $\boldsymbol{\beta}$ [S6, unofficial].
*   **Positional Encoding:** Applied to inputs of both MLPs, as in standard NeRF [S6, unofficial].

### Training Steps
1.  Define bounding box $V$ for the scene [S6, unofficial].
2.  Sample $k$ uniform values for $x, y, z$ within $V$ and $l$ uniform values for $\theta \in [0, \pi], \phi \in [0, 2\pi]$ [S6, unofficial].
3.  Compute $\mathcal{F}_{\text{pos}}$ for all position combinations and $\mathcal{F}_{\text{dir}}$ for all direction combinations to initialize caches (optional, primarily for initialization or analysis; training typically optimizes the MLPs directly on rendered images) [S6, unofficial].
4.  Train $\mathcal{F}_{\text{pos}}$ and $\mathcal{F}_{\text{dir}}$ jointly using the standard NeRF loss (MSE between rendered and ground truth colors) [S6, unofficial].

### Inference Steps
1.  **Ray Generation:** For each pixel, generate a ray $\mathbf{r}$ defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [G].
2.  **Empty Space Skipping:** Intersect $\mathbf{r}$ with the collision mesh. Start integration only after the first hit [S6, unofficial].
3.  **Sampling:** Uniformly sample $N$ points $\mathbf{p}_i$ along the ray within the mesh bounds [S6, unofficial].
4.  **Cache Lookup:**
    *   For each $\mathbf{p}_i$, lookup $\sigma_i$ and $(\mathbf{u}_i, \mathbf{v}_i, \mathbf{w}_i)$ from the Position Cache using nearest-neighbor interpolation [S6, unofficial].
    *   For the ray direction $\mathbf{d}$, lookup $\boldsymbol{\beta}$ from the Direction Cache using trilinear sampling (on the $\theta, \phi$ grid) [S6, unofficial].
5.  **Color Computation:** Compute color $\mathbf{c}_i$ for each sample point:
    $$
    \mathbf{c}_i = \boldsymbol{\beta}^T \cdot (\mathbf{u}_i, \mathbf{v}_i, \mathbf{w}_i) = \sum_{j=1}^D \beta_j (u_{i,j}, v_{i,j}, w_{i,j})
    $$
    where $D$ is the number of components (typically 8 for Synthetic 360, 6 for LLFF) [S6, unofficial].
6.  **Volume Rendering:** Integrate color and density along the ray using the standard NeRF volume rendering equation [G]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ and $\delta_i$ is the distance between samples [G].
7.  **Termination:** Terminate ray integration once transmittance $T_i$ drops below a threshold (saturation) [S6, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def ray_aabb_intersect(origin, direction, bounds):
    """Returns (t_min, t_max) for ray-AABB intersection. bounds = [min_x, max_x, min_y, max_y, min_z, max_z]."""
    t_min = -math.inf
    t_max = math.inf
    for i in range(3):
        if direction[i] == 0:
            if origin[i] < bounds[2*i] or origin[i] > bounds[2*i+1]:
                return None, None
        else:
            t1 = (bounds[2*i] - origin[i]) / direction[i]
            t2 = (bounds[2*i+1] - origin[i]) / direction[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
    if t_max < 0 or t_min > t_max:
        return None, None
    return t_min, t_max

def fast_nerf_infer(ray_origin, ray_direction, pos_cache, dir_cache, D, bounds, step_size):
    """
    Renders a single pixel using FastNeRF caches.
    
    Args:
        ray_origin: (3,) array, camera origin.
        ray_direction: (3,) array, normalized ray direction.
        pos_cache: Sparse 3D grid storing {sigma, u, v, w}. Access via nearest neighbor.
        dir_cache: Sparse 2D grid storing beta. Access via trilinear interpolation on theta/phi.
        D: Number of components (e.g., 8).
        bounds: [min_x, max_x, min_y, max_y, min_z, max_z] of the scene bounding box.
        step_size: Distance between samples (delta).
        
    Returns:
        color: (3,) array, RGB color.
    """
    # 1. Empty Space Skipping using Bounding Box (proxy for collision mesh)
    t_near, t_far = ray_aabb_intersect(ray_origin, ray_direction, bounds)
    if t_near is None:
        return (0.0, 0.0, 0.0)
    
    # Handle rays starting inside the volume
    t_start = max(0.0, t_near)
    t_end = t_far
    
    # 2. Sample points along the ray
    num_samples = int(round((t_end - t_start) / step_size))
    if num_samples <= 0:
        return (0.0, 0.0, 0.0)
        
    # Pre-fetch direction weights
    theta = math.acos(max(-1.0, min(1.0, ray_direction[2])))
    phi = math.atan2(ray_direction[1], ray_direction[0])
    beta = dir_cache.trilinear_sample(theta, phi)
    
    total_color = (0.0, 0.0, 0.0)
    transmittance = 1.0
    
    for i in range(num_samples):
        t = t_start + (i + 0.5) * step_size
        p = ray_origin + ray_direction * t
        
        # 3. Cache Lookup
        sigma, u, v, w = pos_cache.nearest_neighbor_sample(p)
        
        # 4. Color Computation
        r = sum(beta[j] * u[j] for j in range(D))
        g = sum(beta[j] * v[j] for j in range(D))
        b = sum(beta[j] * w[j] for j in range(D))
        c = (r, g, b)
        
        # 5. Volume Rendering Integration
        alpha = 1.0 - math.exp(-sigma * step_size)
        weight = transmittance * alpha
        total_color = (total_color[0] + weight * c[0], 
                       total_color[1] + weight * c[1], 
                       total_color[2] + weight * c[2])
        transmittance *= (1.0 - alpha)
        
        if transmittance < 1e-4:
            break
            
    return total_color
```

### Parameters
*   **$D$ (Components):** Dimensionality of the deep radiance map and direction weights. Typical values: 8 for NeRF Synthetic 360 scenes, 6 for LLFF scenes [S6, unofficial].
*   **$k$ (Position Grid Size):** Number of uniform samples for each worldspace coordinate ($x, y, z$) in the position cache bounding box. Typical values: 512, 768, 1024 [S6, unofficial].
*   **$l$ (Direction Grid Size):** Number of uniform samples for each ray direction coordinate ($\theta, \phi$). Typical values: 256 [S6, unofficial].
*   **Grid Resolution:** Number of divisions of the longest side of the bounding volume. Typical values: $256^3, 384^3, 512^3, 768^3, 1024^3$ [S6, unofficial].

### Edge Cases and Pitfalls
*   **Aliasing:** Caching the model to a grid after training can introduce aliasing artifacts, leading to a decrease in metrics as a function of grid resolution [S6, unofficial].
*   **Memory Limits:** Discretizing the full 5D space with 512 bins per dimension would require ~176 TB, which is infeasible. FastNeRF reduces this to ~54 GB (dense) or ~3 GB (sparse) [S6, unofficial][S5].
*   **Mesh Complexity:** For density volumes larger than $512^3$, the collision mesh must be downsampled by a factor of two to reduce complexity [S6, unofficial].
*   **Uncached Performance:** Without caching, FastNeRF is slower than standard NeRF due to larger MLPs (384 hidden units vs 256), though it performs identically to NeRF when using 256 hidden units [S6, unofficial].

## Subtopics

### [FastNeRF Cache Size Estimation](11428-fastnerf-cache-size-estimation.md)
Provides analytical formulas to calculate the memory footprint of the position-dependent and direction-dependent caches, solving the problem of determining if a configuration fits in hardware memory.

## Operating characteristics
*   **Time Complexity:** Inference is $O(N)$ where $N$ is the number of samples per ray, dominated by cache lookups and dot products, avoiding MLP evaluations [S6, unofficial].
*   **Memory Complexity:** Caching standard NeRF is $O(k^3 l^2)$. FastNeRF reduces this to $O(k^3 (1 + 3D) + l^2 D)$ [S6, unofficial]. For $k=l=1024, D=8$, dense cache size is ~54 GB, compared to ~5600 TB for standard NeRF [S6, unofficial]. Sparse cache size is ~3 GB [S5].
*   **Performance:** Renders high-fidelity photorealistic images at 200Hz on high-end consumer GPUs [S6, unofficial]. It is on average more than 3000x faster than NeRF [S6, unofficial]. On Nvidia Tesla V100, it renders 300x300 pixel images of a face at 30 fps in telepresence scenarios, ~50x faster than NeRF [S6, unofficial].
*   **Strengths:** Extremely fast inference; compatible with hardware-accelerated ray tracing; maintains visual quality comparable to NeRF [S6, unofficial].
*   **Limitations:** High memory usage for caches (several GBs); aliasing artifacts at lower resolutions; slower than NeRF when not cached [S6, unofficial].
*   **When to use:** Scenarios requiring interactive rates on consumer hardware, such as mobile devices, mixed reality, and telepresence [S6, unofficial].
*   **When not to use:** Memory-constrained embedded systems without large caches; scenarios where training speed is the primary bottleneck (FastNeRF does not address training speed) [S6, unofficial].

## Background
FastNeRF was proposed by Garbin et al. in March 2021 [S1]. It was motivated by the need for real-time neural rendering on mobile and mixed reality devices [S6, unofficial]. The method draws inspiration from real-time computer graphics techniques for approximating the rendering equation, such as spherical harmonics, by factorizing the 5D radiance field into position and direction components [S6, unofficial]. It is a key development in the evolution of NeRF acceleration, preceding methods like KiloNeRF and PlenOctrees [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S3] Neural Rendering and Its Hardware Acceleration: A Review, https://doi.org/10.48550/arxiv.2402.00028, fetched 2026-10-26 (unofficial)
[S4] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S5] SqueezeNeRF: Further factorized FastNeRF for memory-efficient inference, https://doi.org/10.1109/cvprw56347.2022.00307, fetched 2026-10-26
[S6] FastNeRF by Garbin et al., https://arxiv.org/abs/2103.10380, fetched 2026-10-26 (unofficial)
