# Deferred Voxel Shading

Also called: Deferred Voxel Cone Tracing, Villegas Voxel Cone Tracing, Villegas Voxel Shading

## Overview
Deferred Voxel Shading is a real-time global illumination technique that merges voxel cone tracing with deferred shading to approximate indirect lighting effects such as diffuse/specular reflection, color bleeding, emissive materials, and ambient occlusion [S2, unofficial]. Unlike standard voxel cone tracing which often operates in forward rendering, this method delays lighting computations until the fragment shader stage, avoiding redundant calculations for overlapping geometry and enabling efficient handling of complex lighting scenarios with multiple light sources [S1, unofficial]. It builds on [Real-time approximations](36-real-time-approximations.md) by utilizing spatial discretization via voxelization and dimensionality reduction through anisotropic mipmap levels to approximate the radiative transfer integral [G].

## How it works
The method consists of four primary stages: Voxelization, Voxel Illumination, Anisotropic Voxel Generation, and Voxel Cone Tracing.

### Inputs and Outputs
**Inputs:**
*   Scene geometry (triangles).
*   Material properties: Albedo ($\rho$), Surface Normal ($N$), Emission ($E$), Specular exponent ($n$).
*   Light sources: Point, Spot, Directional (with intensity $L_i$ and direction $\Psi$).
*   Camera parameters: Position, View Direction, Aperture angle $\theta$ for cones.

**Outputs:**
*   Final fragment color with indirect illumination.
*   Updated 3D texture volumes (Radiance, Directional, Flag).

### Data Structures
The technique relies on several 3D textures bound as `uimage3D` (unsigned integer 3D textures) to support atomic operations and hardware filtering.

1.  **Voxelization Textures (Base Level):**
    *   `voxelAlbedo`: Stores averaged albedo. Format: `r32ui` (or similar RGBA8 equivalent).
    *   `voxelNormal`: Stores averaged surface normals. Format: `r32ui` (or similar RGBA8 equivalent).
    *   `voxelEmission`: Stores emission values. Format: `r32ui` (or similar RGBA8 equivalent).
    *   *Note:* Atomic operations (`imageAtomicRGBA8Avg`) are used to average values from multiple fragments covering the same voxel [S2, unofficial].

2.  **Flag Volume:**
    *   `voxelFlag`: Single-value 3D texture. Identifies voxels as static (1) or dynamic (0). Used to prevent overwriting static geometry during dynamic updates [S2, unofficial].

3.  **Radiance Volume:**
    *   `radianceVolume`: Stores the maximum level of detail (LOD) lighting computations per voxel. Represents the base direct illumination and emission data [S2, unofficial].

4.  **Directional Volumes:**
    *   Six 3D textures at half resolution of the radiance volume.
    *   One texture per axis direction: Positive/Negative X, Y, Z.
    *   Mipmap levels store directional values for anisotropic sampling. Linear interpolation is used between the radiance volume and directional volumes when the required LOD falls between the maximum level and the first filtered level [S2, unofficial].

### Numbered Steps

#### 1. Voxelization
Generate the base voxel structures from input geometry.
1.  Perform conservative voxelization of the scene geometry.
2.  For each voxel covered by a fragment:
    *   Read the `voxelFlag` at the target position.
    *   If the flag indicates the position is **static**, discard writing [S2, unofficial].
    *   If the flag indicates **dynamic** or empty:
        *   Compute fragment albedo, normal, and emission.
        *   Use atomic average operations to update `voxelAlbedo`, `voxelNormal`, and `voxelEmission` at the voxel position [S2, unofficial].
        *   If the voxel was previously empty, write `1` to `voxelFlag` to mark it static (for static geometry) or leave it dynamic (for dynamic geometry, handled by clearing previous dynamic voxels in a prior step) [S2, unofficial].
3.  Clear previous dynamic voxels from the 3D textures before dynamic voxelization begins. This is done by reading the flag volume; if a voxel exists and is dynamic, clear its data [S2, unofficial].

