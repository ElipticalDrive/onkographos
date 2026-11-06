# Precomputed Lighting (Baking)

Also called: Baking Lighting, Baking the lighting, Direct Lighting Caching, Precomputed Light Transmissivity, Precomputed Transfer Function

## Overview
Precomputed Lighting (Baking) is a technique to accelerate volumetric rendering by pre-computing the lighting term ($L_i$) and storing it in the grid's voxels, allowing the renderer to skip evaluating $L_i$ at each raymarching step on the camera ray [S1, S2, S3, unofficial]. It approximates volumetric shadows by precalculating illumination for each voxel center, enabling trilinear interpolation at render time [S2, unofficial]. Unlike [Voxel Cone Tracing](1454-voxel-cone-tracing.md) which computes lighting dynamically via cone tracing, or [Spherical Harmonic Volumetric Visibility](1225-spherical-harmonic-volumetric-visibility.md) which compresses directional data, this method stores the final irradiance value directly, making it view-independent but scene and light source dependent [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid representing the participating medium (density field) and a set of light sources.
*   **Output:** A 3D texture or grid storing the pre-computed lighting term ($L_i$) for each voxel.

**Data Structures:**
The lighting data is stored in a 3D texture or voxel grid. The layout typically follows memory order (e.g., x, then y, then z) to optimize cache coherence during the baking pass [S1, unofficial]. The resolution of this cache can be lower than the actual rendering voxel resolution because light is inherently blurry [S3, unofficial].

**Steps:**
1.  **Baking Pass:** Perform a pre-pass to calculate the direct lighting term ($L_i$) for each voxel in the grid. This involves evaluating the illumination at each voxel center, often by casting rays to light sources or using shadow maps [S1, S2, S3, unofficial].
2.  **Storage:** Store the computed $L_i$ values in the voxel grid.
3.  **Rendering Pass:** During the main raymarching pass, instead of evaluating $L_i$ for every sample, read the pre-computed $L_i$ data from the voxels.
4.  **Interpolation:** Use trilinear interpolation to sample the baked lighting data if the raymarching step falls between voxel centers [S2, unofficial].
5.  **Hybrid Sampling (Optional):** In some implementations, perform a few real light sample steps (e.g., ~2 steps) to capture high-frequency details or dynamic effects, and then sample the cached direct lighting value for the remaining steps [S3, unofficial].

**Formulas:**
The radiative transfer equation is simplified by replacing the integral for incoming radiance $L_i$ with a lookup:
$$ L_{sample} = L_{baked}(v) \cdot T $$
Where $L_{baked}(v)$ is the pre-computed lighting at voxel $v$, and $T$ is the transmittance accumulated along the ray up to that point. The exact formula for $L_{baked}$ depends on the light model used in the baking pass (e.g., direct illumination from point/spot lights) [S1, S2, unofficial].

