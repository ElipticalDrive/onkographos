# Adaptive Beam Tracing for Volume Caustics

Also called: Adaptive beam tracing, Real-time volume caustics with adaptive beam tracing

## Overview
Adaptive Beam Tracing is a real-time algorithm for rendering surface and volume caustics in single-scattering participating media [S1]. It solves the problem of efficiently capturing high-frequency light transport patterns formed by specular reflections and refractions without the prohibitive cost of path tracing or the static limitations of precomputed radiance transfer [S1]. Unlike sibling methods that use suboptimal bounding volumes or non-adaptive grids, this approach adaptively refines a projected grid based on geometric discontinuities and beam contribution, generating tight bounding volumes to minimize fill-rate while supporting multiple bounces and two-sided refraction [S1].

## How it works
The method operates in three main phases: adaptive grid generation, beam tracing with splitting, and inscattering accumulation.

### Inputs and Outputs
*   **Inputs:** Scene geometry (specular surfaces), participating media properties ($\sigma_a, \sigma_s, \sigma_t$), light source properties (position, intensity, type), camera parameters.
*   **Outputs:** Final image with volume and surface caustics.

### Data Structures
1.  **Light G-Buffer:** Stores position, normal, and material properties (transparency, index of refraction) of surfaces directly visible from the light source [S1].
2.  **Caustic Triangle Buffer:** Stores the base triangles of the caustic beams. Each triangle is defined by vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$ [S1].
3.  **Caustic Beam:** Defined by a base triangle and three edges. The edges are defined by the reflection or refraction direction $\mathbf{d}_i$ at each vertex $\mathbf{v}_i$. The beam is bounded by rays $\mathbf{v}_i + t_i \mathbf{d}_i$ for $i \in \{0, 1, 2\}$ [S1]. Two of these edges form bilinear patches [S1].
4.  **Bounding Volume:** A 6-triangle mesh approximating the beam volume, generated to be tight around the warped/diverging/converging beam [S1].

### Algorithm Steps

#### 1. Adaptive Grid Generation and Refinement
1.  **Rasterize Light G-Buffer:** Render specular surfaces from the light source's view into off-screen geometry buffers [S1].
2.  **Initial Grid Projection:** Project a coarse 2D grid onto the sampled geometry to generate initial caustic triangles [S1].
3.  **Adaptive Refinement:** Iteratively refine the grid based on an oracle heuristic [S1]:
    *   Detect undersampled regions in the grid [S1].
    *   Split corresponding grid triangles into smaller ones [S1].
    *   The oracle decides whether to keep, subdivide, or discard a beam based on three factors: geometric discontinuities of specular objects, light intensity (for spot/textured lights), and the beam's contribution in the camera image [S1].
    *   *Note:* The specific heuristic values are empirically set [S1].

#### 2. Beam Tracing and Splitting (Multi-Bounce)
1.  **Beam Generation:** For each triangle in the refined grid, compute the reflection/refraction directions $\mathbf{d}_0, \mathbf{d}_1, \mathbf{d}_2$ at the vertices to form the caustic beam [S1].
2.  **Bounding Volume Computation:** Determine the extrusion length of the beam by intersecting with the view frustum and handling occlusion [S1]. Generate a tight 6-triangle bounding volume for rasterization [S1].
3.  **Storage:** Store primary beams for later passes to handle occlusion and higher-order caustics [S1].
4.  **Multi-Bounce Splitting:**
    *   Rasterize the scene into impostors using a representative center of projection [S1].
    *   Intersect stored beams with surrounding scene geometry using approximate ray tracing [S1].
    *   Detect depth discontinuities by comparing photon hits at beam edges and surface normals [S1].
    *   Split secondary beams at silhouettes where intersections occur [S1].

