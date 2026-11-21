# FastNeRF

Also called: Factorized NeRF, Factorized Neural Radiance Fields

## Overview
FastNeRF is a neural rendering method that accelerates the inference of Neural Radiance Fields (NeRF) by factorizing the view-dependent color function into position-dependent and direction-dependent components, enabling efficient caching of intermediate radiance values [S4, unofficial]. It solves the problem of high computational cost in standard NeRF, which requires hundreds of MLP evaluations per pixel, by reducing inference to memory-bound cache lookups and dot products, achieving speeds over 3000 times faster than the original NeRF [S1][S4, unofficial]. It differs from its siblings, such as [C-NeRF](7275-c-nerf.md) and [VoxGRAF](12633-voxgraf.md), by explicitly separating the 5D radiance field into two independent low-dimensional manifolds (position and direction) to allow for compact, sparse caching of a "deep radiance map" rather than caching the full 5D function or using smaller local MLPs [S4, unofficial][S3].

## How it works
FastNeRF builds on [Voxel-based Representation](10852-voxel-based-representation.md) by replacing the single monolithic MLP with two factorized networks and caching their outputs.

### Inputs and Outputs
*   **Inputs:** A set of training images with known camera poses; a bounding box $V$ enclosing the scene.
*   **Outputs:** A trained position-dependent network $\mathcal{F}_{\text{pos}}$, a direction-dependent network $\mathcal{F}_{\text{dir}}$, and optionally, cached grids for inference.

### Data Structures
1.  **Position Cache:** A sparse 3D grid storing outputs of $\mathcal{F}_{\text{pos}}$.
    *   **Fields:** Density $\sigma$ (scalar) and a deep radiance map components $(\mathbf{u}, \mathbf{v}, \mathbf{w})$ (each $D$-dimensional vectors).
    *   **Layout:** Stored as dense 16-bit floating point values in a grid of size $k^3$ (or sparse subset thereof).
2.  **Direction Cache:** A sparse 2D grid storing outputs of $\mathcal{F}_{\text{dir}}$.
    *   **Fields:** Weight vector $\boldsymbol{\beta}$ ($D$-dimensional).
    *   **Layout:** Stored as dense 16-bit floating point values in a grid of size $l^2$ (parameterized by spherical coordinates $\theta, \phi$).
3.  **Collision Mesh:** A triangle mesh derived from the density volume $\sigma$ using Marching Cubes, used for hardware-accelerated ray tracing to skip empty space [S4, unofficial].

### Network Architecture
*   **$\mathcal{F}_{\text{pos}}$:** An 8-layer MLP with 384 hidden units (256 for LLFF scenes). Input: 3D position $\mathbf{p}$. Output: $\{\sigma, (\mathbf{u}, \mathbf{v}, \mathbf{w})\}$.
*   **$\mathcal{F}_{\text{dir}}$:** A 4-layer MLP with 128 hidden units. Input: 2D viewing direction $\mathbf{d}$ (parameterized as a unit vector or spherical coordinates). Output: $\boldsymbol{\beta}$.
*   **Positional Encoding:** Applied to inputs of both MLPs, as in standard NeRF [S4, unofficial].

### Training Steps
1.  Define bounding box $V$ for the scene.
2.  Sample $k$ uniform values for $x, y, z$ within $V$ and $l$ uniform values for $\theta \in [0, \pi], \phi \in [0, 2\pi]$.
3.  Compute $\mathcal{F}_{\text{pos}}$ for all position combinations and $\mathcal{F}_{\text{dir}}$ for all direction combinations to initialize caches (optional, primarily for initialization or analysis; training typically optimizes the MLPs directly on rendered images).
4.  Train $\mathcal{F}_{\text{pos}}$ and $\mathcal{F}_{\text{dir}}$ jointly using the standard NeRF loss (MSE between rendered and ground truth colors) [S4, unofficial].

