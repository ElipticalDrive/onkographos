# FastNeRF

Also called: FastNeRF by Garbin et al
## Overview
FastNeRF is a method that accelerates Neural Radiance Fields (NeRF) by caching view-independent radiance values at selected 3D points sorted in a sparse spatial structure, such as voxel grid vertices [S1][S2, unofficial]. It solves the high memory and computational cost of evaluating a full MLP for every sample along a ray [S1][S2, unofficial]. Unlike standard NeRF, which evaluates the network on the fly, FastNeRF precomputes and stores these values, enabling high-fidelity photorealistic rendering at 200fps on high-end consumer GPUs [S1][S2, unofficial]. It sets itself apart from siblings like V4D and RenderNet by explicitly trading model compactness for rendering speed via explicit spatial caching [S1][S2, unofficial].
## How it works
### Inputs and Outputs
**Inputs:**
- A set of precomputed view-independent radiance values (density $\sigma$ and view-independent color components) stored in a sparse spatial structure (e.g., voxel grid).
- Camera parameters (position, orientation, intrinsics).
- Ray origins and directions.

**Outputs:**
- Rendered RGB images.

### Data Structures
The method relies on two main caches:
1.  **Position Cache:** Stores view-independent information at voxel grid vertices.
    -   Fields: Density ($\sigma$) and view-independent color factors.
    -   Layout: Sparse spatial structure (voxel grid).
    -   Size: Determined by $k^3$ bins per dimension, where $k$ is the number of bins per dimension in the position-dependent cache.
2.  **Direction Cache:** Stores view-dependent weights.
    -   Fields: Weights $\beta_i$.
    -   Layout: Dense or sparse structure depending on implementation, indexed by direction.
    -   Size: Determined by $l^2$ bins per dimension, where $l$ is the number of bins per dimension in the direction-dependent cache.

### Formulas
The memory size for the standard NeRF cache is calculated as [S2, unofficial]:
$$M_{NeRF} = \alpha(s_\sigma + s_{rgb})k^3l^2$$
where:
-   $s_\sigma$: Size of the stored transparency/density value in bits.
-   $s_{rgb}$: Size of the stored RGB values in bits.
-   $\alpha \in [0, 1)$: Inverse volume sparsity ($\alpha = 1$ indicates a dense volume).
-   $k$: Number of bins per dimension in the position-dependent cache.
-   $l$: Number of bins per dimension in the direction-dependent cache.

The memory size for the FastNeRF cache is calculated as [S2, unofficial]:
$$M_{FastNeRF} = \alpha \left( (D \cdot s_{uvw} + s_{\sigma})k^3 \right) + D \cdot s_{\beta}l^2$$
where:
-   $D$: Number of components/factors.
-   $s_{uvw}$: Size of the stored $(u_i, v_i, w_i)$ values (view-independent color factors).
-   $s_{\sigma}$: Size of the stored transparency/density value in bits.
-   $s_{\beta}$: Size of the weights $\beta_i$.
-   $k, l, \alpha$: As defined above.

