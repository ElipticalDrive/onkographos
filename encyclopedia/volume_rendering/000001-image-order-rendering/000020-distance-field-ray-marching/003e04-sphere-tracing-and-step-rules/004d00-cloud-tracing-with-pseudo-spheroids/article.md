# Cloud Tracing with Pseudo-Spheroids

Also called: Apodaca’s code (variant)

## Overview
Cloud Tracing with Pseudo-Spheroids renders volumetric clouds by tracing rays through a set of implicit surfaces (pseudo-spheroids or ellipsoids) and sampling noise only when the ray traverses these volumes [S1]. It differs from standard sphere tracing by using a list of randomized bounded-surface volumes rather than a single global Signed Distance Function (SDF), optimizing performance by discarding positions outside the cloud and using a collision detection method based on quadratic discriminants [S1].

## How it works
**Inputs:**
*   Camera origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized) for each pixel.
*   List of $N$ pseudo-spheroids, each defined by center $\mathbf{c}_i$, nominal radius $R_{nominal}$, distortion parameter $k$, and maximum density $\rho_{max}$.
*   3D noise function $fBm(\mathbf{p})$ (fractional Brownian motion), normalized to $[0, 1]$.
*   Density threshold $\rho_{thresh}$ for termination.
*   Bounding box for spatial culling.

**Outputs:**
*   Final color $R$ for each pixel, blended with the sky background.

**Data Structures:**
*   `candidates`: A list of indices of pseudo-spheroids intersected by the ray.
*   `sorted_candidates`: The `candidates` list sorted by intersection distance (back-to-front).

**Formulas:**
1.  **Ray-Sphere Intersection:** For a ray $r(t) = \mathbf{o} + t\mathbf{d}$ and sphere $i$ with center $\mathbf{c}_i$ and effective radius $R_i$, the intersection is determined by the discriminant $\Delta$ of the quadratic equation $|\mathbf{o} + t\mathbf{d} - \mathbf{c}_i|^2 = R_i^2$.
    $$ \Delta = (\mathbf{d} \cdot (\mathbf{o} - \mathbf{c}_i))^2 - (|\mathbf{o} - \mathbf{c}_i|^2 - R_i^2) $$
    If $\Delta \ge 0$, the ray intersects the sphere. The entry and exit distances are:
    $$ t_{entry, exit} = -(\mathbf{d} \cdot (\mathbf{o} - \mathbf{c}_i)) \pm \sqrt{\Delta} $$
2.  **Effective Radius:** The nominal radius is modulated by a random component $k$ to create a pseudo-spheroid shape [S1]:
    $$ R_i = R_{nominal} \pm k $$
    *(Note: The source describes scaling the nominal sphere radius by $\pm k$ using Euclidean distance, effectively creating a distorted boundary.)*
3.  **Density Modulation:** The density $\rho$ at a point $\mathbf{p}$ inside the pseudo-spheroid is calculated as [S1]:
    $$ \rho(\mathbf{p}) = \rho_{max} \cdot fBm(\mathbf{p}) \cdot \left(1 - \frac{||\mathbf{p} - \mathbf{c}_i||}{R_i}\right) $$
    *(Inferred from Equation (2) description: Density is modulated by position inside the pseudo-spheroid with a random component. The term $(1 - \frac{||\mathbf{p} - \mathbf{c}_i||}{R_i})$ ensures density drops to zero at the boundary.)*
4.  **Level of Detail (LOD) Step Size:** The step increment $\Delta t$ is controlled by the distance from the cloud center to balance detail and performance [S1]:
    $$ \Delta t = f(\text{distance from center}) $$
    Longer steps are taken when close to the camera (or far from center?), and smaller steps when far. *(Note: The source states "longer steps when close to camera, smaller steps when far" in the context of distance from the center of the cloud. This implies an inverse relationship or a specific LOD function not fully detailed in the text.)*