### Inference Steps
1.  **Ray Generation:** For each pixel, generate a ray $\mathbf{r}$ defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
2.  **Empty Space Skipping:** Intersect $\mathbf{r}$ with the collision mesh. Start integration only after the first hit [S4, unofficial].
3.  **Sampling:** Uniformly sample $N$ points $\mathbf{p}_i$ along the ray within the mesh bounds.
4.  **Cache Lookup:**
    *   For each $\mathbf{p}_i$, lookup $\sigma_i$ and $(\mathbf{u}_i, \mathbf{v}_i, \mathbf{w}_i)$ from the Position Cache using nearest-neighbor interpolation [S4, unofficial].
    *   For the ray direction $\mathbf{d}$, lookup $\boldsymbol{\beta}$ from the Direction Cache using trilinear sampling (on the $\theta, \phi$ grid) [S4, unofficial].
5.  **Color Computation:** Compute color $\mathbf{c}_i$ for each sample point:
    $$
    \mathbf{c}_i = \boldsymbol{\beta}^T \cdot (\mathbf{u}_i, \mathbf{v}_i, \mathbf{w}_i) = \sum_{j=1}^D \beta_j (u_{i,j}, v_{i,j}, w_{i,j})
    $$
    where $D$ is the number of components (typically 8 for Synthetic 360, 6 for LLFF) [S4, unofficial][S3].
6.  **Volume Rendering:** Integrate color and density along the ray using the standard NeRF volume rendering equation [44-radiance-field-optimization.md]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ and $\delta_i$ is the distance between samples [S4, unofficial].
7.  **Termination:** Terminate ray integration once transmittance $T_i$ drops below a threshold (saturation) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def fast_nerf_infer(ray_origin, ray_direction, pos_cache, dir_cache, D, k, l):
    """
    Renders a single pixel using FastNeRF caches.
    
    Args:
        ray_origin: (3,) array, camera origin.
        ray_direction: (3,) array, normalized ray direction.
        pos_cache: Sparse 3D grid storing {sigma, u, v, w}.
                   Access via nearest neighbor interpolation.
        dir_cache: Sparse 2D grid storing beta.
                   Access via trilinear interpolation on theta/phi.
        D: Number of components (e.g., 8).
        k: Grid resolution for position (x,y,z).
        l: Grid resolution for direction (theta, phi).
        
    Returns:
        color: (3,) array, RGB color.
    """
    # 1. Empty Space Skipping using Collision Mesh
    # Note: Collision mesh is pre-computed from density volume via Marching Cubes.
    hit_point, t_near, t_far = intersect_ray_with_mesh(ray_origin, ray_direction)
    if hit_point is None:
        return (0.0, 0.0, 0.0) # Background color
    
    # 2. Sample points along the ray within the mesh bounds
    num_samples = 1024 # Typical value, can vary
    samples = []
    for i in range(num_samples):
        t = t_near + (t_far - t_near) * (i + 0.5) / num_samples
        p = ray_origin + ray_direction * t
        samples.append(p)
        
    # 3. Lookup from Caches
    # Pre-fetch direction weights once per ray
    # Convert ray_direction to spherical coordinates (theta, phi)
    theta = math.acos(ray_direction[2]) # Assuming z-up or standard convention
    phi = math.atan2(ray_direction[1], ray_direction[0])
    beta = dir_cache.trilinear_sample(theta, phi) # Shape: (D,)
    
    # 4. Compute Color and Density for each sample
    sigmas = []
    colors = []
    
    for p in samples:
        # Lookup position cache
        sigma, u, v, w = pos_cache.nearest_neighbor_sample(p)
        sigmas.append(sigma)
        
        # Compute color via inner product
        # c = beta . (u, v, w)
        # u, v, w are each D-dimensional vectors
        color = dot_product(beta, u, v, w, D)
        colors.append(color)
        
    # 5. Volume Rendering Integration
    total_color = (0.0, 0.0, 0.0)
    transmittance = 1.0
    
    # Calculate step size delta
    delta = (t_far - t_near) / num_samples
    
    for i in range(num_samples):
        sigma = sigmas[i]
        c = colors[i]
        
        # Alpha for this segment
        alpha = 1.0 - math.exp(-sigma * delta)
        
        # Weight for this segment
        weight = transmittance * alpha
        
        # Accumulate color
        total_color += weight * c
        
        # Update transmittance
        transmittance *= (1.0 - alpha)
        
        # Early termination if transmittance is negligible
        if transmittance < 1e-4:
            break
            
    return total_color

def dot_product(beta, u, v, w, D):
    """
    Computes the inner product of beta with the deep radiance map (u, v, w).
    """
    r = 0.0
    g = 0.0
    b = 0.0
    for i in range(D):
        r += beta[i] * u[i]
        g += beta[i] * v[i]
        b += beta[i] * w[i]
    return (r, g, b)
