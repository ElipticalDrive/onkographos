# Local Field Interpolation with Dynamic Subdivision

Also called: Dynamic Subdivision for SCALIS, Fast ray tracing of scale-invariant integral surfaces, Local Field Interpolation with Dynamic Subdivision for SCALIS, SCALIS rendering pipeline, Scale-Invariant Integral Surface (SCALIS) Ray Tracing

## Overview
This method is a real-time ray tracing pipeline for visualizing Scale-Invariant Integral Surfaces (SCALIS) defined by skeletons with prescribed radii [S1]. It solves the high computational cost of individual SCALIS field evaluations and the challenge of robust closest ray/isosurface intersection by relying on local field interpolations and subdividing the ray into intervals that capture main field variations [S1]. It sets itself apart from brute-force marching by using a dynamic view-dependent acceleration structure to select primitives and transforming field values to a space allowing good quadratic polynomial approximation [S1].

## How it works
**Inputs and Outputs:**
- Input: A camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, a skeleton structure consisting of line segment primitives $S_i$ with prescribed radii at endpoints, and a target isovalue $c$ [S1].
- Output: The intersection distance $t$ along the ray where the SCALIS field $f(\mathbf{p}) = c$ [S1].

**Data Structures:**
- **Skeleton Primitive $S_i$**: A line segment defined by endpoints $\mathbf{v}_0, \mathbf{v}_1$ and radii $r_0, r_1$ [S1].
- **Radius Function $\tau_{S_i}(q)$**: For a point $q$ on the segment, the radius is the linear interpolation of the endpoint radii [S1].
- **Dynamic A-Buffer**: A GPU-accelerated structure built on the fly to select primitives whose supports overlap a given interval along the ray [S1].

**Field Definitions:**
The SCALIS field is an implicit surface defined by a scalar field $f$ and an isovalue $c$ as the set of points $\mathbf{p}$ satisfying $f(\mathbf{p}) = c$ [S1]. The field is constructed from primitives using the sum blending operator:
$$f(\mathbf{p}) = \sum_i f_i(\mathbf{p}) \quad (1)$$
where $f_i$ is the field function defined by the $i$-th primitive [S1]. The primitives are based on kernels. Compact Polynomial kernels are defined as:
$$k_{i,\sigma}(d) = \begin{cases} \left(1 - \left(\frac{d}{\sigma}\right)^2\right)^{\frac{i}{2}} & \text{if } d < \sigma, \\ 0 & \text{otherwise.} \end{cases} \quad (2)$$
and Cauchy kernels are defined as:
$$k_{i,\sigma}(d) = \frac{1}{\left(1+\left(\frac{d}{\sigma}\right)^2\right)^i} \quad (3)$$
where $i$ is the degree parameter and $\sigma$ is the scale parameter [S1].

**Algorithm Steps:**

