# Per-primitive Ray Sampling

Also called: Primitive-centric ray marching

## Overview
Per-primitive ray sampling is a rendering strategy where each geometric primitive (specifically ellipsoids) independently updates only the ray segments it intersects, avoiding redundant parameter loads and enabling accurate Transfer Function (TF) mapping by accumulating contributions in 3D space [S1, unofficial]. It sets itself apart from sibling methods like [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md) and [Volume Ray Casting](3161-volume-ray-casting.md) by organizing computation around primitives rather than rays, which eliminates the need for explicit per-tile sorting required in 2D rasterization because contributions accumulate in depth order as each ray segment corresponds to a discrete depth interval [S1, unofficial].

## How it works
**Inputs and Outputs:** The method takes a set of primitives, each defined by center $\mu$, covariance $\Sigma$, support radius factor $r_{supp}$, intensity $I$, and scaling factor $k$; a sampling step size $\Delta t$; a projection matrix; and a pixel grid [S1, unofficial]. The output is a set of accumulated ray segments `ray_i` for each pixel, which are subsequently traversed front-to-back for alpha blending to produce the final pixel color [S1, unofficial].

**Data Structures:** Each primitive is defined by $\mu = (\mu_x, \mu_y, \mu_z)$, covariance matrix $\Sigma$, support radius factor $r_{supp}$, and intensity $I$ [S1, unofficial]. Each ray segment is a 1D array `ray_i` for pixel $i$, discretized with step size $\Delta t$, where the $j$-th entry stores the accumulated intensity at depth $t = j\Delta t$ [S1, unofficial].

**Steps:** 
1. **Transform and Cull:** Transform primitives by the projection matrix and process only those visible in screen space [S1, unofficial].
2. **Cache Parameters:** Cache ellipsoid parameters in tile-level shared memory to minimize redundant global memory accesses across pixels [S1, unofficial].
3. **Determine Depth Range:** For a primitive with center $\mu$ and covariance $\Sigma$, determine the valid depth range [S1, unofficial]. Segments within $|t - \mu_z| \le r_{supp} \cdot \sqrt{\Sigma'_{zz}}$ are considered, where $\sqrt{\Sigma'_{zz}}$ is the standard deviation along the z-axis after projection [S1, unofficial].
4. **Update Ray Segments:** For each primitive, calculate the start and end indices of the affected ray segments and update them with the primitive's contribution [S1, unofficial].

**Formulas:** 
* **Ray Parameterization:** In ray space, the ray for pixel $(x_i, y_i)$ is $r_i(t) = (x_i, y_i, 0) + t \cdot (0, 0, 1)$, where $t$ varies along the z-axis [S1, unofficial].
* **Sample Position:** The sample position for index $j$ is $ray_i[j] = \hat{V}(r_i(j\Delta t)) = \hat{V}(x_i, y_i, j\Delta t)$ [S1, unofficial].
* **Primitive Contribution:** The contribution of a primitive to ray segment $j$ is:
  $$ \tilde{P}(x_i, y_i, j \cdot \Delta t) = I \cdot \frac{\sigma(k \cdot (1 - \mathbf{d}^\top \Sigma^{-1} \mathbf{d}))}{\sigma(k)} $$
  where $\mathbf{d} = [x_i, y_i, j\Delta t] - [\mu_x, \mu_y, \mu_z]$ is the displacement from the primitive center, $k$ is a scaling factor, and $\sigma$ is the sigmoid function [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def per_primitive_ray_sampling(primitives, delta_t, ray_segments, pixel_coords):
    """
    primitives: List of primitives with attributes mu, Sigma, r_supp, I, k, screen_x_min, screen_x_max, screen_y_min, screen_y_max
    delta_t: float, sampling step size
    ray_segments: 2D list [H][W] of lists of floats, initialized to 0.0
    pixel_coords: list of (x_i, y_i) tuples for each pixel
    """
    for prim in primitives:
        mu_z = prim.mu[2]
        sigma_zz = prim.Sigma[2, 2]
        r_supp = prim.r_supp
        I = prim.I
        k = prim.k
        Sigma_inv = inverse(prim.Sigma)

        # Calculate depth range indices
        z_min = mu_z - r_supp * sqrt(sigma_zz)
        z_max = mu_z + r_supp * sqrt(sigma_zz)
        index_start = floor(z_min / delta_t)
        index_end = floor(z_max / delta_t)

        # Sources do not specify the exact pixel culling loop; standard screen-space bounding box iteration is used [G].
        for y in range(prim.screen_y_min, prim.screen_y_max + 1):
            for x in range(prim.screen_x_min, prim.screen_x_max + 1):
                pixel_id = y * W + x
                x_i, y_i = pixel_coords[pixel_id]

                for j in range(index_start, index_end + 1):
                    d_x = x_i - prim.mu[0]
                    d_y = y_i - prim.mu[1]
                    d_z = (j * delta_t) - prim.mu[2]
                    d = [d_x, d_y, d_z]

                    quadratic = dot(d, dot(Sigma_inv, d))
                    numerator = 1.0 / (1.0 + exp(-k * (1.0 - quadratic)))
                    denominator = 1.0 / (1.0 + exp(-k))
                    contribution = I * (numerator / denominator)

                    ray_segments[pixel_id][j] += contribution
```

**Edge Cases and Pitfalls:** 
* **Precision:** Floating-point precision in calculating $d^T \Sigma^{-1} d$ can affect the sigmoid evaluation, especially for large displacements [G].
* **Boundaries:** The support radius factor $r_{supp}$ determines the cutoff; if too small, contributions are truncated, and if too large, unnecessary computations occur [S1, unofficial].
* **Degenerate Input:** If $\Sigma$ is singular, $\Sigma^{-1}$ is undefined, requiring a small regularization term [G].

## Operating characteristics
* **Time Complexity:** Depends on the number of primitives and the average number of ray segments they intersect, avoiding the $O(N_{rays} \cdot N_{steps})$ complexity of naive ray marching by only processing relevant primitives for each segment [G].
* **Memory Complexity:** Requires storing ray segments for all pixels and caching primitive parameters in shared memory [S1, unofficial].
* **Performance:** Eliminates redundant global memory traffic by processing each primitive once and caching parameters in tile-level shared memory [S1, unofficial].
* **Strengths:** Accurate TF evaluation in 3D space, no explicit sorting required, and reduced memory traffic [S1, unofficial].
* **Limitations:** Requires efficient culling and bounding box calculation to avoid processing non-intersecting primitives, and performance may degrade with high primitive density and overlap [S1, unofficial].

## Background
Per-primitive ray sampling was introduced in the context of 3D Ellipsoid-based Sparse Volume Rendering (ESVR) [S1, unofficial]. It addresses the overhead of naively adopting ray marching for learned primitives, where each ray step requires checking intersections and repeatedly fetching ellipsoid parameters from memory [S1, unofficial]. By organizing computation around primitives, it enables real-time rendering with accurate TF mapping [S1, unofficial].

## Sources
[S1] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
