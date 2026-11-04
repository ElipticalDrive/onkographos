# Unidirectional Volume Boundary Estimator

Also called: Main-path side-path sampling, Unidirectional estimator

## Overview
The Unidirectional Volume Boundary Estimator is a Monte Carlo method for estimating radiance from volume scattering events occurring exactly on the boundary of participating media. It solves the problem of efficiently sampling the singular contribution of boundary interactions within a unidirectional path tracing framework. Unlike standard volume path tracing which samples interior scattering, this method constructs a "main" path from the detector and generates "side" paths from main path vertices to the volume boundary and then to the light source. It is distinct from the Bidirectional Volume Boundary Estimator, which draws the boundary vertex first and connects it to both the detector and the light source independently.

## How it works
The estimator samples volume boundary paths $\vec{\mathbf{p}}$ from the volume boundary path space $\partial \hat{\Omega}$ to estimate the integral in Eq. (36) of [S1]. The core idea is to build a main path from the detector and use its vertices to spawn side paths that terminate at the volume boundary $\partial BV$ and connect to a light source.

**Inputs and Outputs**
*   **Input:** Scene geometry (including translucent objects with boundaries $\partial BV$), participating media properties, light sources, camera position.
*   **Output:** An unbiased estimate of the radiance contribution from volume boundary scattering events.

**Data Structures**
*   **Main Path:** A sequence of vertices $\vec{\mathbf{p}}^D = (p^D_0, p^D_1, \dots)$, where $p^D_0$ is the detector position.
*   **Side Path:** A sequence of vertices $\vec{\mathbf{p}}^S_n = (p^S_{n,0}, \dots, p^S_{n,m_n})$ connecting the boundary vertex to the light.
*   **Volume Boundary Vertex:** A point $p^B_n \in \partial BV$.

**Algorithm Steps**
1.  **Build Main Path:** Construct a main path $\vec{\mathbf{p}}^D = (p^D_0, p^D_1, \dots)$ using unidirectional volume path tracing starting from the detector $p^D_0$. This involves sampling directions based on the BSDF or phase function at each vertex and tracing rays through the medium [S1].
2.  **Russian Roulette Decision:** For each vertex $p^D_n$ in the main path, stochastically decide whether to generate a side path. This decision is based on the throughput of the main path to avoid high computational overhead when the main path is long. If the random number exceeds the throughput, skip side path generation for this vertex [S1].
3.  **Sample Direction:** If generating a side path, draw a direction $\omega_n$ based on the BSDF or phase function at the current main path vertex $p^D_n$ [S1].
4.  **Find Boundary Intersection:** Trace a ray from $p^D_n$ in direction $\omega_n$. Find the intersection point $p^B_n$ with the "null" interfaces $\partial BV$ (the boundaries of translucent objects). This point $p^B_n$ corresponds to a volume scattering event happening exactly on the boundary [S1].
5.  **Connect to Light:** Connect $p^B_n$ to a light source via a subpath $\vec{\mathbf{p}}^S_n = (p^S_{n,0}, \dots, p^S_{n,m_n})$ where $m_n \ge 0$. This subpath is constructed using unidirectional sampling with next-event estimation [S1].
6.  **Form Volume Boundary Path:** Construct the full volume boundary path $\vec{\mathbf{p}} = (p^S_{n,m_n}, \dots, p^S_{n,0}, p^B_n, p^D_n, \dots, p^D_0)$. The vertex $p^B_n$ is the boundary vertex $p_K$ in the estimator formula [S1].

**Estimator Formula**
The radiance estimate for a sampled path $\vec{\mathbf{p}}$ is given by [S1]:

$$
\langle \mathbf{I}_{\text{bnd}} \rangle = \frac{f(\vec{\mathbf{p}}) \nu_{\perp}(\mathbf{p}_K)}{\text{pdf}(\vec{\mathbf{p}})}
$$

