# Primitive Volume Rendering

Also called: Segment-Based Radiative Transfer

## Overview
Primitive Volume Rendering is a physically-based volume rendering technique that represents a volumetric scene as a collection of localized volumetric primitives, each defined by a density kernel, weight, emission, and phase function [S1]. It solves the radiative transfer equation by partitioning the viewing ray into segments at the entry and exit points of these primitives, allowing for efficient integration of in-scattered radiance [S1]. The method naturally decomposes into different frequency bands and is orientation-selective, maximizing rendering speed and compression without requiring new pyramid levels for level-of-detail (LOD) [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N$ volumetric primitives, where each primitive $i$ is characterized by a density kernel $K_i(\mathbf{x})$, a weight $\alpha_i$, emission properties, and a phase function [S1]. A camera ray defined by origin $\mathbf{x}_0$ and direction $\vec{v}$ [S1].
*   **Output:** The accumulated radiance $L(\mathbf{x}_0, \mathbf{x}_M, \vec{v})$ along the ray segment from $\mathbf{x}_0$ to the final intersection $\mathbf{x}_M$ [S1].

**Data Structures**
*   **Primitive:** Each primitive $i$ stores:
    *   $\alpha_i$: Scalar weight.
    *   $K_i(\mathbf{x})$: Density kernel function. The kernel has limited support bounded by an ellipsoid with a primary axis radius of $3\sigma$. For efficiency, this support is approximated by a triangle mesh [S1].
    *   Emission and phase function parameters (omitted in core formulas for conciseness but required for full radiance calculation) [S1].
*   **Ray Segments:** A list of intersection points $\{\mathbf{x}_1, \dots, \mathbf{x}_M\}$ along the ray, sorted by distance from $\mathbf{x}_0$. These points are determined by the entry and exit intersections of the ray with the bounding meshes of the primitives [S1].

**Formulas**
The extinction field $\kappa_t$ at any point $\mathbf{x}$ is the sum of contributions from all $N$ influencing primitives:
$$ \kappa_t(\mathbf{x}) = \sum_{i=1}^N \alpha_i K_i(\mathbf{x}) \quad (1) $$

The optical depth contribution $\tau_i$ of primitive $i$ along a ray segment $[a, b]$ is:
$$ \tau_i(\mathbf{x}_a, \mathbf{x}_b) = \alpha_i \int_a^b K_i(\mathbf{x}_t) dt \quad (2) $$
where $\mathbf{x}_t = \mathbf{x}_0 + \vec{v} \cdot t$ is the ray parameterization.

The total transmittance $T$ through the segment $[a, b]$ is:
$$ T(\mathbf{x}_a, \mathbf{x}_b) = \exp\left(-\sum_{i=1}^N \tau_i(\mathbf{x}_a, \mathbf{x}_b)\right) \quad (3) $$

The final radiance is computed by summing the contributions of each segment $k$ from $1$ to $M$:
$$ L(\mathbf{x}_0, \mathbf{x}_M, \vec{v}) = \sum_{k=1}^M T_{k-1}(\mathbf{x}_0, \mathbf{x}_k) L_k(\mathbf{x}_{k-1}, \mathbf{x}_k, \vec{v}) \quad (4) $$
where $T_{k-1}$ is the cumulative transmittance from $\mathbf{x}_0$ to $\mathbf{x}_{k-1}$, and $L_k$ is the integrated in-scattered radiance within segment $k$ [S1].

**Sampling Interaction Distance**
To sample the distance to the next interaction event, the cumulative density function is inverted. For a random sample $\xi \in (0, 1)$:
$$ \log(1 - \xi) = -\sum_i \tau_i(\mathbf{x}_0, \mathbf{x}_i) \quad (5) $$
This equation is solved for the intersection point $\mathbf{x}_i$ using root-finding methods (e.g., Newton-Raphson, bisection) or closed-form inversion depending on the choice of kernels [S1].

**Parameters**
*   $N$: The number of primitives influencing a point $\mathbf{x}$ [S1].
*   $\sigma$: The scale parameter of the kernel, defining the ellipsoidal support radius of $3\sigma$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def primitive_volume_rendering(ray_origin, ray_direction, primitives):
    """
    Renders a ray through a set of volumetric primitives.
    
    Args:
        ray_origin: 3D vector x_0
        ray_direction: 3D vector v (assumed normalized)
        primitives: List of Primitive objects, each with:
            - alpha: weight
            - kernel: function K_i(x)
            - mesh: triangle mesh bounding the support (radius 3*sigma)
            - emission: emission coefficient
            - phase: phase function
            
    Returns:
        radiance: float, accumulated radiance
    """
    # 1. Compute ray-primitive intersections to determine segments
    intersection_ts = []
    for prim in primitives:
        # Intersect ray with prim.mesh to get t_entry and t_exit
        # Standard ray-triangle intersection logic [G]
        t_vals = intersect_ray_mesh(ray_origin, ray_direction, prim.mesh)
        for t in t_vals:
            if t > 0:
                intersection_ts.append(t)
    
    # Sort and remove duplicates to get sorted segment boundaries
    sorted_ts = sorted(set(intersection_ts))
    
    # Ensure ray_origin (t=0) is the start of the first segment
    if not sorted_ts or sorted_ts[0] > 0.0:
        sorted_ts.insert(0, 0.0)
        
    segments = []
    for i in range(len(sorted_ts) - 1):
        t_start = sorted_ts[i]
        t_end = sorted_ts[i+1]
        # Determine which primitives are active in this segment
        active_prims = []
        for prim in primitives:
            # Check if segment [t_start, t_end] overlaps with prim.mesh bounds
            if segment_overlaps_mesh(t_start, t_end, prim.mesh):
                active_prims.append(prim)
        segments.append({
            't_start': t_start,
            't_end': t_end,
            'active_prims': active_prims
        })
        
    # 2. Integrate radiance over segments
    total_radiance = 0.0
    cumulative_transmittance = 1.0
    
    for seg in segments:
        t_a = seg['t_start']
        t_b = seg['t_end']
        active_prims = seg['active_prims']
        
        # Calculate optical depth for this segment
        total_optical_depth = 0.0
        for prim in active_prims:
            # Compute integral of prim.kernel along ray segment
            # Use closed-form if available, otherwise numerical quadrature [G]
            integral_K = numerical_integral_kernel(prim.kernel, ray_origin, ray_direction, t_a, t_b)
            tau_i = prim.alpha * integral_K
            total_optical_depth += tau_i
            
        # Transmittance through this segment
        seg_transmittance = math.exp(-total_optical_depth)
        
        # Calculate in-scattered radiance L_k for this segment
        # [G] Source omits exact L_k formula for conciseness. 
        # Standard radiative transfer integrates emission * phase * density 
        # weighted by transmittance from origin over the segment.
        L_k = compute_in_scattered_radiance(active_prims, ray_origin, ray_direction, t_a, t_b, cumulative_transmittance)
        
        # Add contribution: L = sum(T_{k-1} * L_k)
        total_radiance += cumulative_transmittance * L_k
        
        # Update cumulative transmittance for next segment
        cumulative_transmittance *= seg_transmittance
        
    return total_radiance

def numerical_integral_kernel(kernel_func, origin, direction, t_start, t_end, num_samples=16):
    """Numerically integrates the kernel function along the ray segment using midpoint rule."""
    sum_val = 0.0
    dt = (t_end - t_start) / num_samples
    for i in range(num_samples):
        t = t_start + (i + 0.5) * dt
        point = origin + direction * t
        sum_val += kernel_func(point)
    return sum_val * dt

def segment_overlaps_mesh(t_start, t_end, mesh):
    """Checks if the ray segment [t_start, t_end] intersects the primitive's support mesh."""
    # [G] Standard geometric overlap test between ray segment and triangle mesh bounds
    return True  # Placeholder for actual mesh intersection logic

def compute_in_scattered_radiance(active_prims, origin, direction, t_start, t_end, prev_transmittance):
    """Computes the in-scattered radiance L_k for the segment."""
    radiance = 0.0
    num_samples = 16
    dt = (t_end - t_start) / num_samples
    for i in range(num_samples):
        t = t_start + (i + 0.5) * dt
        point = origin + direction * t
        # Evaluate extinction and emission at midpoint
        kappa_t = sum(prim.alpha * prim.kernel(point) for prim in active_prims)
        # [G] Simplified emission/scattering model consistent with primitive inputs
        emission = sum(prim.emission for prim in active_prims)
        phase = sum(prim.phase for prim in active_prims)
        # Integrate: T(x_0, x_t) * emission * phase * density
        seg_T = math.exp(-kappa_t * t)
        radiance += seg_T * emission * phase * kappa_t * dt
    return radiance

def intersect_ray_mesh(origin, direction, mesh):
    """Returns list of positive t values where ray intersects mesh triangles."""
    # [G] Standard ray-triangle intersection algorithm
    return []
```

**Edge Cases and Pitfalls**
*   **Dense Scenes:** For dense scenes with many overlapping primitives, uniform sampling within segments is used as a bias-performance trade-off [S1].
*   **Kernel Support:** The kernel support is bounded by an ellipsoid with radius $3\sigma$, approximated by a triangle mesh. Inaccuracies in this approximation may lead to artifacts at the boundaries of primitives [S1].
*   **Root-Finding:** Solving Eq. (5) for interaction distance requires robust root-finding. The choice between Newton-Raphson, bisection, or closed-form inversion depends on the kernel choice [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of primitives $N$ intersecting the ray and the cost of evaluating the kernel integrals. For dense scenes, uniform sampling reduces the cost of root-finding but may introduce bias [S1].
*   **Strengths:** Natural decomposition into frequency bands and orientation-selectivity, which maximizes rendering speed and compression. It avoids the need for computing new pyramid levels for LOD, allowing continuous spectral representation by masking parts of the asset [S1].
*   **Limitations:** The approximation of ellipsoidal support by triangle meshes may introduce geometric inaccuracies. The effectiveness of uniform sampling in dense scenes is a trade-off between bias and performance [S1].
*   **When to Use:** Suitable for scenes requiring efficient LOD and compression, where orientation-selectivity is beneficial [S1].

## Background
Primitive Volume Rendering was proposed as a pure primitive-based volumetric model that decomposes into different frequency bands and is orientation-selective [S1]. It builds on the concept of physically-based volume rendering using volumetric primitives, where each primitive defines a localized region of matter [S1]. The method was introduced in "Gabor Fields: Orientation-Selective Level-of-Detail for Volume Rendering" [S1].

## Sources
[S1] Gabor Fields: Orientation-Selective Level-of-Detail for Volume Rendering, https://arxiv.org/abs/2602.05081, fetched 2026-10-27
