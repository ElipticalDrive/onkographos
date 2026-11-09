# Deferred Voxel Shading

Also called: Deferred Voxel Cone Tracing, Villegas Voxel Cone Tracing, Villegas Voxel Shading

## Overview
Deferred Voxel Shading is a four-step real-time global illumination technique that merges voxel cone tracing with deferred shading to compute indirect lighting effects including diffuse/specular reflection, color blending, emissive materials, indirect shadows, and ambient occlusion [S2, unofficial]. It differs from standard voxel cone tracing by delaying lighting computations until the fragment shader stage, thereby avoiding redundant calculations for overlapping polygons and enhancing performance in scenes with many light sources [S1, unofficial].

## How it works
The method consists of four sequential stages: Voxelization, Voxel Illumination, Anisotropic Voxel Generation, and Voxel Cone Tracing.

### Inputs and Outputs
**Inputs:**
*   Scene geometry (meshes).
*   Material properties: Albedo ($\rho$), Surface Normal ($N$), Emission, and Specular exponent (for Blinn-Phong BRDF).
*   Light sources: Point, Spot, and Directional lights.
*   Camera parameters: Position, View Direction, and Aperture Angle ($\theta$).

**Outputs:**
*   Final fragment color with indirect illumination.

### Data Structures
The technique relies on several 3D textures bound as `uimage3D` (unsigned integer) or standard 3D textures for interpolation.

1.  **Voxelization Storage:** Three 3D textures with `r32ui` format.
    *   `voxelAlbedo`: Stores averaged albedo values.
    *   `voxelNormal`: Stores averaged surface normals.
    *   `voxelEmission`: Stores emission values.
    *   *Note:* Atomic operations (`imageAtomicRGBA8Avg`) are used to average values per voxel [S2, unofficial].
2.  **Radiance Volume:** A 3D texture storing the maximum level of detail (LOD) for the voxelized scene, containing the computed outgoing radiance per voxel.
3.  **Directional Volumes:** Six 3D textures at half resolution of the radiance volume. One for each axis direction (positive/negative X, Y, Z). Mipmap levels store directional values for anisotropic sampling.
4.  **Flag Volume:** A single-value 3D texture used to identify voxels as static. Used during dynamic voxelization to prevent overwriting static voxels [S2, unofficial].

### Parameters
*   $\theta$: Cone aperture angle.
*   $\beta$: Step size factor modifying the distance $d'$ between cone steps. $\beta = 1$ makes $d'$ equivalent to the current diameter $d$. Values $< 1$ produce higher quality but require more samples [S2, unofficial].
*   $\lambda$: Ambient occlusion falloff factor (used in $f(r) = \frac{1}{1+\lambda r}$) [S2, unofficial].

### Steps

#### 1. Voxelization
Generate three 3D textures from input geometry using conservative voxelization and atomic operations [S2, unofficial].

1.  For each fragment of the input geometry:
    *   Compute world-space position.
    *   Write albedo, normal, and emission to `voxelAlbedo`, `voxelNormal`, and `voxelEmission` using atomic averaging.
2.  **Dynamic Update Handling:**
    *   Read the `flag volume` at the writing position.
    *   If the position is marked static, discard writing.
    *   To clear previous dynamic voxels, read the `flag volume` via compute shaders; clear voxels if they exist and are dynamic [S2, unofficial].

#### 2. Voxel Illumination
Calculate outgoing radiance per voxel in a compute shader using stored normal, position, and light data. This step computes direct lighting and handles occlusion [S2, unofficial].

1.  **Normal Weighted Attenuation:**
    To handle uneven normal vectors within a voxel, calculate attenuation per face:
    $$ D_{x,y,z} = (\hat{i}\cdot\Psi, \hat{j}\cdot\Psi, \hat{k}\cdot\Psi) $$
    where $\Psi$ is the light direction and $\hat{i}, \hat{j}, \hat{k}$ are the voxel axes [S2, unofficial].
2.  **Dominant Face Selection:**
    Select dominant faces based on the sign of the voxel averaged normal vector component $N_\omega$:
    $$ D_{\omega} = \begin{cases} \max\{D_\omega, 0\}, & N_\omega>0 \\ \max\{-D_\omega, 0\}, & \text{otherwise} \end{cases} $$