1.  **Primitive Selection**: Select primitives whose supports overlap a given interval along the ray using a dynamic view-dependent acceleration structure built on the fly using fast GPU construction of A-buffers [S1].
2.  **Ray Segmentation**: Segment the camera ray into sub-intervals based on the distance to the skeleton. The strategy is chosen to capture main field variations while generating a small number of sub-intervals to be processed [S1].
3.  **Field Mapping**: Transform field values to a space that allows good quadratic polynomial approximation of the SCALIS field on each sub-interval. This is achieved by using a bijective mapping of the SCALIS field to an approximate smooth homothetic distance field (squared) which has the exact same isosurface of interest as the SCALIS field [S1]. All processing is done on this normalized field [S1].
4.  **Iterative Interpolation and Root Finding**: For each sub-interval, iteratively build polynomial interpolations using fast quadratic roots computation to rapidly converge toward the real field function [S1]. A different interpolation is used and refined in each interval, locally producing an accurate approximation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def trace_scalis_ray(ray_origin, ray_direction, skeleton_segments, isovalue):
    # 1. Primitive Selection (Dynamic A-Buffer simulation)
    # Filter primitives whose bounding spheres overlap the ray segment [t_min, t_max]
    # The sources describe a dynamic A-buffer built on the fly [S1]
    active_primitives = []
    for seg in skeleton_segments:
        R = max(seg.r0, seg.r1) + math.dist(seg.v0, seg.v1)
        oc = ray_origin - seg.center
        b = math.dot(oc, ray_direction)
        c = math.dot(oc, oc) - R * R
        if c > 0.0 and b > 0.0:
            continue
        disc = b * b - c
        if disc >= 0.0:
            active_primitives.append(seg)

    # 2. Ray Segmentation
    # Divide ray into intervals based on distance to skeleton to capture field variations
    # The exact subdivision heuristic is not specified in the sources [G]
    intervals = compute_intervals(ray_origin, ray_direction, active_primitives)

    t_hit = None
    t_min, t_max = 0.0, 1e6

    for t_start, t_end in intervals:
        if t_start > t_max:
            break
        t_end = min(t_end, t_max)
        if t_start >= t_end:
            continue

        # 3. Field Mapping
        # Map SCALIS field to approximate squared homothetic distance field
        # The isovalue c maps to 0 in the normalized space [S1]
        target_normalized = 0.0
        t_guess = t_start

        for _ in range(50):
            p = ray_origin + t_guess * ray_direction
            f_val = evaluate_scalis_field(p, active_primitives)
            # Bijective mapping to normalized squared distance space
            f_norm = normalize_to_squared_distance(f_val, isovalue)

            if abs(f_norm - target_normalized) < 1e-6:
                t_hit = t_guess
                break

            # Compute derivatives along ray direction
            # f'(t) = grad(f) . d, f''(t) = d . Hessian(f) . d
            df = compute_ray_derivative(p, ray_direction, active_primitives)
            d2f = compute_ray_second_derivative(p, ray_direction, active_primitives)

            if abs(df) < 1e-12:
                df = 1e-12

            # Fast quadratic root computation: solve f_norm + df*dt + 0.5*d2f*dt^2 = 0
            a = 0.5 * d2f
            b = df
            c = f_norm - target_normalized
            if abs(a) < 1e-12:
                dt = -c / b
            else:
                disc = b * b - 4 * a * c
                if disc < 0:
                    disc = 0
                dt = (-b - math.sqrt(disc)) / (2 * a)

            t_guess += dt
            t_guess = max(t_start, min(t_end, t_guess))

            if abs(dt) < 1e-8:
                t_hit = t_guess
                break

        if t_hit is not None:
            break

    return t_hit
```

**Parameters:**
- `max_iterations`: Fixed to 50 in the implementation to prevent infinite loops. Not given in the sources.
- Convergence thresholds (`1e-6` for field value, `1e-8` for step size) are standard numerical values. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision**: The method relies on the bijective mapping to a normalized field to ensure good quadratic approximation. If the mapping is inaccurate, convergence may fail [S1].
- **Boundaries**: The ray segmentation must ensure that intervals are small enough to capture main field variations but large enough to minimize overhead [S1].
- **Degenerate Input**: The method assumes a skeleton structure with line segment primitives. The sources do not specify handling for degenerate segments [G].

## Operating characteristics
- **Time Complexity**: The method reduces the number of field evaluations by using local interpolations and dynamic subdivision [S1]. The dynamic A-buffer construction is performed on the fly using fast GPU construction [S1].
- **Memory Complexity**: Requires memory for the dynamic A-buffer and the skeleton structure [S1].
- **Measured Performance**: The method provides improvements for various resolutions on all combinations of tested models and GPU hardware [S1]. It goes beyond Sherstyuk's fast ray-tracing algorithm for convolution surfaces, notably allowing better support of large radius variation while being less sensitive to skeleton tessellation [S1].
- **Strengths**: Real-time visualization of SCALIS, efficient primitive selection, robust intersection finding [S1].
- **Limitations**: High computational cost of individual SCALIS field evaluations is mitigated but not eliminated; the method is specific to SCALIS fields [S1].

## Background
Scale-invariant integral surfaces (SCALIS) are implicit representations of surfaces that provide a way to define smooth surfaces from skeletons with prescribed radii defined at their vertices [S1]. The rendering pipeline was introduced to visualize such surfaces in real-time, addressing challenges in robust intersection, efficient primitive selection, and high computational cost [S1]. The method builds on previous work in metaballs rendering and transparency using A-buffers [S1]. It improves upon Sherstyuk's fast ray-tracing algorithm for convolution surfaces [S1].

## Sources
[S1] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
