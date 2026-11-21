# Transmitted Caustic Estimation via Local Space Path Tracing

Also called: Caustic Estimator, Two-Point Tracing for Caustics

## Overview
An unbiased estimator for transmitted caustics through glass panels with surface perturbations, bubbles, and heterogeneous refractive index (RIF) [S1]. It solves the difficulty of sampling Specular-Diffuse-Specular (SDS) paths in complex glass assemblies, which bidirectional path tracers struggle to capture effectively [S1]. Unlike [Unidirectional Path Tracing](2558-unidirectional-path-tracing.md) which may miss these specific paths, this method transforms the problem into a local panel space to simplify intersection and estimation [S1].

## How it works
**Inputs:**
*   Ray origin and direction in world space.
*   Composite glass panel geometry (mesh, surface perturbations, bubbles, RIF).
*   Transformation matrix $T$ mapping world space to local panel space $R_0$.

**Outputs:**
*   Radiance contribution of the transmitted caustic path.
*   Intersection point $p$ in world space.

**Data Structures:**
*   **Transformation $T$**: Rigid transformation (rotation and translation) from world space to local panel space $R_0$.
*   **Frame Vectors**: Tangent $e_x$, bitangent $e_y$, and normal $e_z$ at intersection point $p$.
*   **Points**: $x_0$ (receiver/end point), $x_n$ (source/end point) in world space.

**Steps:**
1.  **World-to-Local Intersection**:
    *   Intersect the ray with the composite object mesh in world space to find intersection point $p$ and frame vectors $\{e_x, e_y, e_z\}$ [S1].
    *   Construct the rigid transformation $T$ using the frame vectors [S1].
    *   Transform the ray from world space to local space $R_0$ using $T$ [S1].
    *   Intersect the transformed ray with the source panel in local space to find intersection point $p'$ [S1].
    *   Transform $p'$ back to world space using $T^{-1}$ to obtain the final intersection point $p$ [S1].

2.  **Local Space Caustic Estimation**:
    *   Transform the two end points of the path, $x_0$ and $x_n$, from world space to local panel space using $T$ [S1].
    *   Estimate the caustic contribution in the local panel space [S1].
    *   Since $T$ is rigid, the contribution computed in local space can be used directly in world space [S1].

3.  **Local-to-World Transformation**:
    *   Transform the resulting path connecting the receiver to the source back to world space using $T^{-1}$ [S1].

**Formulas:**
*   Transformation: $p_{local} = T \cdot p_{world}$
*   Inverse Transformation: $p_{world} = T^{-1} \cdot p_{local}$
*   Note: The specific formula for the "generalized geometric factor" is not provided in the sources, but its regularization is critical for variance reduction [S1].

**Parameters:**
*   **Regularization**: Applied to the generalized geometric factor to reduce outliers. Trade-off: may cause local underestimation of the caustic [S1].

**Edge Cases and Pitfalls:**
*   **High Variance**: Occurs if the generalized geometric factor is not regularized, especially on panels with strong geometric displacement or RIF variations [S1].
*   **Low Convergence**: The solver has low convergence rates for strong geometric displacement or RIF variations, leading to high variance or darker-than-ground-truth caustics [S1].
*   **Outliers**: Implementation may produce outliers. Regularization can partially eliminate them but risks local underestimation [S1].
*   **Cost**: The number of possible connections increases dramatically for panels with finely detailed geometric displacements, making unbiased inverse probability estimation costly [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_caustic(ray_origin, ray_direction, panel, T):
    # Step 1: World-to-Local Intersection
    # Intersect ray with mesh in world space
    p_world, ex, ey, ez = intersect_mesh(ray_origin, ray_direction, panel.mesh)
    
    # Construct rigid transformation T from frame vectors
    # T is assumed to be pre-computed or derived from ex, ey, ez
    # Transform ray to local space
    ray_local_origin = T @ ray_origin
    ray_local_direction = T @ ray_direction
    
    # Intersect with source panel in local space
    p_local_prime = intersect_panel(ray_local_origin, ray_local_direction, panel.source_panel)
    
    # Transform back to world space
    p_final = T.inverse() @ p_local_prime
    
    # Step 2: Local Space Caustic Estimation
    # Transform end points x0 (receiver) and xn (source) to local space
    x0_local = T @ x0_world
    xn_local = T @ xn_world
    
    # Estimate caustic in local space
    # Note: Specific estimator algorithm details are not provided in sources [S1]
    # This step involves computing the path contribution using local geometry
    contribution_local = estimate_caustic_local(x0_local, xn_local, panel.local_geometry)
    
    # Step 3: Return contribution
    # Since T is rigid, contribution can be used directly
    return contribution_local
```

## Subtopics

### Panel Modeling
Represents cylinder/crown blown glass, surface deformation, bubbles, and heterogeneous RIF [S1].

### Variance Reduction
Uses regularization of the generalized geometric factor to reduce outliers, trading off potential local underestimation [S1].

### Comparison with Bidirectional Methods
Unlike [Unidirectional Path Tracing](2558-unidirectional-path-tracing.md) or bidirectional methods like Pediredla [12], this method effectively samples SDS paths [S1].

## Operating characteristics
*   **Time Complexity**: High for complex panels. Rendering time was 20.69 hours for 4M samples per pixel on a 3D assembly of stained glass [S1].
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: On an Nvidia RTX A6000 GPU, 1 minute of rendering time produced a comparison image against Mitsuba3 native path tracing [S1].
*   **Strengths**: Unbiased; handles heterogeneous RIF and surface perturbations; effectively samples SDS paths [S1].
*   **Limitations**: High variance with strong geometric displacement or RIF variations; low convergence rates for complex panels; outliers; costly unbiased inverse probability estimation for detailed displacements [S1].
*   **When to Use**: For unbiased caustic estimation in complex glass assemblies with surface perturbations and RIF where bidirectional methods fail to sample SDS paths [S1].
*   **When Not to Use**: For simple glass panels where standard path tracing is sufficient; for real-time applications due to high rendering times [S1].

## Background
Originated from research on parabolic sphere tracing of signed distance fields for old glass modelling and rendering [S1]. Implemented as a plugin for the Mitsuba3 rendering engine [S1]. The method addresses the lack of unbiased methods capable of handling both specular interfaces and spatially varying refractive index [S1]. Future work includes generalizing to general sets of SDF, bubbles, and RIF voxel grids [S1].

## Sources
[S1] Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering, https://doi.org/10.1109/tvcg.2025.3613853, fetched 2026-11-05