3.  **Reflectance Model:**
    Compute the direct radiance contribution $V_r$:
    $$ W = N^2 $$
    $$ V_{r} = L_{i}\frac{\rho}{\pi}(W_x D_x + W_y D_y + W_z D_z) $$
    where $L_i$ is light intensity, $\rho$ is voxel albedo, and $N$ is the voxel normal vector [S2, unofficial].
4.  **Emission Handling:**
    Add the emission term from the voxelization process to $V_r$. These voxels contribute indirect light even in occluded areas [S2, unofficial].
5.  **Voxel Occlusion (Soft Shadows):**
    Trace a ray from the voxel position in the direction of the light. Sample the voxelized volume to determine if a voxel exists.
    *   **Soft Shadow Accumulation:** Instead of stopping at the first collision, accumulate value $\kappa$ per collision and divide by traced distance $t$:
        $$ \nu = \nu + (1 - \nu)\kappa\div t $$
        where $1 - \nu$ represents the final occlusion value [S2, unofficial].
6.  Store the resulting radiance and occlusion in the **Radiance Volume**.

#### 3. Generate Anisotropic Voxels
Use volumetric integration in depth to compute directional values for precise cone tracing [S2, unofficial].

1.  Execute a compute shader thread per voxel at each mipmap level.
2.  Use values from the previous level to compute directional averages.
3.  Store results in the six **Directional Volumes** (one per axis direction, positive and negative) [S2, unofficial].

#### 4. Voxel Cone Tracing
Performed in the fragment shader as a screenspace effect. Cones advance, sampling volume increases via mipmap levels. Quadrilinear interpolation is used for smooth variation [S2, unofficial].

1.  **Cone Definition:**
    Define cone by origin $C_o$, direction $C_d$, and aperture angle $\theta$.
2.  **Step Loop:**
    For each step of the cone trace:
    *   Calculate traced distance $t$.
    *   Calculate cone diameter:
        $$ d=2t\times\tan(\theta\div 2) $$
    *   Determine mipmap level:
        $$ V_{level} = log_2(d\div V_{size}) $$
        where $V_{size}$ is the size of a voxel at the maximum level of detail.
    *   **Directional Selection:** Select three directional volumes based on the cone direction $C_d$. Linearly interpolate samples from these volumes.
    *   **Volumetric Front-to-Back Accumulation:**
        Retrieve occlusion $\alpha_2$ and outgoing radiance $c_2$ from the voxel structure. Update accumulated color $c$ and occlusion $\alpha$:
        $$ c =\alpha c + (1-\alpha)\alpha_2c_2 $$
        $$ \alpha=\alpha+(1-\alpha)\alpha_2 $$
    *   Advance step distance $d'$ modified by factor $\beta$ [S2, unofficial].
3.  **Indirect Illumination Approximation:**
    Use a crude Monte Carlo approximation. Partition the hemisphere integral into a sum of integrals resembling cones.
    *   **Diffuse:** Use few large cones distributed over the normal-oriented hemisphere.
    *   **Specular:** Use a single cone in the reflected direction, where aperture depends on the specular exponent (Blinn-Phong) [S2, unofficial].
