# Dynamic Diffuse Global Illumination (DDGI)

## Overview
Dynamic Diffuse Global Illumination (DDGI) is a real-time global illumination technique that blends hardware-accelerated ray tracing with traditional rasterization to compute indirect diffuse lighting [S1, unofficial]. It extends classic irradiance probes by compactly encoding the full irradiance field in a scene using a 3D grid of very low-resolution cubemaps, augmented with a second depth cubemap to prevent light and shadow leaks [S2, unofficial]. Unlike its sibling [Hardware Ray Tracing Global Illumination](3648-hardware-ray-tracing-global-illumination.md), which performs one-bounce tracing per frame, DDGI maintains persistent probe data that adapts to geometry changes over time, offering $O(1)$ rendering complexity at the cost of slightly lower quality than Voxel Cone Tracing (VCT) and no specular support [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry (for rasterization), scene lighting (direct), and a 3D grid of active probes.
*   **Output:** Indirect diffuse irradiance values stored in the probe atlas, blended into the final shading.

**Data Structures:**
1.  **Probe Atlas:** A 3D grid array of cubemaps. Each probe is a very low-resolution cubemap, typically $4 \times 4$ texels [S2, unofficial].
2.  **Depth Cubemap:** A second cubemap per probe storing distance to the probe's center (e.g., in the alpha channel) to mitigate light/shadow leaks [S2, unofficial].
3.  **Surfel G-Buffer:** A structure storing position and normal data for geometry hits [S1, unofficial].

**Algorithm Steps:**
The method is often implemented in two stages depending on hardware availability: Stage 1 uses Voxel Cone Tracing (VCT) to build probes, and Stage 2 uses hardware-accelerated ray tracing (e.g., RTX/DXR or RadeonRays) [S2, unofficial]. The core update loop for a frame is as follows:

1.  **Ray Generation:** From each of the $m$ active probes, generate and trace $n$ rays [S1, unofficial].
2.  **Hit Collection:** Store geometry hits for up to $m \times n$ surfaces in a G-Buffer structure. Each hit is recorded as a "surfel" containing position and normal data [S1, unofficial].
3.  **Surfel Shading:** Shade each surfel with direct illumination and any existing indirect illumination [S1, unofficial].
4.  **Probe Update:** Update the texels in the active probes by blending in the updated shading, distance, and square-distance results for each intersected surfel [S1, unofficial].
5.  **Blending and Interpolation:** Blend each frame's updated raytraced illumination into the probe atlas. Interpolate probe depth information to allow probe data to adapt to changes in scene geometry [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_ddgi_probes(probe_grid, scene_geometry, direct_light):
    """
    Updates DDGI probes for the current frame.
    probe_grid: List of active probes, each containing color and depth cubemaps.
    scene_geometry: Accessible geometry for ray intersection.
    direct_light: Direct illumination contribution.
    """
    # Step 1: Generate and trace rays from active probes
    # m = number of active probes, n = number of rays per probe
    m = len(probe_grid)
    n = 8  # Example ray count, not specified in sources
    
    surfels = []
    
    for probe in probe_grid:
        for i in range(n):
            # Generate ray direction (e.g., cosine-weighted hemisphere or uniform)
            ray_origin = probe.position
            ray_direction = generate_ray_direction(probe.normal, i)
            
            # Trace ray against scene geometry
            hit = trace_ray(ray_origin, ray_direction, scene_geometry)
            
            if hit.is_valid:
                # Step 2: Store surfel data
                surfel = {
                    'position': hit.point,
                    'normal': hit.normal,
                    'distance': hit.distance,
                    'probe_index': probe_grid.index(probe)
                }
                surfels.append(surfel)

    # Step 3: Shade surfels
    for surfel in surfels:
        # Calculate direct illumination
        direct_illuminance = compute_direct_light(surfel.position, surfel.normal, direct_light)
        # Indirect illumination is approximated by the probe itself or previous frame
        indirect_illuminance = get_probe_indirect(surfel.probe_index, surfel.position)
        surfel.shading = direct_illuminance + indirect_illuminance

    # Step 4 & 5: Update probes and blend
    for surfel in surfels:
        probe = probe_grid[surfel.probe_index]
        
        # Blend shading, distance, and square-distance into probe texels
        # The specific blending function (e.g., exponential decay based on distance)
        # is not detailed in sources [G: Standard irradiance probe update logic]
        update_probe_texel(probe, surfel.shading, surfel.distance, surfel.square_distance)
        
        # Update depth cubemap to prevent leaks
        update_probe_depth(probe, surfel.position, surfel.distance)

    # Interpolate probe data to adapt to geometry changes
    interpolate_probe_grid(probe_grid)
```

**Parameters:**
*   **Probe Resolution:** Very low, e.g., $4 \times 4$ [S2, unofficial].
*   **Active Probes:** All probes in the grid are active by default [S1, unofficial]. Heuristics to select a subset can be implemented to reduce updates [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Light/Shadow Leaks:** Mitigated by the second depth cubemap storing distance to the probe's center [S2, unofficial].
*   **Probe Selection:** The base method does not offer heuristics for choosing active probes, meaning all are updated every frame, which can be computationally expensive if the grid is large [S1, unofficial].
*   **Specular Limitations:** DDGI handles only diffuse GI; it does not support specular reflections [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ during rendering [S2, unofficial].
*   **Memory Complexity:** Consumes far less memory than Voxel Cone Tracing (VCT) due to the low-resolution cubemaps [S2, unofficial].
*   **Performance:** Real-time capable, especially when built using hardware-accelerated ray tracing (RTX/DXR) [S2, unofficial].
*   **Quality:** Slightly lower quality than diffuse VCT [S2, unofficial].
*   **Strengths:** No light/shadow leaks (with depth cubemap), low memory footprint, real-time performance.
*   **Limitations:** Diffuse only, no specular reflections, potential performance cost if all probes are active without heuristics.
*   **When to use:** Real-time applications requiring diffuse GI with strict memory constraints and hardware ray tracing support.
*   **When not to use:** When specular global illumination is required or when VCT quality is preferred and memory is not a constraint.

## Background
DDGI was presented by McGuire et al. [S1, unofficial]. It extends prior work from McGuire et al. [41] (cited in [S1, unofficial]) by extending classic irradiance probes to compactly encode the full irradiance field. A primary concern in its design was to eliminate the light and shadow leak issues present in other global illumination approaches [S1, unofficial]. The technique is often implemented in two stages: first using VCT to build probes (allowing development without ray tracing hardware), and later switching to hardware-accelerated ray tracing (RTX/DXR or RadeonRays) for performance [S2, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Voxel Cone Tracing | OGRE - Open Source 3D Graphics Engine, https://www.ogre3d.org/2019/08/05/voxel-cone-tracing, fetched 2026-10-27 (unofficial)
