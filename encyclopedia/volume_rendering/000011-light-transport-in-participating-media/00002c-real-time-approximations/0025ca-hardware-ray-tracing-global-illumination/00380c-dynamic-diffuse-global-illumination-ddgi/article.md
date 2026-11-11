# Dynamic Diffuse Global Illumination (DDGI)

## Overview
Dynamic Diffuse Global Illumination (DDGI) is a real-time global illumination technique that computes one-bounce diffuse indirect lighting by maintaining a 3D grid of low-resolution cubemap probes [S2, unofficial]. It extends classic irradiance probes by blending hardware-accelerated ray tracing with rasterization to compactly encode the full irradiance field, specifically aiming to eliminate the light and shadow leak issues inherent in voxel-based methods like Voxel Cone Tracing (VCT) [S1, unofficial]. Unlike its parent [Hardware Ray Tracing Global Illumination](3648-hardware-ray-tracing-global-illumination.md), which traces rays per-pixel, DDGI traces rays per-probe to build a spatially coherent lighting field, offering $O(1)$ rendering complexity and significantly lower memory consumption at the cost of slightly lower visual quality compared to VCT [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry (for surfel generation), active probe locations, ray tracing acceleration structure (BVH).
*   **Output:** A probe atlas containing irradiance data (color) and depth information for each probe, used during shading to interpolate indirect lighting.

**Data Structures:**
*   **Probe Atlas:** A 3D grid array of cubemaps. Each probe consists of two very low-resolution cubemaps (e.g., $4 \times 4$ resolution) [S2, unofficial].
    *   **Irradiance Cubemap:** Stores the accumulated indirect diffuse lighting.
    *   **Depth Cubmap:** Stores the distance to the nearest surface hit from the probe center, used to prevent light/shadow leaks [S2, unofficial].
*   **Surfel G-Buffer:** A structure storing position and normal data for geometry hits intersected by probe rays [S1, unofficial].

**Algorithm Steps:**
The method operates in two main phases: Probe Building and Shading.

1.  **Ray Tracing and Surfel Generation:**
    For each of the $m$ active probes, generate and trace $n$ rays into the scene [S1, unofficial]. The rays can be generated via cone tracing (if hardware RT is unavailable) or hardware-accelerated ray tracing (RTX/DXR/RadeonRays) [S2, unofficial]. Store all geometry hits in a G-Buffer structure as surfels, recording their position and normal vectors [S1, unofficial].

2.  **Surfel Shading:**
    Shade each surfel with direct illumination (sun, sky) and indirect illumination [S1, unofficial]. This step calculates the radiance contribution of the hit surface.

3.  **Probe Update (Blending):**
    Update the texels in the active probes by blending the results from the intersected surfels [S1, unofficial]. For each surfel hit by a ray from a probe:
    *   Calculate the distance $d$ and squared distance $d^2$ from the probe center to the surfel.
    *   Blend the surfel's shading, distance, and square-distance results into the corresponding texel of the probe's irradiance and depth cubemaps [S1, unofficial].
    *   This blending process allows the probe data to adapt to changes in scene geometry over time [S1, unofficial].

4.  **Interpolation:**
    During rendering, interpolate probe depth information and irradiance data from the grid to the shading point to determine the indirect illumination [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_ddgi_probes(probes, scene_bvh, num_rays_per_probe):
    """
    Updates the DDGI probe atlas by tracing rays and blending surfel data.
    """
    for probe in probes:
        # 1. Generate and trace rays
        hits = []
        for i in range(num_rays_per_probe):
            ray_direction = cosine_sample_hemisphere(probe.normal)
            ray_origin = probe.position
            hit = trace_ray(ray_origin, ray_direction, scene_bvh)
            if hit.is_valid():
                hits.append(Surfel(
                    position=hit.point,
                    normal=hit.normal,
                    distance=distance(probe.position, hit.point)
                ))
        
        # 2. Shade surfels (Direct + Indirect)
        # Note: Indirect here refers to the contribution from the surfel itself
        # which is then blended into the probe.
        for surfel in hits:
            direct_light = compute_direct_irradiance(surfel.position, surfel.normal)
            # Indirect contribution from this specific hit is calculated based on
            # the surfel's material and direct lighting, then blended into probe.
            surfel_radiance = direct_light * surfel.material.albedo
            
            # 3. Blend into probe atlas
            # Map hit direction to cubemap face and texel
            face, uv = world_to_cubemap(surfel.position - probe.position)
            
            # Blend irradiance
            probe.irradiance_cubemap[face][uv] = blend(
                probe.irradiance_cubemap[face][uv],
                surfel_radiance,
                weight=surfel.distance # Distance-based weighting
            )
            
            # Blend depth (to prevent leaks)
            probe.depth_cubemap[face][uv] = blend_depth(
                probe.depth_cubemap[face][uv],
                surfel.distance
            )

def shade_with_ddgi(probe_grid, shading_point):
    """
    Retrieves indirect illumination for a shading point by interpolating probes.
    """
    # Find surrounding probes in the 3D grid
    surrounding_probes = get_surrounding_probes(probe_grid, shading_point)
    
    # Interpolate irradiance and depth from surrounding probes
    irradiance = interpolate_cubemaps(surrounding_probes, shading_point)
    depth = interpolate_depths(surrounding_probes, shading_point)
    
    # Use depth to mask out invalid or leaking samples if necessary
    # [G] Specific masking logic depends on implementation details not fully specified in sources.
    
    return irradiance
```

**Parameters:**
*   **Probe Resolution:** Very low, e.g., $4 \times 4$ per cubemap face [S2, unofficial].
*   **Active Probes:** All probes in the grid are active by default; no heuristics are provided for selecting a subset, though heuristics could be implemented to reduce updates [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Light/Shadow Leaks:** Mitigated by the second depth cubemap, which stores distance to the nearest surface to prevent light from leaking through geometry [S2, unofficial].
*   **Performance:** Without heuristics for active probes, all probes are updated every frame, which can be computationally expensive if the grid is large [S1, unofficial].
*   **Diffuse Only:** The method does not handle specular reflections [S2, unofficial].

## Subtopics

### [Probe-based Global Illumination (DDGI-like)](12753-probe-based-global-illumination-ddgi-like.md)
A probe-based technique that approximates indirect lighting using spherical harmonics coefficients in a 3D grid. It differs from DDGI by replacing depth-texture-based visibility with ray-traced visibility to eliminate aliasing and reduce memory overhead, while retaining the core structure of low-resolution cubemaps or spherical harmonics probes.

## Operating characteristics
*   **Complexity:** Rendering complexity is $O(1)$, as shading requires only interpolation from the probe grid [S2, unofficial]. Probe building complexity depends on the number of probes $m$ and rays $n$, but is performed asynchronously or in a separate pass.
*   **Memory:** Consumes far less memory than Voxel Cone Tracing (VCT) due to the use of very low-resolution cubemaps instead of a dense voxel grid [S2, unofficial].
*   **Performance:** Ideal for real-time applications due to $O(1)$ shading cost. Building probes can be accelerated significantly using hardware ray tracing (RTX/DXR) compared to rasterization or cone tracing [S2, unofficial].
*   **Quality:** Handles only diffuse GI. Visual quality is slightly lower than diffuse VCT [S2, unofficial].
*   **Limitations:** No specular reflections. Lack of heuristics for active probe selection may lead to unnecessary updates [S1, unofficial].

## Background
DDGI was presented by McGuire et al. [S1, unofficial]. It builds upon prior work by McGuire et al. [41] (cited in [S1, unofficial]) and extends classic irradiance probes to compactly encode the full irradiance field. The technique was developed to address performance issues with sparse octrees and light/shadow leaks in previous global illumination approaches [S1, unofficial]. It is often implemented in two stages: first using Voxel Cone Tracing (VCT) to build probes (without hardware RT), and later migrating to hardware-accelerated ray tracing (RTX/DXR/RadeonRays) for faster and more accurate probe building [S2, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Voxel Cone Tracing | OGRE - Open Source 3D Graphics Engine, https://www.ogre3d.org/2019/08/05/voxel-cone-tracing, fetched 2026-10-27 (unofficial)