#### 3. Inscattering Accumulation
1.  **Rasterize Bounding Volumes:** Rasterize the tight bounding volumes of the caustic beams [S1].
2.  **Fragment Shader Processing:** For each fragment covered by a beam:
    *   **Beam Intersection:** Intersect the camera ray with the actual beam (not just the bounding volume) to find the intersection interval $\Delta x$ [S1]. This is done by transforming the view ray to the beam's local space and evaluating the cross-section area function $A(y)$ at the midpoint of $\Delta x$ [S1].
    *   **Radiance Estimation:** Compute the scattered radiance using the approximation:
        $$ L(\omega) \approx \Delta x e^{-\sigma_t (s_1 + s_2)} \sigma_s p(\omega \cdot \omega') L_{in}(\omega') $$
        where:
        *   $L(\omega)$ is the radiance scattered towards the viewer [S1].
        *   $\Delta x$ is the length of the ray segment inside the beam, obtained via a cheap 2D ray-triangle edge intersection [S1].
        *   $\sigma_t$ is the extinction coefficient [S1].
        *   $s_1 + s_2$ is the total light path length from the caustic triangle to the camera [S1].
        *   $\sigma_s$ is the scattering coefficient [S1].
        *   $p(\cdot)$ is the phase function [S1].
        *   $L_{in}(\omega')$ is the incident radiance at the sample point [S1].
        *   $\omega$ is the viewing direction, $\omega'$ is the incident direction [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_beam_tracing(scene, light, camera, media):
    # 1. Light Space Rasterization
    light_gbuffer = rasterize_light_space(scene.specular_surfaces, light)
    
    # 2. Adaptive Grid Generation
    grid = CoarseGrid()
    caustic_triangles = []
    
    # Initial projection
    caustic_triangles = project_grid_to_geometry(grid, light_gbuffer)
    
    # Adaptive Refinement Loop
    while not grid.converged():
        # Detect undersampled regions
        regions = detect_undersampled(caustic_triangles, light_gbuffer, camera)
        
        # Split triangles in undersampled regions
        new_triangles = split_triangles(regions, caustic_triangles)
        
        # Oracle decision
        for tri in new_triangles:
            if oracle_should_keep(tri, light, camera):
                caustic_triangles.append(tri)
            elif oracle_should_subdivide(tri):
                # Further subdivision logic
                pass
            else:
                # Discard
                pass
        grid.refine()

    # 3. Beam Tracing and Bounding Volumes
    beams = []
    for tri in caustic_triangles:
        # Compute reflection/refraction directions at vertices
        d0, d1, d2 = compute_refraction_directions(tri.vertices, light, scene)
        
        # Create beam structure
        beam = Beam(base=tri, edges=[d0, d1, d2])
        
        # Compute tight bounding volume
        beam.bounding_volume = compute_tight_bounding_volume(beam, camera, scene)
        
        beams.append(beam)

    # 4. Multi-Bounce Handling (Simplified)
    # Rasterize scene into impostors for depth discontinuity detection
    impostors = rasterize_impostors(scene)
    
    for beam in beams:
        # Intersect with scene geometry
        intersections = approximate_ray_trace(beam, impostors)
        
        if intersections:
            # Split beam at silhouettes
            split_beams = split_beam_at_intersection(beam, intersections)
            beams.extend(split_beams)

    # 5. Inscattering Accumulation
    image = zeros(camera.resolution)
    
    for beam in beams:
        # Rasterize bounding volume
        fragments = rasterize(beam.bounding_volume, camera)
        
        for frag in fragments:
            # Intersect camera ray with actual beam
            ray = camera.get_ray(frag.uv)
            intersection = intersect_beam_with_ray(beam, ray)
            
            if intersection.exists():
                # Compute interval length Delta x
                delta_x = compute_intersection_length(intersection)
                
                # Compute path lengths s1 + s2
                s1 = distance(beam.base, intersection.point)
                s2 = distance(intersection.point, camera.position)
                
                # Estimate radiance
                incident_radiance = beam.get_radiance_at(intersection.point)
                scattered_radiance = (delta_x * 
                                      exp(-media.sigma_t * (s1 + s2)) * 
                                      media.sigma_s * 
                                      phase_function(ray.direction, incident_radiance.direction) * 
                                      incident_radiance.value)
                
                image[frag.uv] += scattered_radiance

    return image

def intersect_beam_with_ray(beam, ray):
    # Transform ray to beam local space
    local_ray = transform_to_beam_local(ray, beam)
    
    # Evaluate cross-section area function A(y) at midpoint
    # This determines if the ray passes through the beam
    midpoint = ray.origin + 0.5 * ray.direction * distance_to_beam_plane
    area = evaluate_cross_section(midpoint, beam)
    
    if area > 0:
        # Compute 2D ray-triangle edge intersection for Delta x
        delta_x = compute_2d_edge_intersection(ray, beam.base)
        return Intersection(point=midpoint, length=delta_x)
    return None
```

### Parameters
*   **Grid Resolution:** Starts coarse, refined adaptively. Not given in the sources.
*   **Extrusion Length:** Determined by view frustum intersection and occlusion handling [S1].
*   **Heuristic Factors:** Geometric discontinuities, light intensity, beam contribution [S1].

### Edge Cases and Pitfalls
*   **Fill Rate:** Suboptimal bounding prisms consume vast amounts of fill-rate [S1]. The tight 6-triangle bounding volume mitigates this [S1].
*   **Blocky Artifacts:** Constant shading with too few beams results in blocky artifacts [S1]. Adaptive refinement prevents this by increasing resolution in critical areas [S1].
*   **Undersampling:** Tracing photons and rendering line primitives can suffer from undersampling in highly divergent regions [S1]. Beam tracing with adaptive grids avoids this [S1].
*   **Beam Splitting:** Splitting beams where a surface is partially intersected is non-trivial. The method uses scene geometry impostors and depth discontinuity detection to handle this [S1].

## Subtopics

### Adaptive Refinement Heuristic
A method to decide whether a caustic beam should be kept, subdivided, or discarded. It detects undersampled regions in the grid and splits corresponding triangles. The decision is based on an empirically set-up heuristic depending on three factors: geometric discontinuities of specular objects, light intensity (for spot/textured lights), and the beam's contribution in the camera image [S1].

### Multi-Bounce Beam Splitting via Depth Discontinuity Detection
A method to simulate higher-order light bounces by detecting intersections of caustic beams with other specular surfaces. It rasterizes the scene into impostors and compares photon hits at beam edges and surface normals to detect depth discontinuities, allowing splitting of secondary beams at silhouettes [S1].

### Inscattering and Surface Caustics
A fragment shader technique to compute radiance scattered towards the viewer. It approximates the integral with a single sample assuming beams are narrow enough in image space: $L(\omega) \approx \Delta x e^{-\sigma_t(s_1+s_2)} \sigma_s p(\omega \cdot \omega') L_{in}(\omega')$. It intersects the beam (not the bounding volume) with the viewing ray and computes the incident radiance and scattered fraction [S1].

## Operating characteristics
*   **Performance:** Renders at real-time frame rates (27 FPS, 23 FPS, 26 FPS shown in tests) [S1].
*   **Complexity:** Fill-rate is minimized by tight bounding volumes (6-triangle method) compared to bounding prisms (38 FPS vs 32 FPS in tests) [S1].
*   **Strengths:** Handles two-sided refractions, inhomogeneous participating media, and multi-bounce light-surface interactions without precomputation [S1]. Adaptive generation reduces sampling resolution and fill-rate at defocused regions [S1].
*   **Limitations:** Single-scattering approximation [S1]. Requires approximate ray tracing for beam splitting [S1].

## Background
The method was presented by Gábor Liktor and Carsten Dachsbacher at SIGGRAPH 2011 [S1]. It builds on adaptive beam tracing [Heckbert and Hanrahan 1984] and caustic mapping [S1]. It improves upon Ernst et al. [2005] by computing tighter bounding volumes and supporting multiple bounces, and upon Liktor and Dachsbacher [2010] by making beam generation adaptive [S1].

## Sources
[S1] Real-time volume caustics with adaptive beam tracing, https://doi.org/10.1145/1944745.1944753, fetched 2026-10-28