Where:
*   $f(\vec{\mathbf{p}})$ is the path contribution (product of BSDFs, phase functions, geometry terms, etc.).
*   $\nu_{\perp}(\mathbf{p}_K)$ is the perpendicular velocity (or geometric term related to the boundary intersection) at the boundary vertex $\mathbf{p}_K$ (which is $p^B_n$).
*   $\text{pdf}(\vec{\mathbf{p}})$ is the probability density of sampling the entire path $\vec{\mathbf{p}}$, including the main path, the direction $\omega_n$, the boundary intersection, and the side path to the light [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unidirectional_volume_boundary_estimator(scene, camera, light_sources):
    # 1. Build main path from detector
    main_path = []
    current_pos = camera.position
    throughput = 1.0
    direction = camera.direction
    
    # Standard unidirectional volume path tracing loop
    while throughput > 0.01: # Russian roulette threshold
        # Sample medium interaction or surface
        hit = trace_ray(current_pos, direction, scene)
        
        if hit is None:
            break
            
        main_path.append(hit)
        
        # Sample BSDF/Phase function for next direction
        new_direction = sample_bsdf_or_phase(hit, direction)
        throughput *= evaluate_bsdf_or_phase(hit, direction, new_direction) * geometry_term(hit, new_direction)
        
        # Russian Roulette for path termination
        if random() > throughput:
            break
            
        current_pos = hit.position + new_direction * epsilon
        direction = new_direction

    # 2. Generate side paths from main path vertices
    total_estimate = 0.0
    
    for n, p_D_n in enumerate(main_path):
        # Russian roulette to skip side path generation for long main paths
        if random() > throughput:
            continue
            
        # 3. Draw direction based on BSDF/Phase at p_D_n
        omega_n = sample_bsdf_or_phase(p_D_n, main_path[n-1].direction if n > 0 else camera.direction)
        
        # 4. Trace ray to find boundary intersection p_B_n
        ray = Ray(p_D_n.position, omega_n)
        boundary_hit = trace_ray_to_boundary(ray, scene) # Finds intersection with null interfaces \partial BV
        
        if boundary_hit is None:
            continue
            
        p_B_n = boundary_hit.position
        
        # 5. Connect p_B_n to light source via subpath
        # Construct subpath using unidirectional sampling with next-event estimation
        side_path = construct_side_path_to_light(p_B_n, light_sources, scene)
        
        if side_path is None:
            continue
            
        # 6. Form full path and compute contribution
        # Path: side_path reversed + [p_B_n] + main_path[n:] reversed + [p_D_0]
        full_path = reverse(side_path) + [p_B_n] + main_path[n:]
        
        # Calculate path contribution f(p)
        f_p = compute_path_contribution(full_path, scene)
        
        # Calculate pdf of the full path
        pdf_p = compute_path_pdf(full_path, scene)
        
        # Perpendicular velocity at boundary vertex p_K = p_B_n
        nu_perp = compute_perpendicular_velocity(p_B_n, omega_n)
        
        # Estimator
        estimate = (f_p * nu_perp) / pdf_p
        total_estimate += estimate
        
    return total_estimate
```

**Edge Cases and Pitfalls**
*   **Boundary Intersection:** The ray from $p^D_n$ must correctly identify "null" interfaces $\partial BV$. If the ray misses the boundary or intersects a non-null interface, the side path generation should be skipped or handled differently [S1].
*   **Throughput Decay:** The throughput of the main path can become very small, leading to high variance if Russian roulette is not applied correctly. The Russian roulette probability should be based on the current throughput to maintain unbiasedness [S1].
*   **Degenerate Input:** If the main path vertex $p^D_n$ is already on the boundary, the direction $\omega_n$ might be tangent to the boundary, leading to no intersection. A small epsilon offset or alternative sampling strategy may be needed [G].

## Subtopics

### Bidirectional Volume Boundary Estimator
A Monte Carlo estimator that samples volume boundary paths by first drawing the volume boundary vertex $p^B$ from the volume boundary path space, then constructing two subpaths: one from $p^B$ to the light source and another from $p^B$ to the detector, both using unidirectional sampling with next-event estimation. It is conceptually similar to the multi-directional technique by Zhang et al. [ZMY* 20] for sampling ordinary boundary paths, but applied to volume boundary paths. It is combined with the unidirectional estimator using Multiple Importance Sampling (MIS) with the balanced heuristic [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the length of the main path and the number of side paths generated. Russian roulette limits the number of side paths for long main paths, reducing overhead [S1].
*   **Memory Complexity:** Proportional to the length of the main path and the side paths.
*   **Strengths:** Efficiently samples boundary scattering events within a unidirectional framework. Supports full radiative transfer model including multiple scattering, anisotropic phase functions, and heterogeneity [S1].
*   **Limitations:** May have high variance if the boundary intersection is rare or if the throughput is very low. Requires efficient ray-boundary intersection testing [S1].
*   **When to use:** When boundary scattering is significant and a unidirectional path tracer is already in use.
*   **When not to use:** When boundary scattering is negligible or when a bidirectional approach is more suitable for the scene geometry.

## Background
The Unidirectional Volume Boundary Estimator was introduced in the paper "Efficient Path-Space Differentiable Volume Rendering With Respect To Shapes" by Zhang et al. [S1]. It is part of a framework that includes both unidirectional and bidirectional estimators for volume boundary paths, combined using Multiple Importance Sampling (MIS). The method builds on the principles of unidirectional path tracing and extends them to handle the specific case of volume scattering events occurring on the boundary of participating media.

## Sources
[S1] Efficient Path-Space Differentiable Volume Rendering With Respect To Shapes, https://www.shuangz.com/projects/psdr-vol2-egsr23/psdr-vol2-egsr23.pdf, fetched 2026-10-27
