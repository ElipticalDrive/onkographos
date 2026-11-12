# Precomputed Lighting (Baking)

Also called: Baking Lighting, Baking the lighting, Direct Lighting Caching, Precomputed Light Transmissivity, Precomputed Transfer Function

## Overview
Precomputed Lighting (Baking) is a volume rendering optimization technique that accelerates rendering by pre-computing the incident radiance term ($L_i$) for each voxel in a grid and storing it in the data structure, thereby skipping the evaluation of $L_i$ during the per-pixel raymarching pass [S1, S2, S3, unofficial]. It solves the performance bottleneck of evaluating volumetric shadows or direct illumination for every sample point along a camera ray by trading memory and offline computation time for runtime speed [S1, S2, S3, unofficial]. Unlike [Modular Transfer](1249-modular-transfer.md), which decomposes the transfer function into separable components for flexible runtime lighting, Precomputed Lighting is static and view-independent, requiring a full re-bake whenever the scene geometry, lighting, or density simulation changes [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid representing the volume density/opacity, and a static lighting configuration (light positions, intensities, colors) and scene geometry.
*   **Output:** A 3D texture or data structure where each voxel stores the pre-computed incident radiance ($L_i$) or light transmissivity.

**Data Structures:**
The lighting data is stored in a 3D texture or voxel grid. To optimize cache performance during the baking pass, the data should be iterated and stored in memory order (e.g., x-fastest, z-slowest) to avoid cache misses [S1, unofficial]. The resolution of this cache can be lower than the actual simulation voxel resolution because light attenuation and illumination are inherently smooth and blurry [S3, unofficial].

**Steps:**
1.  **Baking Pass (Pre-computation):** Iterate through every voxel in the grid. For each voxel center, calculate the incident radiance ($L_i$) from all light sources. This typically involves casting rays from the voxel center to each light source to determine visibility (shadowing) and accumulating direct illumination [S1, S2, S3, unofficial].
2.  **Storage:** Store the calculated $L_i$ value in the corresponding voxel of the precomputed lighting grid [S1, S2, unofficial].
3.  **Rendering Pass:** During real-time volume rendering, when a raymarching step reaches a sample point, retrieve the pre-computed $L_i$ value by trilinearly interpolating the values from the surrounding voxels in the precomputed grid [S2, S3, unofficial].
4.  **Shading:** Use the interpolated $L_i$ value directly in the shading equation, skipping the expensive ray-casting to light sources for that sample [S1, S3, unofficial].

**Formulas:**
The incident radiance $L_i$ at a voxel center is precomputed as:
$$ L_i(\mathbf{x}) = \sum_{j} L_{e,j} \cdot T(\mathbf{x}, \mathbf{l}_j) \cdot \max(0, \mathbf{n} \cdot \mathbf{l}_j) $$
Where:
*   $\mathbf{x}$ is the voxel center position.
*   $L_{e,j}$ is the emitted radiance of light source $j$.
*   $T(\mathbf{x}, \mathbf{l}_j)$ is the transmissivity (visibility) from $\mathbf{x}$ to light source $j$, accounting for attenuation by the volume [S2, unofficial].
*   $\mathbf{l}_j$ is the direction to light source $j$.
*   $\mathbf{n}$ is the surface normal (if applicable for boundary lighting) or assumed direction [S2, unofficial].

At render time, the incident radiance is approximated via trilinear interpolation:
$$ L_i(\mathbf{p}) \approx \text{TrilinearInterpolate}(\text{BakedGrid}, \mathbf{p}) $$
Where $\mathbf{p}$ is the current sample position on the camera ray [S2, unofficial].

**Parameters:**
*   **Cache Resolution:** The resolution of the precomputed lighting grid. It can be lower than the density grid resolution because light is blurry [S3, unofficial]. Not given in the sources for exact ratios.
*   **Light Samples:** In some hybrid approaches, a few real light samples are taken before sampling the cache, but the core method relies on full precomputation [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Dynamic Scenes:** The technique requires re-baking whenever the lighting configuration or the density simulation changes [S1, unofficial]. This makes it unsuitable for highly dynamic scenes without significant offline computation overhead [S1, unofficial].
*   **Memory Usage:** Storing full radiance vectors per voxel increases memory consumption compared to storing just density [S1, unofficial].
*   **Precision:** Trilinear interpolation of baked lighting can introduce artifacts if the cache resolution is too low relative to sharp shadow boundaries [S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bake_lighting(volume_grid, lights, resolution):
    """
    Pre-computes incident radiance for each voxel.
    volume_grid: 3D array of density values.
    lights: List of light sources with position, color, intensity.
    resolution: Dimensions of the voxel grid.
    Returns: 3D array of precomputed radiance values.
    """
    baked_lighting = create_3d_array(resolution, channels=3) # RGB
    
    for z in range(resolution.z):
        for y in range(resolution.y):
            for x in range(resolution.x):
                voxel_center = get_voxel_center(x, y, z, resolution)
                total_radiance = vec3(0.0, 0.0, 0.0)
                
                for light in lights:
                    # Calculate visibility/transmissivity from voxel to light
                    # This involves raymarching from voxel_center towards light
                    transmissivity = compute_transmissivity(volume_grid, voxel_center, light.position)
                    
                    # Accumulate direct lighting
                    # Assuming simple diffuse or direct illumination model
                    light_direction = normalize(light.position - voxel_center)
                    # Note: For pure volume, normal might be undefined or assumed; 
                    # S2 mentions 'n' for surface, but for volume, we often just use transmissivity * intensity
                    # S3 implies direct lighting calculation per voxel
                    contribution = light.color * light.intensity * transmissivity
                    total_radiance += contribution
                
                baked_lighting[x, y, z] = total_radiance
                
    return baked_lighting

def render_with_baked_lighting(camera_ray, volume_grid, baked_lighting, resolution):
    """
    Renders volume using precomputed lighting.
    """
    accumulated_color = vec3(0.0, 0.0, 0.0)
    accumulated_alpha = 0.0
    step_size = get_step_size()
    t = 0.0
    
    while t < 1.0 and accumulated_alpha < 1.0:
        sample_pos = camera_ray.origin + camera_ray.direction * t
        density = sample_density(volume_grid, sample_pos)
        
        if density > 0:
            # Retrieve precomputed lighting via trilinear interpolation
            # S2: Trilinearly interpolate at render time
            li = trilinear_interpolate(baked_lighting, sample_pos, resolution)
            
            # Standard volume rendering integration (front-to-back)
            # Li is the incident radiance, assuming no in-scattering for simplicity or 
            # that in-scattering is handled separately/not baked
            scattering = density * li # Simplified shading
            
            alpha = 1.0 - exp(-density * step_size)
            accumulated_color += scattering * alpha * (1.0 - accumulated_alpha)
            accumulated_alpha += alpha * (1.0 - accumulated_alpha)
            
        t += step_size
        
    return accumulated_color
```

## Subtopics

### [Hybrid Preshading with Postclassification](2171-hybrid-preshading-with-postclassification.md)
Decouples classification and shading to balance fidelity and performance, using precomputed reflectance maps for shading before interpolation.

### [Pre-computation](2330-pre-computation.md)
Calculates intermediate radiance variables and density derivatives to avoid redundant calculations during ray marching, including first-order inscattering approximations.

### [Precomputed Radiance Transfer (PRT)](9429-precomputed-radiance-transfer-prt.md)
Models light interaction using transfer functions for arbitrary runtime lighting, offering flexibility at the cost of higher runtime complexity compared to static baking.

## Operating characteristics
*   **Time Complexity:** Baking is $O(V \cdot L \cdot S)$, where $V$ is the number of voxels, $L$ is the number of lights, and $S$ is the number of samples per light ray for visibility [S1, S2, unofficial]. Rendering is $O(R \cdot S)$, where $R$ is the number of rays and $S$ is samples per ray, but with a significantly smaller constant factor because $L_i$ evaluation is replaced by a texture lookup [S1, S3, unofficial].
*   **Memory Complexity:** $O(V)$, where $V$ is the number of voxels, storing additional RGB values per voxel [S1, unofficial].
*   **Performance:** Significantly speeds up rendering by eliminating per-sample light visibility tests [S1, S3, unofficial]. However, the initial baking pass can be slow [S1, unofficial].
*   **Strengths:** Fast runtime rendering; view-independent [S2, unofficial].
*   **Limitations:** Static lighting; requires re-baking for changes; high memory usage for high-resolution grids; not suitable for dynamic scenes [S1, unofficial].
*   **When to use:** Static scenes with complex volumetric shadows where performance is critical and lighting does not change frequently [S1, S2, unofficial].
*   **When not to use:** Dynamic lighting, dynamic geometry, or highly interactive scenes requiring frequent lighting updates [S1, unofficial].

## Background
Precomputed Lighting (Baking) is a historical technique mentioned for reference and educational purposes in modern volume rendering contexts [S1, unofficial]. It is based on the principle of precalculating illumination for each voxel center to allow trilinear interpolation at render time [S2, unofficial]. The method is view-independent but scene and light source dependent [S2, unofficial]. While shadow maps and deep shadow maps are related concepts, baking lighting directly into the voxel grid is a distinct approach for volumetric data [S1, unofficial].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S3] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