**Parameters:**
*   **Cache Resolution:** Can be lower than the actual voxel resolution [S3, unofficial]. Not given in the sources.
*   **Baking Frequency:** Must be restarted whenever the lighting or the simulation changes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Dynamic Scenes:** The technique requires re-baking whenever the lighting or the simulation changes, which can be computationally expensive [S1, unofficial].
*   **Memory Usage:** Storing full lighting data per voxel increases memory consumption [S1, unofficial].
*   **Blurriness:** Since light is blurry, lower resolution caches are acceptable, but aggressive downsampling may lose shadow detail [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Baking Pass
def bake_lighting(voxel_grid, lights):
    # Initialize baked lighting grid
    baked_lighting = create_3d_texture(voxel_grid.resolution)
    
    # Iterate over voxels in memory order (x, y, z) for cache efficiency [S1, unofficial]
    for z in range(voxel_grid.resolution.z):
        for y in range(voxel_grid.resolution.y):
            for x in range(voxel_grid.resolution.x):
                voxel_center = voxel_grid.get_center(x, y, z)
                density = voxel_grid.get_density(x, y, z)
                
                if density > 0:
                    # Calculate direct lighting for this voxel
                    # This may involve casting rays to lights or using shadow maps [S2, unofficial]
                    li = compute_direct_lighting(voxel_center, lights, voxel_grid)
                    baked_lighting.set_value(x, y, z, li)
                else:
                    baked_lighting.set_value(x, y, z, vec3(0.0))
    
    return baked_lighting

# Rendering Pass
def render_with_baked_lighting(voxel_grid, baked_lighting, camera, lights):
    # Raymarching loop
    while ray_t < max_distance:
        voxel_pos = ray_t_to_voxel(ray_t)
        
        # Sample baked lighting instead of computing it [S1, unofficial]
        li = baked_lighting.trilinear_sample(voxel_pos)
        
        # Optional: Hybrid approach [S3, unofficial]
        # If near start of ray, compute real lighting for first few steps
        if ray_t < early_step_threshold:
            li = compute_direct_lighting(ray_t, lights, voxel_grid)
        
        # Accumulate radiance
        density = voxel_grid.get_density(voxel_pos)
        transmittance = compute_transmittance(ray_t)
        radiance += li * density * transmittance * step_size
        
        ray_t += step_size
```

## Subtopics

### [Hybrid Preshading with Postclassification](2171-hybrid-preshading-with-postclassification.md)
Hybrid Preshading with Postclassification is a real-time volume rendering optimization that decouples classification from shading to balance visual fidelity and performance. Unlike pure preclassification/preshading which risks artifacts without opacity-weighted interpolation, or pure postclassification/postshading which is computationally expensive, this method performs classification after interpolation (postclassification) but shading before interpolation (preshading). It builds on [Precomputed Lighting (Baking)](2169-precomputed-lighting-baking.md) by precomputing lighting contributions for a discrete set of surface normals, allowing the renderer to apply shading via texture lookup during the raymarching step while retaining the visual benefits of post-classification.

### [Pre-computation](2330-pre-computation.md)
Pre-computation is a technique used in volume rendering to accelerate the evaluation of light transport in participating media by calculating and storing intermediate radiance variables as voxel data. It solves the problem of redundant per-voxel integration during ray marching by pre-calculating terms such as direct radiance ($L_{ri}$), diffusion radiance ($L_{0d}, \vec{L}_{1d}$), and their density derivatives, allowing the renderer to reuse these values instead of recomputing them for every camera ray. Unlike [Hybrid Preshading with Postclassification](2171-hybrid-preshading-with-postclassification.md), which may handle post-classification differently, this method explicitly pre-computes the diffusion components via a solver to enable efficient derivative-based shading.

## Operating characteristics
*   **Time Complexity:** Baking pass is $O(N)$ where $N$ is the number of voxels, with a high constant factor due to lighting calculations per voxel. Rendering pass is $O(M)$ where $M$ is the number of samples per ray, with a low constant factor due to texture lookup.
*   **Memory Complexity:** $O(N)$ for storing the baked lighting grid.
*   **Measured Performance:** Speeds up rendering by skipping lighting evaluation per sample [S1, unofficial]. However, the baking pass itself can take a long time [S1, unofficial].
*   **Scaling:** Performance scales with voxel resolution and number of light sources during baking. Rendering performance scales with raymarching steps.
*   **Strengths:** Fast rendering once baked; simple implementation [S1, unofficial].
*   **Limitations:** View-independent; requires re-baking for dynamic scenes or lights; high memory usage for high-resolution grids [S1, S2, unofficial].
*   **When to use:** Static scenes with static lighting where rendering performance is critical [S1, unofficial].
*   **When not to use:** Dynamic scenes with moving lights or changing geometry [S1, unofficial].

## Background
This technique is mentioned for reference and historical reasons rather than current practical utility [S1, unofficial]. It is related to the concept of deep shadow maps, which store volume density as a function of distance from the light, similar to baking lighting into voxels [S1, unofficial]. The idea of precomputing light transmissivity and interpolating at render time is a classic approach in volumetric visualization [S2, unofficial].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S3] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
