# Spherical Harmonic Volumetric Visibility

Also called: SH Volumetric Visibility, Spherical Harmonics in Volume Rendering

## Overview
Spherical Harmonic Volumetric Visibility is a real-time dynamic lighting technique for direct volume rendering that encodes local and global volumetric visibility using spherical harmonic basis functions [S2, unofficial]. It solves the problem of computing incident radiance in participating media by representing light source radiance and transparency (visibility) as basis functions, allowing dynamic light changes without expensive visibility updates [S1]. It sets itself apart from its siblings by explicitly compressing the directional occlusion and lighting information into spherical harmonics stored in a multiresolution grid, reducing the rendering overhead to a few texture lookups and scalar products [S1]. Unlike [Voxel Cone Tracing](1454-voxel-cone-tracing.md) which uses geometric cones, or [Froxel-based Volumetric Rendering](1919-froxel-based-volumetric-rendering.md) which uses spatial clustering, this method relies on frequency-domain representation of visibility [S1].

## How it works
**Inputs:**
*   Volume density data $\rho(\mathbf{x})$.
*   Light source definitions (directional, point, or environment maps) [S2, unofficial].
*   Transfer function (mapping density to color/opacity) [S1].
*   Clip planes (if used) [S1].

**Outputs:**
*   Rendered image with dynamic volumetric illumination [S2, unofficial].

**Data Structures:**
*   **Multiresolution Grid:** A hierarchical data structure (e.g., octree or mipmapped 3D texture) storing spherical harmonic (SH) coefficients for visibility [S1]. The grid allows sampling at different levels of detail (LOD), where the LOD decreases as the sampling distance from the voxel increases [S1].
*   **SH Coefficients:** For each voxel, a vector of coefficients representing the visibility function. The number of coefficients determines the frequency of the representation [S1].

**Method Steps:**
1.  **Precomputation of Local Visibility:**
    *   For each voxel, compute local visibility over a small neighborhood [S1].
    *   Send rays uniformly distributed on the sphere from the voxel center [S1].
    *   Integrate the extinction along these rays to determine local occlusion [S1].
    *   *Note:* This step is expensive if the transfer function or clip planes change [S1].

2.  **Precomputation of Global Visibility:**
    *   Perform piecewise integration of the local visibility to compute global visibility [S1].
    *   This involves traversing the multiresolution data structure with increasing step sizes, sampling at decreasing LODs as distance from the voxel increases [S1].
    *   Store the resulting spherical harmonic coefficients in the multiresolution grid [S2, unofficial].

3.  **Light Source Representation:**
    *   Represent the light source radiance $L_l$ using spherical harmonics [S1].
    *   For dynamic lights, the light source representation must be recomputed or rotated, which is generally fast and can be performed in real-time [S1].

4.  **Rendering (Ray-casting):**
    *   During ray-casting, sample the multiresolution grid at the current ray position [S1].
    *   Retrieve the precomputed SH coefficients for visibility [S1].
    *   Compute the spherical integral over the product of the area light source and voxel occlusion efficiently using the precomputed occlusion information [S1].
    *   Use a modified spherical harmonic projection approach tailored for volume rendering to realize non cone-shaped phase functions [S1].
    *   The final incident radiance is computed via dot products between the light SH coefficients and the visibility SH coefficients [S1].

**Formulas:**
*   The incident radiance $L_i$ is computed by projecting the product of light radiance and visibility onto the SH basis [S1].
*   Let $V(\omega)$ be the visibility function and $L(\omega)$ be the light radiance. The SH coefficients of the visibility $c_V$ and light $c_L$ are used to compute the integral:
    $$ \int_{S^2} V(\omega) L(\omega) d\omega \approx \sum_{l,m} c_{V,l,m} \cdot c_{L,l,m} $$
    where $l$ is the degree and $m$ is the order of the spherical harmonics [S1].
*   *Note:* The exact projection formula for non-cone-shaped phase functions is described in [S1] but not explicitly detailed in the provided notes.

**Parameters:**
*   **SH Degree:** Determines the number of coefficients. Higher frequencies allow better accuracy but increase storage and computation [S1].
*   **LOD Levels:** The number of levels in the multiresolution grid [S1].
*   **Neighborhood Size:** The size of the local neighborhood for local visibility computation [S1].