#### 2. Voxel Illumination
Calculate direct lighting per voxel using a compute shader.
1.  Iterate over all voxels. Extract the voxel position in world space and the averaged normal $N$ from `voxelNormal`.
2.  For each light source $i$:
    *   Determine light direction $\Psi$ and intensity $L_i$.
    *   **Voxel Occlusion Check:** Trace a ray from the voxel position in direction $\Psi$. Sample the voxelized volume (e.g., `voxelAlbedo` or a dedicated occupancy texture). If a voxel exists at the ray position, the voxel is occluded [S2, unofficial].
    *   **Soft Shadow Accumulation:** If soft shadows are enabled, accumulate occlusion $\nu$ along the ray:
        $$ \nu = \nu + (1 - \nu)\kappa \div t $$
        where $\kappa$ is the value accumulated per collision and $t$ is the traced distance. The final occlusion factor is $1 - \nu$. This exploits the observation that rays passing through geometry borders have more collisions [S2, unofficial].
    *   **Normal-Weighted Attenuation:** To handle uneven normals within a voxel, calculate attenuation per face:
        $$ D_{x,y,z} = (\hat{i}\cdot\Psi, \hat{j}\cdot\Psi, \hat{k}\cdot\Psi) $$
        Select dominant faces based on the sign of the voxel averaged normal components $N_\omega$:
        $$ D_{\omega} = \begin{cases} \max\{D_\omega, 0\}, & N_\omega>0 \\ \max\{-D_\omega, 0\}, & \text{otherwise} \end{cases} $$
        where $\omega \in \{x, y, z\}$ [S2, unofficial].
    *   **Reflectance Model:** Compute the direct radiance contribution $V_r$:
        $$ W = N^2 $$
        $$ V_{r} = L_{i}\frac{\rho}{\pi}(W_x D_x + W_y D_y + W_z D_z) $$
        where $\rho$ is the voxel albedo and $N$ is the voxel normal vector [S2, unofficial].
    *   **Emission Handling:** Add the emission term from `voxelEmission` to the direct illumination value. These voxels contribute indirect light even in occluded areas during cone tracing [S2, unofficial].
3.  Store the resulting outgoing radiance in the `radianceVolume` [S2, unofficial].

#### 3. Generate Anisotropic Voxels
Compute directional values for indirect lighting approximation.
1.  For each mipmap level (from base up):
    *   Execute a compute shader thread per voxel.
    *   Perform volumetric integration in depth to compute directional values.
    *   Average the values to obtain the resulting directional value for each of the six axes (positive/negative X, Y, Z).
    *   Store these values in the corresponding `directionalVolume` textures at the current mipmap level [S2, unofficial].

#### 4. Voxel Cone Tracing
Performed in the fragment shader as a screenspace effect.
1.  **Cone Definition:** Define a cone with origin $C_o$ (fragment position), direction $C_d$ (view direction or BRDF reflected direction), and aperture angle $\theta$.
2.  **BRDF Matching:**
    *   **Diffuse:** Use a few large cones distributed over the normal-oriented hemisphere.
    *   **Specular:** Use a single cone in the reflected direction. The aperture $\theta$ depends on the specular exponent [S2, unofficial].
3.  **Cone Advancement:**
    *   Initialize accumulated color $c = 0$ and accumulated occlusion $\alpha = 0$.
    *   Loop for cone steps:
        *   Calculate traced distance $t$.
        *   Calculate cone diameter:
            $$ d = 2t \times \tan(\theta \div 2) $$
        *   Determine mipmap level:
            $$ V_{level} = \log_2(d \div V_{size}) $$
            where $V_{size}$ is the size of a voxel at the maximum level of detail [S2, unofficial].
        *   **Sampling:**
            *   Determine which three directional volumes to sample based on the cone direction.
            *   Retrieve outgoing radiance $c_2$ and occlusion $\alpha_2$ from the voxel structure using quadrilinear interpolation (hardware support for 3D texture filtering) [S2, unofficial].
            *   If $V_{level}$ is between the maximum level and the first filtered level, linearly interpolate between the `radianceVolume` and the `directionalVolume` [S2, unofficial].
        *   **Accumulation:** Update $c$ and $\alpha$ using volumetric front-to-back accumulation:
            $$ c = \alpha c + (1-\alpha)\alpha_2 c_2 $$
            $$ \alpha = \alpha + (1-\alpha)\alpha_2 $$
        *   **Step Size:** Advance the cone. The distance $d'$ between steps is modified by a factor $\beta$:
            $$ d' = \beta \times d $$
            Typical value $\beta = 1$. Values $< 1$ produce higher quality but require more samples [S2, unofficial].
