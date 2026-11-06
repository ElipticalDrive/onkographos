# Per-primitive Ray Sampling

Also called: Primitive-centric ray marching

## Overview
Per-primitive ray sampling is a rendering strategy where each geometric primitive (specifically ellipsoids) independently updates only the ray segments it intersects, avoiding redundant parameter loads and enabling accurate Transfer Function (TF) mapping by accumulating contributions in 3D space [S1, unofficial]. It sets itself apart from sibling methods like [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md) and [Image-order Ray-Casting](3161-image-order-ray-casting.md) by organizing computation around primitives rather than rays, which eliminates the need for explicit per-tile sorting required in 2D rasterization because contributions accumulate in depth order as each ray segment corresponds to a discrete depth interval [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of primitives, each defined by center $\mu$, covariance $\Sigma$, support radius factor $r_{supp}$, and intensity $I$; a sampling step size $\Delta t$; and a projection matrix.
*   **Outputs:** Accumulated ray segments `ray_i` for each pixel, which are subsequently traversed front-to-back for alpha blending [S1, unofficial].

**Data Structures:**
*   **Primitive:** Defined by $\mu = (\mu_x, \mu_y, \mu_z)$, covariance matrix $\Sigma$, support radius factor $r_{supp}$, and intensity $I$.
*   **Ray Segment:** A 1D array `ray_i` for pixel $i$, discretized with step size $\Delta t$. The $j$-th entry stores the accumulated intensity at depth $t = j\Delta t$.

**Steps:**
1.  **Transform and Cull:** Transform primitives by the projection matrix. Process only those visible in screen space [S1, unofficial].
2.  **Cache Parameters:** Cache ellipsoid parameters in tile-level shared memory to minimize redundant global memory accesses across pixels [S1, unofficial].
3.  **Determine Depth Range:** For a primitive with center $\mu$ and covariance $\Sigma$, determine the valid depth range. Segments within $|t - \mu_z| \le r_{supp} \cdot \sqrt{\Sigma'_{zz}}$ are considered, where $\sqrt{\Sigma'_{zz}}$ is the standard deviation along the z-axis after projection [S1, unofficial].
4.  **Update Ray Segments:** For each primitive, calculate the start and end indices of the affected ray segments and update them with the primitive's contribution [S1, unofficial].

**Formulas:**
*   **Ray Parameterization:** In ray space, the ray for pixel $(x_i, y_i)$ is $r_i(t) = (x_i, y_i, 0) + t \cdot (0, 0, 1)$, where $t$ varies along the z-axis [S1, unofficial].
*   **Sample Position:** The sample position for index $j$ is $ray_i[j] = \hat{V}(r_i(j\Delta t)) = \hat{V}(x_i, y_i, j\Delta t)$ [S1, unofficial].
*   **Primitive Contribution:** The contribution of a primitive to ray segment $j$ is:
    $$ \tilde{P}(x_i, y_i, j \cdot \Delta t) = I \cdot \frac{\sigma(k \cdot (1 - \mathbf{d}^\top \Sigma^{-1} \mathbf{d}))}{\sigma(k)} $$
    where $\mathbf{d} = [x_i, y_i, j\Delta t] - [\mu_x, \mu_y, \mu_z]$ is the displacement from the primitive center, $k$ is a scaling factor, and $\sigma$ is the sigmoid function [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def per_primitive_ray_sampling(primitives, delta_t, ray_segments):
    """
    primitives: List of primitives, each with attributes:
        - mu: (mu_x, mu_y, mu_z)
        - Sigma: 3x3 covariance matrix
        - r_supp: support radius factor
        - I: intensity
        - k: scaling factor (assumed constant or per-primitive)
    delta_t: float, sampling step size
    ray_segments: Dict mapping pixel_id to list of floats (accumulated intensity)
    """
    # Step 1 & 2: Transform and Cache (Implementation detail: handled by hardware/cache)
    # For each primitive, determine its valid depth range and update ray segments.
    
    for prim in primitives:
        mu = prim.mu
        Sigma = prim.Sigma
        r_supp = prim.r_supp
        I = prim.I
        k = prim.k
        
        # Calculate projected standard deviation along z-axis
        # Note: Sigma'_{zz} is the z-component of covariance after projection.
        # In the source, this is denoted as Sigma'_{zz}. We assume Sigma provided is already in ray space or transformed.
        # If Sigma is in world space, it must be transformed by the projection matrix first.
        # Here we assume Sigma is the relevant covariance for the ray space calculation.
        sigma_zz_sqrt = sqrt(Sigma[2, 2]) 
        
        # Determine valid depth range indices
        # index_start = floor((mu_z - r_supp * sigma_zz_sqrt) / delta_t)
        # index_end = floor((mu_z + r_supp * sigma_zz_sqrt) / delta_t)
        
        # However, the source formula for index calculation is:
        # index_start = floor((mu_z - r * sqrt(Sigma_zz)) / delta_t)
        # index_end = floor((mu_z + r * sqrt(Sigma_zz)) / delta_t)
        
        # We need to iterate over pixels that this primitive affects.
        # The source implies a loop over primitives, then over ray segments.
        # But ray segments are per-pixel. The source algorithm 1 is simplified.
        # A complete implementation requires mapping the primitive's screen-space bounding box to pixels.
        # Since the source does not provide the pixel mapping logic, we assume a helper function
        # that yields (pixel_id, j) for all ray segments j affected by the primitive.
        
        affected_segments = get_affected_segments(prim, delta_t, sigma_zz_sqrt)
        
        for pixel_id, j in affected_segments:
            # Calculate displacement d
            # Ray position at j: (x_i, y_i, j * delta_t)
            # We need x_i, y_i for the pixel. This is typically derived from pixel_id.
            x_i, y_i = get_pixel_coords(pixel_id)
            
            d_x = x_i - mu[0]
            d_y = y_i - mu[1]
            d_z = (j * delta_t) - mu[2]
            d = [d_x, d_y, d_z]
            
            # Calculate quadratic form d^T Sigma^{-1} d
            # Sigma_inv = inverse(Sigma)
            # quadratic = dot(d, dot(Sigma_inv, d))
            # For efficiency, if Sigma is diagonal, this simplifies.
            # General case:
            Sigma_inv = inverse(Sigma)
            quadratic = dot(d, dot(Sigma_inv, d))
            
            # Calculate contribution
            # sigma(k * (1 - quadratic)) / sigma(k)
            # sigma(x) = 1 / (1 + exp(-x))
            
            numerator = sigmoid(k * (1 - quadratic))
            denominator = sigmoid(k)
            
            contribution = I * (numerator / denominator)
            
            # Update ray segment
            ray_segments[pixel_id][j] += contribution

def get_affected_segments(prim, delta_t, sigma_zz_sqrt):
    """
    Returns a list of (pixel_id, j) tuples for ray segments affected by the primitive.
    This function encapsulates the logic of determining which pixels and depth indices
    fall within the primitive's projected extent.
    """
    mu = prim.mu
    r_supp = prim.r_supp
    
    # Depth range
    z_min = mu[2] - r_supp * sigma_zz_sqrt
    z_max = mu[2] + r_supp * sigma_zz_sqrt
    
    index_start = floor(z_min / delta_t)
    index_end = floor(z_max / delta_t)
    
    # Determine pixel range
    # This requires the primitive's screen-space bounding box.
    # The source mentions "Transform primitives by the projection matrix".
    # We assume the primitive's extent in x,y is known or can be derived from Sigma.
    # For a full implementation, one would project the ellipsoid's bounding box.
    # Since the source does not provide this, we return a placeholder range.
    # In practice, this would be a loop over pixels in the bounding box.
    
    # Placeholder: Assume we have a function to get pixels in bounding box
    pixels = get_pixels_in_bbox(prim)
    
    segments = []
    for pixel_id in pixels:
        for j in range(index_start, index_end + 1):
            segments.append((pixel_id, j))
            
    return segments

def sigmoid(x):
    return 1.0 / (1.0 + exp(-x))

def get_pixel_coords(pixel_id):
    # Returns (x_i, y_i) for the given pixel ID
    # Implementation depends on rasterization grid
    pass

def get_pixels_in_bbox(prim):
    # Returns list of pixel IDs within the primitive's screen-space bounding box
    # Implementation depends on projection and bounding box calculation
    pass
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision in calculating $d^T \Sigma^{-1} d$ can affect the sigmoid evaluation, especially for large displacements.
*   **Boundaries:** The support radius factor $r_{supp}$ determines the cutoff. If too small, contributions are truncated; if too large, unnecessary computations occur.
*   **Degenerate Input:** If $\Sigma$ is singular, $\Sigma^{-1}$ is undefined. A small regularization term may be needed [G].

## Operating characteristics
*   **Time Complexity:** Depends on the number of primitives and the average number of ray segments they intersect. It avoids the $O(N_{rays} \cdot N_{steps})$ complexity of naive ray marching by only processing relevant primitives for each segment.
*   **Memory Complexity:** Requires storing ray segments for all pixels and caching primitive parameters in shared memory.
*   **Performance:** Eliminates redundant global memory traffic by processing each primitive once and caching parameters in tile-level shared memory [S1, unofficial].
*   **Strengths:** Accurate TF evaluation in 3D space, no explicit sorting required, reduced memory traffic.
*   **Limitations:** Requires efficient culling and bounding box calculation to avoid processing non-intersecting primitives. Performance may degrade with high primitive density and overlap.

## Background
Per-primitive ray sampling was introduced in the context of 3D Ellipsoid-based Sparse Volume Rendering (ESVR) [S1, unofficial]. It addresses the overhead of naively adopting ray marching for learned primitives, where each ray step requires checking intersections and fetching parameters [S1, unofficial]. By organizing computation around primitives, it enables real-time rendering with accurate TF mapping [S1, unofficial].

## Sources
[S1] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