### Parameters and Typical Values
-   $k, l$: Number of bins per dimension. Typical value: 1024 [S2, unofficial].
-   $D$: Number of factors. Typical value: 8 [S2, unofficial].
-   Data types: $(r, g, b)$ stored as individual bytes; other values as half-precision floats [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_fastnerf(ray_origin, ray_direction, position_cache, direction_cache, alpha, k, l, D):
    """
    Renders a single ray using FastNeRF caching.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        position_cache: Sparse structure storing view-independent radiance (sigma, u, v, w).
        direction_cache: Structure storing view-dependent weights (beta).
        alpha: Inverse volume sparsity.
        k: Number of bins per dimension in position cache.
        l: Number of bins per dimension in direction cache.
        D: Number of components/factors.
        
    Returns:
        color: 3D vector, final RGB color.
        depth: float, accumulated depth.
    """
    # Initialize accumulation variables
    color = [0.0, 0.0, 0.0]
    depth = 0.0
    transmittance = 1.0
    
    # Sample points along the ray
    # Note: The exact sampling strategy (e.g., uniform, stratified, or adaptive) 
    # is not specified in the sources. We assume a standard NeRF-like sampling loop.
    for sample_point in sample_along_ray(ray_origin, ray_direction):
        
        # 1. Retrieve view-independent radiance from position cache
        # Interpolate or lookup sigma, u, v, w at sample_point in position_cache
        sigma, u, v, w = lookup_position_cache(sample_point, position_cache)
        
        # 2. Retrieve view-dependent weights from direction cache
        # Interpolate or lookup beta_i for ray_direction in direction_cache
        betas = lookup_direction_cache(ray_direction, direction_cache)
        
        # 3. Compute view-dependent color
        # The view-dependent color is a weighted sum of spherical harmonic bases 
        # or factors. FastNeRF factorizes this.
        # Assuming the factors (u, v, w) are combined with weights (betas)
        # to produce the final color contribution.
        # Specific combination formula for u, v, w and betas is not detailed 
        # in the provided notes, but typically involves a dot product or linear combination.
        # [G] Assuming a simple linear combination for demonstration:
        view_independent_color = [u, v, w] # Placeholder for actual factor combination
        view_dependent_color = dot_product(view_independent_color, betas)
        
        # 4. Compute opacity (transmittance) contribution
        # Using Beer-Lambert Law
        # delta_t is the distance between samples
        delta_t = distance_between_samples(sample_point, previous_sample_point)
        opacity = 1.0 - exp(-sigma * delta_t)
        
        # 5. Accumulate color and depth
        weight = transmittance * opacity
        color += weight * view_dependent_color
        depth += weight * sample_point.z # Assuming z is depth along ray
        transmittance *= (1.0 - opacity)
        
        # 6. Early termination
        if transmittance < 0.001:
            break
            
    return color, depth
```

### Edge Cases and Pitfalls
-   **Memory Usage:** Caching radiance values into explicit spatial data structures comes at the expense of model compactness [S1][S2, unofficial].
-   **Hardware Limits:** Even for highly sparse volumes with $\alpha = 0.01$, the NeRF cache size is nearly 60TB, which exceeds the memory capacity of consumer-grade hardware, making the standard NeRF architecture impractical without caching optimizations [S2, unofficial].
-   **Precision:** The choice of data types (bytes for RGB, half-precision for others) impacts memory and precision [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** Rendering is significantly faster than standard NeRF due to cache lookups instead of MLP evaluations. High-fidelity photorealistic images can be rendered at 200fps on a high-end consumer GPU [S1][S2, unofficial].
-   **Memory Complexity:** The memory requirement is high due to the explicit spatial structure. For $k=l=1024$, $D=8$, and specific data types, the NeRF cache size is approximately $\alpha \cdot 5.6$ PB, while FastNeRF reduces this to approximately $\alpha \cdot 54$ GB (specifically $\alpha \cdot 53,687,091,200 + 16,777,216$) [S2, unofficial].
-   **Strengths:** High rendering speed, suitable for real-time applications on consumer hardware.
-   **Limitations:** High memory consumption, less compact than implicit representations.
-   **When to use:** When real-time rendering speed is critical and sufficient GPU memory is available.
-   **When not to use:** When memory is constrained or high compactness is required.

## Background
FastNeRF was introduced by Garbin et al. in 2021 [S2, unofficial]. It builds on the NeRF architecture by Garbin et al. [S1][S2, unofficial]. The method was developed to address the high computational cost of NeRF by caching view-independent radiance values [S1][S2, unofficial]. It is used in applications requiring real-time novel view synthesis [S1][S2, unofficial].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] FastNeRF by Garbin et al., https://arxiv.org/abs/2103.10380, fetched 2026-10-26 (unofficial)
