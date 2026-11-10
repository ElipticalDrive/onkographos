# Deferred Voxel Shading

Also called: Deferred Voxel Cone Tracing, Villegas Voxel Cone Tracing, Villegas Voxel Shading

## Overview
Deferred Voxel Shading is a real-time global illumination technique that merges voxel cone tracing with deferred shading to approximate indirect lighting effects such as diffuse/specular reflection, color blending, emissive materials, and shadows [S2, unofficial]. It solves the problem of computing multiple light bounces efficiently by voxelizing scene data, computing direct lighting per voxel, and using voxel cone tracing for indirect illumination [S2, unofficial]. Unlike forward rendering methods, it delays lighting computations until after all triangles are rendered, avoiding redundant calculations for overlapping polygons [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry (static and dynamic), light sources (point, spot, directional), camera position.
*   **Outputs:** Final screen-space color with indirect illumination.

**Data Structures:**
The method relies on several 3D textures bound as `uimage3D` with `r32ui` format (or equivalent integer/float formats for specific channels):
1.  **Voxelization Storage:** Three textures storing `voxelAlbedo`, `voxelNormal`, and `voxelEmission`. Atomic operations (`imageAtomicRGBA8Avg`) are used to average values per voxel [S2, unofficial].
2.  **Radiance Volume:** A 3D texture storing the maximum level of detail for the voxelized scene, containing the result of lighting computations per voxel [S2, unofficial].
3.  **Directional Volumes:** Six 3D textures at half resolution of the radiance volume. One for each axis direction (positive and negative). Mipmap levels store the levels of detail for anisotropic sampling [S2, unofficial].
4.  **Flag Volume:** A single-value 3D texture used to identify voxels as static. Used during dynamic voxelization to prevent overwriting static voxels [S2, unofficial].

**Steps:**

1.  **Voxelization:**
    *   Generate three 3D textures from input geometry: albedo, surface-normal, and light-emission [S1, unofficial].
    *   Use conservative voxelization and atomic operations to store these values [S2, unofficial].
    *   **Dynamic Update:** Before generating a voxel, read the flag volume at the writing position. If the value indicates the position is marked static, discard writing. Clear previous dynamic voxels using compute shaders by reading the flag volume; clear if voxel exists and is dynamic [S2, unofficial].

2.  **Voxel Illumination:**
    *   Calculate outgoing radiance per voxel using stored normal and position. Direct lighting is computed per voxel in a compute shader [S2, unofficial].
    *   **Normal Weighted Attenuation:** To handle uneven normal vectors within a voxel, calculate normal attenuation per face:
        $$D_{x,y,z} = (\hat{i}\cdot\Psi, \hat{j}\cdot\Psi, \hat{k}\cdot\Psi)$$
        where $\Psi$ is the light direction [S2, unofficial].
    *   Select three dominant faces depending on the axes sign of the voxel averaged normal vector $N$:
        $$D_{\omega} = \begin{cases} max\{D_\omega, 0\}, & N_\omega>0 \\ max\{-D_\omega, 0\}, & \text{otherwise} \end{cases}$$
        where $N_\omega$ is the sign of the voxel averaged normal vector component [S2, unofficial].
    *   Compute the resulting reflectance model:
        $$W = N^2$$
        $$V_{r} = L_{i}\frac{\rho}{\pi}(W_x D_x + W_y D_y + W_z D_z)$$
        where $L_i$ is light intensity, $\rho$ is voxel albedo, and $N$ is the voxel normal vector [S2, unofficial].
    *   **Emission Handling:** Add the emission term from the voxelization process to the direct illumination value per voxel. During cone tracing, these voxels contribute indirect light even in occluded areas [S2, unofficial].
    *   **Voxel Occlusion:** Trace a ray from the voxel position in the direction of the light. Sample the voxelized volume to determine if a voxel exists at the ray position. If true, the voxel is occluded [S2, unofficial].
    *   **Soft Voxel Shadows:** Instead of stopping the ray on first collision, accumulate a value $\kappa$ per collision and divide by traced distance $t$:
        $$\nu = \nu + (1 - \nu)\kappa\div t$$
        where $1 - \nu$ represents the final occlusion value [S2, unofficial].

3.  **Generate Anisotropic Voxels:**
    *   Use volumetric integration in depth to compute directional values. Execute a compute shader thread per voxel at each mipmap level, using values from the previous level [S2, unofficial].
    *   Generate mipmaps using anisotropic filtering methods for more precise cone trace results [S1, unofficial].
    *   The directional sample is obtained by linearly interpolating the three samples obtained from the selected directional volumes [S2, unofficial].

4.  **Voxel Cone Tracing:**
    *   Performed in the fragment shader as a screenspace effect. Cones advance, sampling volume increases via mipmap levels. Quadrilinear interpolation is used for smooth variation [S2, unofficial].
    *   **Cone Definition:** Each cone is defined by origin $C_o$, direction $C_d$, and aperture angle $\theta$.
    *   **Cone Diameter:**
        $$d=2t\times\tan(\theta\div 2)$$
        where $t$ is the traced distance [S2, unofficial].
    *   **Mipmap Level Selection:**
        $$V_{level} = log_2(d\div V_{size})$$
        where $V_{size}$ is the size of a voxel at the maximum level of detail [S2, unofficial].
    *   **Accumulation:** Keep track of occlusion value $\alpha$ and color value $c$. Retrieve occlusion $\alpha_2$ and outgoing radiance $c_2$ from the voxel structure. Update using volumetric front-to-back accumulation:
        $$c =\alpha c + (1-\alpha)\alpha_2c_2$$
        $$\alpha=\alpha+(1-\alpha)\alpha_2$$
        where $c$ and $\alpha$ are the accumulated color and occlusion, and $c_2$ and $\alpha_2$ are the retrieved outgoing radiance and occlusion from the voxel structure [S2, unofficial].
    *   **Step Distance:** Modify the distance $d'$ between cone steps by a factor $\beta$. With $\beta = 1$, $d'$ is equivalent to current diameter $d$. Values $< 1$ produce higher quality but require more samples [S2, unofficial].
    *   **Indirect Illumination Approximation:** Crude Monte Carlo approximation. Partition the hemisphere integral into a sum of integrals resembling cones. Weight resulting values to obtain accumulated contribution at the cone origin [S2, unofficial].
    *   **BRDF Matching:** For Blinn-Phong material, use few large cones distributed over the normal-oriented hemisphere for diffuse reflection. Use a single cone in the reflected direction for specular reflection, where aperture depends on the specular exponent [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Voxelization (Compute Shader)
def voxelize_geometry(geometry, flag_volume):
    for voxel in geometry:
        pos = voxel.position
        # Check if static
        if flag_volume.read(pos) == STATIC:
            continue
        
        # Atomic average for albedo, normal, emission
        voxelAlbedo.atomic_avg(pos, voxel.albedo)
        voxelNormal.atomic_avg(pos, voxel.normal)
        voxelEmission.atomic_avg(pos, voxel.emission)
        
        # Mark as dynamic (or static if applicable, though dynamic is cleared later)
        # Note: Static voxels are marked during static voxelization pass
```

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 2: Voxel Illumination (Compute Shader)
def compute_voxel_radiance(radiance_volume, voxel_normal, voxel_albedo, voxel_emission, lights):
    for voxel in radiance_volume:
        pos = voxel.position
        normal = voxel_normal.read(pos)
        albedo = voxel_albedo.read(pos)
        emission = voxel_emission.read(pos)
        
        radiance = vec3(0.0)
        
        for light in lights:
            # Voxel Occlusion
            occluded = trace_ray_from_light(pos, light.direction)
            if occluded:
                continue
            
            # Normal Weighted Attenuation
            Psi = light.direction
            D_x = dot(vec3(1,0,0), Psi)
            D_y = dot(vec3(0,1,0), Psi)
            D_z = dot(vec3(0,0,1), Psi)
            
            N_x = sign(normal.x)
            N_y = sign(normal.y)
            N_z = sign(normal.z)
            
            D_w_x = max(D_x, 0) if N_x > 0 else max(-D_x, 0)
            D_w_y = max(D_y, 0) if N_y > 0 else max(-D_y, 0)
            D_w_z = max(D_z, 0) if N_z > 0 else max(-D_z, 0)
            
            W = normal * normal
            Li = light.intensity
            rho = albedo
            
            # Reflectance Model
            V_r = Li * (rho / PI) * (W.x * D_w_x + W.y * D_w_y + W.z * D_w_z)
            radiance += V_r
            
        # Add Emission
        radiance += emission
        
        radiance_volume.write(pos, radiance)
```

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 3: Generate Anisotropic Voxels (Compute Shader)
def generate_anisotropic_voxels(directional_volumes, radiance_volume):
    for level in mipmap_levels:
        for voxel in directional_volumes[level]:
            # Volumetric integration in depth from previous level
            # Thread per voxel
            prev_level = level - 1
            # Average directional values from previous level
            # This step is simplified; actual implementation involves depth integration
            directional_volumes[level].write(voxel.pos, integrate_directional(prev_level, voxel.pos))
```

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 4: Voxel Cone Tracing (Fragment Shader)
def voxel_cone_tracing(origin, direction, aperture_angle, radiance_volume, directional_volumes):
    alpha = 0.0
    color = vec3(0.0)
    t = 0.0
    beta = 1.0 # Step factor
    
    while t < max_distance:
        # Cone Diameter
        d = 2 * t * tan(aperture_angle / 2)
        
        # Mipmap Level
        V_size = voxel_size_at_max_detail
        V_level = log2(d / V_size)
        
        # Determine which directional volumes to sample based on direction
        # Select 3 volumes from 6 (pos/neg x, y, z)
        selected_volumes = get_selected_directional_volumes(direction)
        
        # Quadrilinear interpolation
        sample_color, sample_alpha = quadrilinear_interpolate(
            selected_volumes, radiance_volume, V_level, origin + direction * t
        )
        
        # Front-to-back accumulation
        new_color = alpha * color + (1 - alpha) * sample_alpha * sample_color
        new_alpha = alpha + (1 - alpha) * sample_alpha
        
        color = new_color
        alpha = new_alpha
        
        # Step distance
        d_prime = beta * d
        t += d_prime
        
    return color, alpha
```

## Operating characteristics
*   **Time Complexity:** Dependent on the number of voxels, lights, and cone trace steps. The voxelization is $O(N)$ where $N$ is the number of primitives. Illumination is $O(V \times L)$ where $V$ is voxels and $L$ is lights. Cone tracing is $O(S)$ where $S$ is the number of samples per pixel.
*   **Memory Complexity:** Requires multiple 3D textures. Radiance volume, 6 directional volumes (half resolution), and flag volume. Memory usage scales with voxel grid resolution.
*   **Performance:** Enhanced performance in scenes with many light sources due to deferred shading approach [S1, unofficial]. High quality results with $\beta < 1$ but at the cost of more samples [S2, unofficial].
*   **Strengths:** Handles complex lighting effects like emissive materials and multiple light bounces [S1, unofficial]. Compatible with point, spot, and directional lights without shadow maps [S2, unofficial].
*   **Limitations:** Requires a deferred rendering backend [S2, unofficial]. High memory usage for large voxel grids.

## Background
Deferred Voxel Shading was presented by Jose Villegas [S2, unofficial]. It expands upon rudimentary voxel cone tracing by merging it with deferred shading [S1, unofficial]. The technique is inspired by voxel cone tracing and deferred rendering [S2, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] Deferred Voxel Shading for Real Time Global Illumination, https://jose-villegas.github.io/post/deferred_voxel_shading/, fetched 2026-10-27 (unofficial)