```

### Parameters
*   **$D$ (Components):** Dimensionality of the deep radiance map and direction weights. Typical values: 8 for NeRF Synthetic 360 scenes, 6 for LLFF scenes [S4, unofficial].
*   **$k$ (Position Grid Size):** Number of uniform samples for each worldspace coordinate ($x, y, z$) in the position cache bounding box. Typical values: 512, 768, 1024 [S4, unofficial].
*   **$l$ (Direction Grid Size):** Number of uniform samples for each ray direction coordinate ($\theta, \phi$). Typical values: 256, 512 [S4, unofficial].
*   **Grid Resolution:** Number of divisions of the longest side of the bounding volume. Typical values: $256^3, 384^3, 512^3, 768^3, 1024^3$ [S4, unofficial].

### Edge Cases and Pitfalls
*   **Aliasing:** Caching the model to a grid after training can introduce aliasing artifacts, leading to a decrease in metrics as a function of grid resolution [S4, unofficial].
*   **Memory Limits:** Discretizing the full 5D space with 512 bins per dimension would require ~176 TB, which is infeasible. FastNeRF reduces this to ~54 GB (dense) or ~3 GB (sparse) [S4, unofficial][S3].
*   **Mesh Complexity:** For density volumes larger than $512^3$, the collision mesh must be downsampled by a factor of two to reduce complexity [S4, unofficial].
*   **Uncached Performance:** Without caching, FastNeRF is slower than standard NeRF due to larger MLPs (384 hidden units vs 256) [S4, unofficial].

## Subtopics

### [FastNeRF Cache Size Estimation](11428-fastnerf-cache-size-estimation.md)
Provides analytical formulas to calculate the memory footprint of the position-dependent and direction-dependent caches, solving the problem of determining if a configuration fits in hardware memory.

## Operating characteristics
*   **Time Complexity:** Inference is $O(N)$ where $N$ is the number of samples per ray, dominated by cache lookups and dot products, avoiding MLP evaluations [S4, unofficial].
*   **Memory Complexity:** Caching standard NeRF is $O(k^3 l^2)$. FastNeRF reduces this to $O(k^3 (1 + 3D) + l^2 D)$ [S4, unofficial]. For $k=l=1024, D=8$, dense cache size is ~54 GB, compared to ~5600 TB for standard NeRF [S4, unofficial]. Sparse cache size is ~3 GB [S3].
*   **Performance:** Renders high-fidelity photorealistic images at 200Hz on high-end consumer GPUs [S4, unofficial]. It is on average more than 3000x faster than NeRF [S4, unofficial]. On Nvidia Tesla V100, it renders 300x300 pixel images of a face at 30 fps in telepresence scenarios, ~50x faster than NeRF [S4, unofficial].
*   **Strengths:** Extremely fast inference; compatible with hardware-accelerated ray tracing; maintains visual quality comparable to NeRF [S4, unofficial].
*   **Limitations:** High memory usage for caches (several GBs); aliasing artifacts at lower resolutions; slower than NeRF when not cached [S4, unofficial][S3].
*   **When to use:** Scenarios requiring interactive rates on consumer hardware, such as mobile devices, mixed reality, and telepresence [S4, unofficial].
*   **When not to use:** Memory-constrained embedded systems without large caches; scenarios where training speed is the primary bottleneck (FastNeRF does not address training speed) [S4, unofficial].

## Background
FastNeRF was proposed by Garbin et al. in March 2021 [S1]. It was motivated by the need for real-time neural rendering on mobile and mixed reality devices [S4, unofficial]. The method draws inspiration from real-time computer graphics techniques for approximating the rendering equation, such as spherical harmonics, by factorizing the 5D radiance field into position and direction components [S4, unofficial]. It is a key development in the evolution of NeRF acceleration, preceding methods like KiloNeRF and PlenOctrees [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S3] SqueezeNeRF: Further factorized FastNeRF for memory-efficient inference, https://doi.org/10.1109/cvprw56347.2022.00307, fetched 2026-10-26
[S4] FastNeRF by Garbin et al., https://arxiv.org/abs/2103.10380, fetched 2026-10-26 (unofficial)