4.  **Ambient Occlusion (Optional):**
    *   Use the same diffuse cones.
    *   Accumulate only the occlusion value $\alpha_2$.
    *   Multiply by weighting function $f(r) = \frac{1}{1+\lambda r}$, where $r$ is the current radius and $\lambda$ is a parameter [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Deferred Voxel Shading Implementation

# Data Structures
# voxelAlbedo: uimage3D
# voxelNormal: uimage3D
# voxelEmission: uimage3D
# voxelFlag: uimage3D (single channel)
# radianceVolume: uimage3D
# directionalVolumes: [uimage3D, uimage3D, uimage3D, uimage3D, uimage3D, uimage3D] # +X, -X, +Y, -Y, +Z, -Z
# V_size: float (voxel size at max LOD)
# beta: float (step size factor, default 1.0)

def step_1_voxelization(geometry, is_static):
    """
    Voxelize scene geometry into base 3D textures.
    """
    for fragment in geometry:
        voxel_pos = get_voxel_position(fragment.position)
        
        # Check flag for dynamic/static handling
        flag = imageLoad(voxelFlag, voxel_pos).r
        if is_static:
            if flag == 0:
                # Write new static voxel
                imageStore(voxelFlag, voxel_pos, vec4(1.0, 0.0, 0.0, 0.0))
                # Atomic average for albedo, normal, emission
                imageAtomicRGBA8Avg(voxelAlbedo, voxel_pos, fragment.albedo)
                imageAtomicRGBA8Avg(voxelNormal, voxel_pos, fragment.normal)
                imageAtomicRGBA8Avg(voxelEmission, voxel_pos, fragment.emission)
        else:
            # Dynamic geometry
            if flag == 1:
                # Do not overwrite static voxels
                return
            else:
                # Overwrite or create dynamic voxel
                imageAtomicRGBA8Avg(voxelAlbedo, voxel_pos, fragment.albedo)
                imageAtomicRGBA8Avg(voxelNormal, voxel_pos, fragment.normal)
                imageAtomicRGBA8Avg(voxelEmission, voxel_pos, fragment.emission)

def step_2_voxel_illumination():
    """
    Compute direct lighting per voxel in a compute shader.
    """
    for voxel_pos in all_voxels:
        normal = imageLoad(voxelNormal, voxel_pos).xyz
        albedo = imageLoad(voxelAlbedo, voxel_pos).rgb
        emission = imageLoad(voxelEmission, voxel_pos).rgb
        world_pos = voxel_to_world(voxel_pos)
        
        direct_radiance = vec3(0.0)
        
        for light in scene_lights:
            light_dir = normalize(light.position - world_pos)
            light_intensity = light.intensity
            
            # Voxel Occlusion Check
            occlusion_factor = 1.0
            if soft_shadows_enabled:
                nu = 0.0
                t = 0.0
                ray_pos = world_pos
                while t < max_distance:
                    ray_voxel = world_to_voxel(ray_pos)
                    if imageLoad(voxelAlbedo, ray_voxel).a > 0: # Check occupancy
                        nu = nu + (1.0 - nu) * 1.0 / t # kappa=1
                    t += step_size
                    ray_pos += light_dir * step_size
                occlusion_factor = 1.0 - nu
            else:
                # Hard shadow check
                ray_voxel = world_to_voxel(world_pos + light_dir * epsilon)
                if imageLoad(voxelAlbedo, ray_voxel).a > 0:
                    occlusion_factor = 0.0
            
            # Normal-Weighted Attenuation
            D_x = dot(vec3(1,0,0), light_dir)
            D_y = dot(vec3(0,1,0), light_dir)
            D_z = dot(vec3(0,0,1), light_dir)
            
            N_x = normal.x
            N_y = normal.y
            N_z = normal.z
            
            # Select dominant faces
            if N_x > 0: D_x = max(D_x, 0.0)
            else: D_x = max(-D_x, 0.0)
            
            if N_y > 0: D_y = max(D_y, 0.0)
            else: D_y = max(-D_y, 0.0)
            
            if N_z > 0: D_z = max(D_z, 0.0)
            else: D_z = max(-D_z, 0.0)
            
            # Reflectance Model
            W_x = N_x * N_x
            W_y = N_y * N_y
            W_z = N_z * N_z
            
            radiance_contribution = light_intensity * (albedo / PI) * (W_x * D_x + W_y * D_y + W_z * D_z)
            direct_radiance += radiance_contribution * occlusion_factor
            
        # Add Emission
        final_radiance = direct_radiance + emission
        imageStore(radianceVolume, voxel_pos, vec4(final_radiance, 1.0))

def step_3_generate_anisotropic_voxels():
    """
    Compute directional values for mipmap levels.
    """
    for level in range(1, max_mip_levels):
        for voxel_pos in voxels_at_level(level):
            # Volumetric integration in depth to compute directional values
            # Average values from previous level (level-1)
            # Store in corresponding directionalVolume based on axis
            pass # Implementation details of volumetric integration omitted

def step_4_cone_tracing(fragment_normal, fragment_specular_exponent):
    """
    Fragment shader entry point for indirect lighting.
    """
    # Define Cone
    cone_origin = fragment_position
    cone_dir = normalize(fragment_normal) # For diffuse
    theta = calculate_aperture(fragment_specular_exponent) # For specular, or fixed for diffuse
    
    accumulated_color = vec3(0.0)
    accumulated_alpha = 0.0
    t = 0.0
    
    while t < max_trace_distance:
        # Calculate diameter and level
        diameter = 2.0 * t * tan(theta / 2.0)
        level = log2(diameter / V_size)
        
        # Determine which directional volumes to sample
        # Based on cone_dir, select 3 volumes from the 6 directional volumes
        samples = sample_directional_volumes(cone_dir, level)
        
        # Retrieve radiance and occlusion
        c2 = samples.radiance
        alpha2 = samples.occlusion
        
        # Update accumulation
        new_alpha = accumulated_alpha + (1.0 - accumulated_alpha) * alpha2
        new_color = accumulated_alpha * accumulated_color + (1.0 - accumulated_alpha) * alpha2 * c2
        
        accumulated_alpha = new_alpha
        accumulated_color = new_color
        
        # Advance step
        step_dist = beta * diameter
        t += step_dist
        
        if accumulated_alpha >= 1.0:
            break
            
    return accumulated_color
```

### Parameters
*   $\beta$: Factor modifying the distance $d'$ between cone steps. $\beta = 1$ makes $d'$ equivalent to current diameter $d$. Values $< 1$ produce higher quality but require more samples [S2, unofficial].
*   $\theta$: Aperture angle of the cone. Depends on specular exponent for specular reflection [S2, unofficial].
*   $\lambda$: Parameter in ambient occlusion weighting function $f(r) = \frac{1}{1+\lambda r}$ [S2, unofficial].

### Edge Cases and Pitfalls
*   **Dynamic Voxelization:** Must carefully manage the `voxelFlag` to avoid overwriting static geometry. Clearing dynamic voxels requires a separate pass before dynamic voxelization [S2, unofficial].
*   **Normal Averaging:** Uneven normals within a voxel can lead to incorrect lighting. The normal-weighted attenuation model mitigates this but adds complexity [S2, unofficial].
*   **Precision:** Soft shadow accumulation $\nu$ can suffer from precision issues if $t$ is very small or very large. The formula $\nu = \nu + (1 - \nu)\kappa \div t$ assumes $\kappa$ is normalized [S2, unofficial].
*   **Mipmap Interpolation:** Linear interpolation between `radianceVolume` and `directionalVolume` is required when the calculated LOD falls between the maximum level and the first filtered level. Failure to do so causes visual artifacts [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Voxelization is $O(N)$ where $N$ is the number of fragments. Voxel Illumination is $O(V \times L)$ where $V$ is the number of voxels and $L$ is the number of lights. Cone Tracing is $O(F \times S)$ where $F$ is the number of fragments and $S$ is the number of cone steps.
*   **Memory Complexity:** Requires multiple 3D textures. Base voxelization: 3 textures. Radiance: 1 texture. Directional: 6 textures at half resolution. Flag: 1 texture. Total memory scales with voxel grid resolution and mipmap levels.
*   **Strengths:** Handles complex lighting effects (emissive, multiple bounces, color bleeding) in real-time. Deferred shading backend avoids redundant lighting for overlapping geometry. Compatible with standard light types.
*   **Limitations:** High memory usage due to 3D textures. Voxelization can be expensive for high-resolution scenes. Approximation quality depends on voxel resolution and cone step size.
*   **When to Use:** Scenes with many light sources, emissive materials, and where indirect lighting is desired without iterative solvers.
*   **When Not to Use:** Memory-constrained environments, scenes with very fine geometric details that require high voxel resolution, or when exact shadowing is required.

## Background
Deferred Voxel Shading was presented by Jose Villegas as an expansion upon rudimentary voxel cone tracing, merging it with deferred shading techniques [S1, unofficial]. It aims to provide a real-time global illumination solution that approximates indirect diffuse, specular reflectance, color-blending, emissive materials, indirect shadows, and ambient occlusion [S2, unofficial]. The technique leverages the spatial and directional coherence of voxels to approximate the radiative transfer equation without iterative solving [G].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] Deferred Voxel Shading for Real Time Global Illumination, https://jose-villegas.github.io/post/deferred_voxel_shading/, fetched 2026-10-27 (unofficial)
