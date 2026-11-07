# Ray Casting and Compositing Loop

Also called: MLS Ray Casting, Moving Least Squares Volume Ray Casting

## Overview
This method performs direct volume rendering by integrating radiative energy along rays using Moving Least Squares (MLS) approximation for scalar field reconstruction and precomputed integration tables for optical accumulation [S1, unofficial]. It distinguishes itself from standard voxel-based ray casting by handling irregular or sparse data distributions without requiring a regular grid, relying on local neighborhood queries to define the basis functions [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Scalar field data points (irregular or sparse), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), transfer function $T(d)$, and precomputed integration tables.
- Output: Final pixel color $C_{final}$ and accumulated opacity $\alpha_{final}$.

**Data Structures:**
1.  **Spatial Search Structure:** A dynamic redundant octree (Guennebaud et al.) used for efficient neighbor queries in GPU implementations [S1, unofficial].
2.  **Preintegration Tables:** Lookup tables storing the integral of the MLS basis functions weighted by the transfer function over specific interval lengths [S1, unofficial].
3.  **Local Basis Matrix:** For each evaluation point, a covariance matrix $\mathbf{B}^T \mathbf{W}(\mathbf{x}) \mathbf{B}$ is computed, where $\mathbf{B}$ is the basis matrix and $\mathbf{W}$ is the weight matrix [S1, unofficial].

**Steps:**
1.  **Ray-Volume Intersection:** Intersect the ray $r(t) = \mathbf{o} + t\mathbf{d}$ with the boundary of the volume hull to determine entry $t_{start}$ and exit $t_{end}$ intervals [S1, unofficial]. Boundary points are critical because the MLS approximation is ill-defined outside the data support [S1, unofficial].
2.  **Adaptive Interval Determination:** Along each ray segment, determine integration intervals using an adaptive scheme [S1, unofficial].
3.  **MLS Evaluation:** At the front and back of each interval, compute the MLS function value and its gradient [S1, unofficial]. This involves:
    -   Querying neighbors via the spatial search structure.
    -   Constructing the weight matrix $\mathbf{W}$ based on distances to neighbors.
    -   Solving the normal equation $(\mathbf{B}^T \mathbf{W} \mathbf{B}) \mathbf{c} = \mathbf{B}^T \mathbf{W} \mathbf{d}$ for coefficients $\mathbf{c}$ using Cholesky decomposition [S1, unofficial].
4.  **Preintegration Lookup:** Compute the volume integration along the interval by looking up values in the preintegration tables based on the interval length and the computed MLS parameters [S1, unofficial]. If the interval length is not present in the tables, combine values from adjacent tables [S1, unofficial].
5.  **Compositing:** Composite the resulting color and opacity with the accumulated values on the ray using the front-to-back over operator [S1, unofficial].
6.  **Early Ray Termination:** Terminate the ray if the accumulated opacity $\alpha'$ exceeds a threshold (typically 0.99) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_cast_mls(ray_origin, ray_direction, volume_data, transfer_function, preintegration_tables, octree):
    # 1. Intersect ray with volume hull
    t_start, t_end = intersect_ray_hull(ray_origin, ray_direction, volume_data.boundary)
    
    accumulated_color = 0.0
    accumulated_opacity = 0.0
    transmittance = 1.0
    
    # 2. Determine integration intervals (adaptive scheme)
    intervals = determine_adaptive_intervals(t_start, t_end, ray_direction, volume_data)
    
    for interval in intervals:
        t_front, t_back = interval.start, interval.end
        interval_length = t_back - t_front
        
        # 3. Evaluate MLS function and gradient at interval boundaries
        # Query neighbors using octree
        neighbors_front = octree.query_neighbors(ray_origin + t_front * ray_direction, radius)
        neighbors_back = octree.query_neighbors(ray_origin + t_back * ray_direction, radius)
        
        # Compute MLS coefficients at front and back
        # Solve normal equation: (B^T W B) c = B^T W d
        coeffs_front = solve_normal_equation(neighbors_front, transfer_function)
        coeffs_back = solve_normal_equation(neighbors_back, transfer_function)
        
        # 4. Lookup preintegration table
        # Table lookup based on interval_length and MLS parameters
        color_contrib, opacity_contrib = lookup_preintegration_table(
            preintegration_tables, interval_length, coeffs_front, coeffs_back
        )
        
        # 5. Composite
        # C' = C' + (1 - alpha') * alpha * C
        # alpha' = alpha' + (1 - alpha') * alpha
        accumulated_color += transmittance * opacity_contrib * color_contrib
        accumulated_opacity += transmittance * opacity_contrib
        transmittance *= (1.0 - opacity_contrib)
        
        # 6. Early Ray Termination
        if accumulated_opacity > 0.99:
            break
            
    return accumulated_color, accumulated_opacity
```

**Parameters:**
-   Early ray termination threshold: 0.99 [S1, unofficial].
-   Neighbor radius: Determined by the spatial search structure's configuration [S1, unofficial].

**Edge Cases and Pitfalls:**
-   The MLS function approximation is ill-defined away from data points, making accurate boundary intersection crucial [S1, unofficial].
-   If an interval length is not present in the preintegration tables, values must be carefully combined from different tables, adding minor complexity [S1, unofficial].

## Subtopics

### CUDA GPU Implementation
A variant optimized for GPU execution where each thread processes a single ray [S1, unofficial]. Ray traversal is decoupled from the volume data structure, using a dynamic redundant octree for neighbor queries [S1, unofficial]. Recursion for adaptive stepping is managed via a small stack in local memory [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Dominated by MLS evaluation (neighborhood query and linear system solution) per interval [S1, unofficial]. The linear system solution is relatively cheap after covariance matrix computation [S1, unofficial].
-   **Memory Complexity:** Depends on the size of the neighborhood and the number of degrees of freedom in the basis [S1, unofficial]. The dynamic redundant octree allows control over memory footprint [S1, unofficial].
-   **Performance:** Measured on irregular tetrahedral grids (e.g., 1.2M tets) showing significant speedup on GPU compared to CPU [S1, unofficial].
-   **Strengths:** Handles irregular and sparse data without regular grid constraints; robust boundary handling [S1, unofficial].
-   **Limitations:** MLS evaluation is computationally intensive; ill-defined outside data support [S1, unofficial].

## Background
Originated in the "Volume MLS Ray Casting" paper, which introduced Moving Least Squares approximation for volume rendering to handle irregular data distributions [S1, unofficial]. The method was later implemented on CUDA GPUs to achieve interactive performance [S1, unofficial].

## Sources
[S1] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