**Algorithm Steps:**
1.  **Culling:** For each pixel, identify pseudo-spheroids within the bounding box.
2.  **Collision Detection:** For each candidate sphere, compute the discriminant $\Delta$. If $\Delta \ge 0$, add the sphere index to the `candidates` list.
3.  **Sorting:** Sort the `candidates` list by intersection distance using insertion sort, ensuring back-to-front processing [S1].
4.  **Ray Marching:** For each pixel:
    *   Initialize accumulated color $C = 0$ and opacity $\alpha = 0$.
    *   For each sphere in `sorted_candidates`:
        *   Calculate intersection interval $[t_{entry}, t_{exit}]$.
        *   If the ray does not intersect the sphere, skip.
        *   March from $t_{entry}$ to $t_{exit}$ with step size $\Delta t$ (LOD adjusted).
        *   At each step point $\mathbf{p}$:
            *   Calculate density $\rho(\mathbf{p})$ using the formula above.
            *   If $\rho(\mathbf{p}) < \rho_{thresh}$, continue.
            *   Compute lighting $L(\mathbf{p})$ (e.g., using normal derived from SDF gradient or noise gradient).
            *   Update color and opacity using volume rendering integration (e.g., standard front-to-back compositing, though the source mentions back-to-front tracing, standard alpha blending is implied for "Blend the result R with the sky").
            *   If $\alpha \ge 1.0$ (or sufficient opacity), break the marching loop for this sphere.
    *   Blend the accumulated cloud color $R$ with the sky background.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cloud_tracing(ray_origin, ray_direction, spheres, noise_func, rho_thresh):
    """
    Render clouds using pseudo-spheroids.
    Args:
        ray_origin: Camera origin (vec3)
        ray_direction: Normalized ray direction (vec3)
        spheres: List of dicts with keys: 'center', 'R_nominal', 'k', 'rho_max'
        noise_func: Function fBm(p) returning normalized noise [0,1]
        rho_thresh: Density threshold for termination
    Returns:
        color: Final pixel color (vec3)
    """
    candidates = []
    
    # 1. Culling and Collision Detection
    for i, sphere in enumerate(spheres):
        c = sphere['center']
        R = sphere['R_nominal'] + sphere['k'] # Effective radius
        
        oc = ray_origin - c
        a = dot(ray_direction, ray_direction)
        b = 2.0 * dot(oc, ray_direction)
        c_val = dot(oc, oc) - R * R
        
        discriminant = b * b - 4 * a * c_val
        
        if discriminant >= 0:
            # Ray intersects sphere
            sqrt_disc = sqrt(discriminant)
            t1 = (-b - sqrt_disc) / (2 * a)
            t2 = (-b + sqrt_disc) / (2 * a)
            candidates.append({'index': i, 't_entry': t1, 't_exit': t2})
            
    # 2. Sort Candidates (Back-to-Front)
    # Insertion sort as per source
    for i in range(1, len(candidates)):
        key = candidates[i]
        j = i - 1
        while j >= 0 and candidates[j]['t_entry'] > key['t_entry']:
            candidates[j + 1] = candidates[j]
            j -= 1
        candidates[j + 1] key
        
    # 3. Ray Marching and Integration
    accumulated_color = vec3(0.0)
    accumulated_alpha = 0.0
    
    for cand in candidates:
        t_start = max(cand['t_entry'], 0.0)
        t_end = cand['t_exit']
        
        if t_start >= t_end:
            continue
            
        # LOD Step Size: Simplified assumption based on distance
        # Source: "longer steps when close to camera, smaller steps when far"
        # We use a distance-based step size. 
        # Note: Exact LOD formula not provided in source [G]
        dist_to_center = length(ray_origin - spheres[cand['index']]['center'])
        step_size = 0.1 * (1.0 + dist_to_center) # Placeholder for LOD function [G]
        
        t = t_start
        while t < t_end:
            p = ray_origin + t * ray_direction
            sphere = spheres[cand['index']]
            c = sphere['center']
            R = sphere['R_nominal'] + sphere['k']
            
            # Density Calculation
            dist_to_c = length(p - c)
            if dist_to_c < R:
                noise_val = noise_func(p)
                density = sphere['rho_max'] * noise_val * (1.0 - dist_to_c / R)
                
                if density > rho_thresh:
                    # Lighting Calculation (Placeholder)
                    # Source mentions lighting(p) but no formula [G]
                    light = vec3(1.0) 
                    color = light * density
                    
                    # Volume Rendering (Front-to-Back Compositing)
                    # Note: Source says "trace from back to front" but standard
                    # compositing is front-to-back. We assume standard alpha blending.
                    alpha = density * step_size # Approximation
                    alpha = min(alpha, 1.0 - accumulated_alpha)
                    
                    accumulated_color += color * alpha
                    accumulated_alpha += alpha * (1.0 - alpha) # Simplified [G]
                    
                    if accumulated_alpha >= 0.99:
                        break
            t += step_size
            
        if accumulated_alpha >= 0.99:
            break
            
    # 4. Blend with Sky
    sky_color = vec3(0.5, 0.7, 1.0) # Placeholder
    final_color = accumulated_color + sky_color * (1.0 - accumulated_alpha)
    
    return final_color
```

**Edge Cases and Pitfalls:**
*   **Precision:** Discriminant calculation must handle floating-point errors near $\Delta = 0$.
*   **Boundary:** Density drops to zero at the effective radius $R_i$. Points outside this radius should not be sampled.
*   **LOD:** The exact LOD equation is not provided in the source. A distance-based heuristic is used as a placeholder [G].
*   **Sorting:** Insertion sort is efficient for small lists of candidates but may be slow for very large $N$.

## Subtopics

### Apodaca’s code
A ray-tracing algorithm over a list of $N$ randomized bounded-surface volumes (pseudo-spheroids) to render clouds, using collision detection and insertion sort for optimization [S1]. It differs from the general method by explicitly detailing the candidate selection and sorting steps. It is chosen when a simple, GPU-friendly implementation with spatial culling is required. Link: [Cloud Tracing with Pseudo-Spheroids](9643-cloud-tracing-with-pseudo-spheroids.md).

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of spheres and $M$ is the number of steps per sphere. The insertion sort adds $O(K^2)$ where $K$ is the number of intersected spheres.
*   **Memory Complexity:** $O(N)$ to store sphere data and $O(K)$ for candidates.
*   **Performance:** The method reduces computational effort by discarding positions outside the cloud and using a short list of intersected spheres [S1].
*   **Strengths:** Efficient culling, simple geometry (spheres/ellipsoids), good for real-time GPU rendering.
*   **Limitations:** Approximation of cloud shape (pseudo-spheroids), LOD step size is heuristic.
*   **When to use:** Real-time cloud rendering where performance is critical and high-fidelity global noise is not required.
*   **When not to use:** When complex, non-spherical cloud structures are needed.

## Background
Originated in the context of real-time GPU volumetric cloud rendering. The method is an improved version of Apodaca’s code [27] and is described in [S1]. It builds on the principle of using implicit surfaces to define cloud volumes and sampling noise only within those volumes.

## Sources
[S1] Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry, https://doi.org/10.3390/sym10040125, fetched 2026-10-26