**Edge Cases and Pitfalls:**
*   **Transfer Function Changes:** Visibility must be recomputed when the transfer function changes, which is expensive [S1].
*   **Clip Plane Changes:** Visibility must be recomputed when clip planes change, which is expensive [S1].
*   **Light Source Position:** Light sources are generally constrained to be outside the volume, unless local visibility is used [S1].
*   **Storage Requirements:** High memory consumption due to storing SH coefficients for each voxel [S1]. An extra 3D texture for the SAT (Spherical Attenuation Texture) can be of lower resolution than the volume data [S1].
*   **Accuracy Limits:** Storage and computation time limit the accuracy, restricting illumination to low-frequency effects [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputation Phase
def precompute_visibility(volume_data, transfer_function, clip_planes, light_sources):
    # Initialize multiresolution grid for SH coefficients
    sh_grid = MultiresolutionGrid()
    
    # For each voxel in the volume
    for voxel in volume_data:
        # 1. Compute Local Visibility
        local_visibility_coeffs = compute_local_visibility(voxel, volume_data, transfer_function, clip_planes)
        
        # 2. Compute Global Visibility via Piecewise Integration
        # Sample multiresolution grid with increasing step size
        global_visibility_coeffs = compute_global_visibility(local_visibility_coeffs, sh_grid)
        
        # 3. Store Coefficients
        sh_grid.set(voxel.position, global_visibility_coeffs)
        
    return sh_grid

def compute_local_visibility(voxel, volume_data, transfer_function, clip_planes):
    # Send rays uniformly distributed on the sphere
    rays = get_uniform_rays_on_sphere()
    local_occlusion = []
    
    for ray in rays:
        # Integrate extinction along the ray
        extinction = integrate_extinction(ray, volume_data, transfer_function, clip_planes)
        local_occlusion.append(extinction)
        
    # Project local occlusion to SH coefficients
    return project_to_sh(local_occlusion)

def compute_global_visibility(local_visibility_coeffs, sh_grid):
    # Piecewise integration using multiresolution structure
    # Step size increases as distance from voxel increases
    # LOD decreases as distance increases
    global_visibility = local_visibility_coeffs
    # ... integration logic ...
    return global_visibility

# Rendering Phase
def render_ray_casting(volume_data, sh_grid, light_sources):
    for each pixel:
        ray = generate_ray(pixel)
        accumulated_radiance = 0
        
        while ray_inside_volume(ray):
            voxel = sample_volume(ray)
            
            # Retrieve precomputed SH coefficients for visibility
            visibility_coeffs = sh_grid.get(voxel.position)
            
            # Get light source SH coefficients
            light_coeffs = get_light_sh_coefficients(light_sources, voxel.position)
            
            # Compute incident radiance via dot product
            incident_radiance = dot_product(visibility_coeffs, light_coeffs)
            
            # Integrate radiance along ray
            accumulated_radiance += integrate_radiance(incident_radiance, ray)
            
            ray.step()
            
        return accumulated_radiance
```

## Subtopics

### [Real-Time Image-Based Volume Lighting](2202-real-time-image-based-volume-lighting.md)
A two-step, GPU-friendly technique that estimates spherical scattered radiance, projects it into SH, and uses it for single scattering. It sets itself apart by splitting the scattered radiance integral into volume-independent and volume-dependent factors to reduce the triple product cost to a dot product.

### [Spherical Harmonics Irradiance Encoding](4336-spherical-harmonics-irradiance-encoding.md)
Encodes full spherical irradiance information using SH basis functions for surface lighting. It sets itself apart by focusing on surface normal-dependent irradiance rather than volumetric light transport.

### [Spherical Harmonical Exponentiation](4345-spherical-harmonical-exponentiation.md)
Encodes directional volumetric visibility using SH basis functions. It sets itself apart by encoding the *absence* of light (occlusion) in the frequency domain.

## Operating characteristics
*   **Time Complexity:** Interactive rendering time, requiring only 1-4 extra texture lookups and scalar products per voxel [S1]. Precomputation of visibility is expensive, especially when the transfer function or clip planes change [S1].
*   **Memory Complexity:** High memory requirements as each SH coefficient needs to be stored for the visibility in the volume [S1]. An extra 3D texture for SAT is required, which can be of lower resolution than the volume data [S1].
*   **Strengths:** Supports dynamic light changes without expensive visibility updates [S1]. Supports an arbitrary number of dynamic directional light sources, environment maps, area lights, and point lights [S1]. Allows switching between local and global visibility during rendering [S1].
*   **Limitations:** Restricted to low-frequency illumination effects due to storage and computation limits [S1]. Light sources are generally constrained to be outside the volume [S1]. Visibility recomputation is expensive when transfer functions or clip planes change [S1].
*   **When to Use:** Real-time dynamic lighting in direct volume rendering where light sources are dynamic and outside the volume [S2, unofficial].
*   **When Not to Use:** When high-frequency lighting effects are required, or when the transfer function/clip planes change frequently [S1].

## Background
The method was developed by Kronander et al. in 2011 to achieve real-time dynamic lighting in direct volume rendering by encoding local and global volumetric visibility using spherical harmonic basis functions stored in a multiresolution grid [S2, unofficial]. It builds on earlier work by Ritschel (2007) for spherical harmonic projection and Ljung et al. (2006) for sparse volume representation [S1]. The technique allows for the support of directional lights, point lights, and environment maps [S2, unofficial]. It is used in applications requiring interactive volume rendering with advanced light setups [S1].

## Sources
[S1] Interactive Volume Rendering with Volumetric Illumination, https://doi.org/10.2312/conf/eg2012/stars/053-074, fetched 2026-10-26
[S2] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