4.  **Ambient Occlusion:**
    Approximate using the same cones as diffuse reflection. Accumulate only the occlusion value $\alpha_2$, weighted by $f(r) = \frac{1}{1+\lambda r}$, where $r$ is the current radius of the cone [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Voxelization (Compute Shader)
def voxelize_geometry(geometry, flag_volume):
    for fragment in geometry:
        pos = fragment.world_position
        albedo = fragment.albedo
        normal = fragment.normal
        emission = fragment.emission
        
        # Check if static
        if flag_volume.read(pos) == STATIC:
            continue
            
        # Atomic average into 3D textures
        imageAtomicRGBA8Avg(voxelAlbedo, pos, albedo)
        imageAtomicRGBA8Avg(voxelNormal, pos, normal)
        imageAtomicRGBA8Avg(voxelEmission, pos, emission)
        
        # Mark as dynamic (or static if geometry is static)
        flag_volume.write(pos, DYNAMIC)

# Step 2: Voxel Illumination (Compute Shader)
def illuminate_voxels(lights, flag_volume):
    # Clear dynamic voxels first if needed
    clear_dynamic_voxels(flag_volume)
    
    for voxel_pos in radiance_volume:
        normal = voxelNormal.read(voxel_pos)
        albedo = voxelAlbedo.read(voxel_pos)
        emission = voxelEmission.read(voxel_pos)
        
        radiance = vec3(0.0)
        occlusion = 0.0
        
        for light in lights:
            # Normal Weighted Attenuation
            D_x = dot(normal.x_axis, light.direction)
            D_y = dot(normal.y_axis, light.direction)
            D_z = dot(normal.z_axis, light.direction)
            
            # Dominant Face Selection
            N_x = sign(normal.x)
            N_y = sign(normal.y)
            N_z = sign(normal.z)
            
            D_x_dom = max(D_x, 0) if N_x > 0 else max(-D_x, 0)
            D_y_dom = max(D_y, 0) if N_y > 0 else max(-D_y, 0)
            D_z_dom = max(D_z, 0) if N_z > 0 else max(-D_z, 0)
            
            # Reflectance Model
            W = normal * normal
            Li = light.intensity
            rho = albedo
            Vr = Li * (rho / PI) * (W.x * D_x_dom + W.y * D_y_dom + W.z * D_z_dom)
            
            radiance += Vr
            
            # Soft Voxel Shadows
            nu = 0.0
            t = 0.0
            ray_pos = voxel_pos
            ray_dir = light.direction
            while t < MAX_DISTANCE:
                ray_pos += ray_dir * step_size
                if voxel_exists(ray_pos):
                    kappa = 1.0 # Collision value
                    nu = nu + (1 - nu) * kappa / t
                t += step_size
            
            occlusion += nu
            
        # Add Emission
        radiance += emission
        
        # Store in Radiance Volume
        radiance_volume.write(voxel_pos, radiance)
        occlusion_volume.write(voxel_pos, occlusion)

# Step 3: Generate Anisotropic Voxels (Compute Shader)
def generate_anisotropic_voxels():
    for mipmap_level in range(1, MAX_MIPMAPS):
        for voxel_pos in directional_volumes:
            # Volumetric integration from previous level
            prev_values = directional_volumes.read_prev_level(voxel_pos)
            # Average directional values
            new_directional = average_directional(prev_values)
            directional_volumes.write(voxel_pos, new_directional)

# Step 4: Voxel Cone Tracing (Fragment Shader)
def fragment_shader(camera, material):
    # Cone definition
    Co = camera.position
    Cd = normalize(camera.view_direction)
    theta = camera.aperture_angle
    
    accumulated_color = vec3(0.0)
    accumulated_alpha = 0.0
    
    t = 0.0
    while t < MAX_DISTANCE:
        # Cone diameter and level
        d = 2 * t * tan(theta / 2)
        V_level = log2(d / V_size)
        
        # Select directional volumes based on Cd
        # Interpolate samples from selected directional volumes
        c2 = sample_directional_volumes(Cd, V_level)
        alpha2 = sample_occlusion_volume(Co + Cd * t)
        
        # Volumetric front-to-back accumulation
        accumulated_color = accumulated_alpha * accumulated_color + (1 - accumulated_alpha) * alpha2 * c2
        accumulated_alpha = accumulated_alpha + (1 - accumulated_alpha) * alpha2
        
        # Step size modification
        d_prime = d * beta
        t += d_prime
        
    # Final color
    final_color = accumulated_color + material.emission
    return final_color
```

## Operating characteristics
*   **Time Complexity:** Dependent on the number of voxels, lights, and cone trace steps. The use of mipmaps reduces the sampling volume as distance increases, but the number of steps is controlled by $\beta$.
*   **Memory Complexity:** Requires storage for Albedo, Normal, Emission, Radiance, six Directional volumes, and Flag volume. Directional volumes are at half resolution of the radiance volume.
*   **Performance:** Enhanced performance in scenes with many light sources compared to forward rendering, as lighting is computed per fragment rather than per light [S1, unofficial].
*   **Strengths:** Handles complex lighting effects like emissive materials and multiple light bounces. No shadow maps required for voxel occlusion.
*   **Limitations:** Approximate nature of voxel cone tracing can lead to artifacts. High memory usage due to multiple 3D textures. Quality depends on $\beta$ and voxel resolution.

## Background
Deferred Voxel Shading was presented by Jose Villegas [S2, unofficial]. It expands upon rudimentary voxel cone tracing by merging it with deferred shading techniques. The approach was inspired by earlier work on voxel cone tracing, including contributions from Fredrik Präntare and Crassin [S1, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] Deferred Voxel Shading for Real Time Global Illumination, https://jose-villegas.github.io/post/deferred_voxel_shading/, fetched 2026-10-27 (unofficial)
