# Deferred Voxel Shading

Also called: Deferred Voxel Cone Tracing, Villegas Voxel Cone Tracing, Villegas Voxel Shading

## Overview
Deferred Voxel Shading is a four-step real-time global illumination technique that merges voxel cone tracing with deferred shading to compute indirect lighting effects, including diffuse/specular reflection, color blending, emissive materials, indirect shadows, and ambient occlusion [S2, unofficial]. Unlike forward rendering approaches, it delays lighting computations until the fragment shader stage, avoiding redundant calculations for overlapping polygons and enhancing performance in scenes with many light sources [S1, unofficial]. It builds on [Voxel Cone Tracing](1454-voxel-cone-tracing.md) by introducing anisotropic voxel filtering and deferred integration of direct and indirect light.

## How it works
The method consists of four sequential stages: Voxelization, Voxel Illumination, Anisotropic Voxel Generation, and Voxel Cone Tracing.

### Inputs and Outputs
**Inputs:**
*   Scene geometry (triangles).
*   Material properties: Albedo ($\rho$), Surface Normal ($N$), Emission ($E$), Specular exponent (for Blinn-Phong).
*   Light sources: Point, Spot, Directional (with intensity $L_i$ and direction $\Psi$).
*   Camera parameters: Position, View Direction, Aperture angle $\theta$ for cones.

**Outputs:**
*   Final fragment color with global illumination.

### Data Structures
The voxel structure consists of several 3D textures bound as `uimage3D` (unsigned integer) or standard 3D textures for interpolation:

1.  **Voxelization Storage (Input to Illumination):**
    *   `voxelAlbedo`: `r32ui` format. Stores averaged albedo per voxel.
    *   `voxelNormal`: `r32ui` format. Stores averaged surface normal per voxel.
    *   `voxelEmission`: `r32ui` format. Stores averaged emission per voxel.
    *   *Note:* Atomic operations (`imageAtomicRGBA8Avg`) are used to average values from fragments surrounding the voxel volume [S2, unofficial].

2.  **Flag Volume:**
    *   Single-value 3D texture. Identifies voxels as static (1) or dynamic (0). Used to prevent overwriting static voxels during dynamic updates [S2, unofficial].

3.  **Radiance Volume:**
    *   3D texture storing the maximum level of detail (LOD) for the voxelized scene. Contains the result of direct lighting computations per voxel [S2, unofficial].

4.  **Directional Volumes (Anisotropic Voxels):**
    *   Six 3D textures at half resolution of the Radiance Volume.
    *   One texture per axis direction (positive/negative X, Y, Z).
    *   Mipmap levels store directional values for anisotropic sampling [S2, unofficial].

### Numbered Steps

#### Step 1: Voxelization
1.  **Conservative Voxelization:** Rasterize scene geometry into a 3D grid.
2.  **Atomic Averaging:** For each voxel covered by a fragment, update the storage textures using atomic averages:
    *   `imageAtomicRGBA8Avg(voxelNormal, position, normal)`
    *   `imageAtomicRGBA8Avg(voxelAlbedo, position, albedo)`
    *   `imageAtomicRGBA8Avg(voxelEmission, position, emissive)`
3.  **Dynamic Update Handling:**
    *   Read the `Flag Volume` at the target voxel position.
    *   If the flag indicates the voxel is **static**, discard writing.
    *   If **dynamic**, proceed with writing.
    *   *Clearing:* Before dynamic voxelization, use compute shaders to clear previous dynamic voxels. Read the flag volume; if a voxel exists and is dynamic, clear its entries in the albedo/normal/emission textures [S2, unofficial].

#### Step 2: Voxel Illumination (Direct Lighting)
Performed in a compute shader. Calculate outgoing radiance per voxel using stored normal, position, and light data.

1.  **Voxel Position:** Extract world-space position from voxel 3D coordinates.
2.  **Normal Weighted Attenuation:** To handle uneven normals within a voxel, calculate attenuation per face:
    $$ D_{x,y,z} = (\hat{i}\cdot\Psi, \hat{j}\cdot\Psi, \hat{k}\cdot\Psi) $$
    where $\Psi$ is the light direction and $\hat{i}, \hat{j}, \hat{k}$ are the voxel axes [S2, unofficial].
3.  **Dominant Face Selection:** Select dominant faces based on the sign of the voxel averaged normal vector component $N_\omega$:
    $$ D_{\omega} = \begin{cases} \max\{D_\omega, 0\}, & N_\omega>0 \\ \max\{-D_\omega, 0\}, & \text{otherwise} \end{cases} $$
4.  **Reflectance Model:** Compute the direct radiance contribution $V_r$:
    $$ W = N^2 $$
    $$ V_{r} = L_{i}\frac{\rho}{\pi}(W_x D_x + W_y D_y + W_z D_z) $$
    where $L_i$ is light intensity, $\rho$ is voxel albedo, and $N$ is the voxel normal vector [S2, unofficial].
5.  **Emission Handling:** Add the emission term from the voxelization process to $V_r$. This allows emissive materials to contribute indirect light even in occluded areas [S2, unofficial].
6.  **Voxel Occlusion (Shadows):**
    *   Trace a ray from the voxel position in the direction of the light $\Psi$.
    *   Sample the voxelized volume (any of the voxelization textures) to determine if a voxel exists at the ray position.
    *   **Soft Shadows:** Accumulate occlusion $\nu$ by tracing the ray. For each collision, accumulate value $\kappa$:
        $$ \nu = \nu + (1 - \nu)\kappa\div t $$
        where $t$ is the traced distance. The final occlusion value is $1 - \nu$. This exploits the observation that rays passing through geometry borders have more collisions [S2, unofficial].
7.  **Store Result:** Write the computed radiance (including emission and shadowing) to the **Radiance Volume** [S2, unofficial].

#### Step 3: Generate Anisotropic Voxels
Performed in a compute shader, one thread per voxel per mipmap level.

1.  **Volumetric Integration:** For each directional volume (6 directions), perform volumetric integration in depth to compute directional values.
2.  **Averaging:** Average the directional values to obtain the resulting value for a certain direction.
3.  **Mipmap Generation:** Execute a thread per voxel at the current mipmap level, using values from the previous (higher resolution) level to filter the data. This creates anisotropic voxels that store directional information [S2, unofficial].

#### Step 4: Voxel Cone Tracing
Performed in the fragment shader as a screenspace effect.

1.  **Cone Definition:** Define cone by origin $C_o$, direction $C_d$, and aperture angle $\theta$.
2.  **Step Loop:** Advance the cone along $C_d$.
3.  **Diameter Calculation:** Calculate cone diameter $d$ at traced distance $t$:
    $$ d=2t\times\tan(\theta\div 2) $$
4.  **Mipmap Level Selection:** Determine which LOD to sample:
    $$ V_{level} = log_2(d\div V_{size}) $$
    where $V_{size}$ is the size of a voxel at the maximum level of detail [S2, unofficial].
5.  **Directional Sampling:**
    *   Determine which three directional volumes to sample based on the cone direction $C_d$.
    *   Linearly interpolate the three samples from the selected directional volumes.
    *   If $V_{level}$ ranges between the maximum level and the first filtered level, use linear interpolation between samples from the Radiance Volume and Directional Volumes [S2, unofficial].
6.  **Volumetric Accumulation:** Update accumulated color $c$ and occlusion $\alpha$ using front-to-back accumulation:
    $$ c =\alpha c + (1-\alpha)\alpha_2c_2 $$
    $$ \alpha=\alpha+(1-\alpha)\alpha_2 $$
    where $c_2$ and $\alpha_2$ are the retrieved outgoing radiance and occlusion from the voxel structure. Quadrilinear interpolation is used for smooth variation [S2, unofficial].
7.  **Step Size:** Modify the distance $d'$ between cone steps by a factor $\beta$:
    *   $\beta = 1$: $d'$ equals current diameter $d$.
    *   $\beta < 1$: Higher quality, more samples [S2, unofficial].
8.  **Indirect Illumination Approximation:**
    *   **Diffuse:** Distribute a few large cones over the normal-oriented hemisphere.
    *   **Specular:** Use a single cone in the reflected direction, with aperture depending on the specular exponent (Blinn-Phong) [S2, unofficial].
    *   **Ambient Occlusion:** Use the same cones as diffuse. Accumulate only $\alpha_2$ weighted by $f(r) = \frac{1}{1+\lambda r}$, where $r$ is the current radius and $\lambda$ is a parameter [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Voxelization (Compute Shader)
def voxelize_geometry(geometry, flag_volume):
    for fragment in geometry:
        voxel_pos = get_voxel_position(fragment)
        
        # Check static/dynamic flag
        is_static = flag_volume.read(voxel_pos)
        if is_static:
            continue # Discard writing for static voxels
            
        # Atomic averaging
        voxel_normal.atomic_avg(voxel_pos, fragment.normal)
        voxel_albedo.atomic_avg(voxel_pos, fragment.albedo)
        voxel_emission.atomic_avg(voxel_pos, fragment.emission)

# Step 2: Voxel Illumination (Compute Shader)
def compute_voxel_radiance():
    for voxel in radiance_volume:
        normal = voxel_normal.read(voxel.pos)
        albedo = voxel_albedo.read(voxel.pos)
        emission = voxel_emission.read(voxel.pos)
        
        radiance = vec3(0.0)
        
        for light in lights:
            # Normal Weighted Attenuation
            D_x = dot(vec3(1,0,0), light.direction)
            D_y = dot(vec3(0,1,0), light.direction)
            D_z = dot(vec3(0,0,1), light.direction)
            
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
            
            direct_contrib = Li * (rho / PI) * (W.x * D_x_dom + W.y * D_y_dom + W.z * D_z_dom)
            
            # Soft Shadows
            nu = 0.0
            t = 0.0
            ray_pos = voxel.pos
            while t < max_distance:
                ray_pos += light.direction * step_size
                t += step_size
                if voxel_exists(ray_pos): # Sample any voxelization texture
                    nu = nu + (1 - nu) * kappa / t
                    # Break or continue for hard shadows? Source implies accumulation for soft.
            
            occlusion = 1 - nu
            radiance += direct_contrib * occlusion
            
        # Add Emission
        radiance += emission
        
        radiance_volume.write(voxel.pos, radiance)

# Step 3: Anisotropic Voxels (Compute Shader)
def generate_anisotropic_voxels():
    for mip_level in mipmap_levels:
        for voxel in directional_volumes:
            # Volumetric integration in depth from previous level
            # Average directional values
            directional_value = integrate_depth(voxel.pos, mip_level, previous_level_data)
            directional_volumes.write(voxel.pos, directional_value)

# Step 4: Voxel Cone Tracing (Fragment Shader)
def fragment_shader():
    # Cone parameters
    Co = camera_position
    Cd = view_direction
    theta = aperture_angle
    beta = 1.0 # Step factor
    
    accumulated_color = vec3(0.0)
    accumulated_alpha = 0.0
    
    t = 0.0
    while t < max_trace_distance:
        # Diameter and Level
        d = 2 * t * tan(theta / 2)
        V_level = log2(d / V_size)
        
        # Determine directional volumes to sample based on Cd
        selected_volumes = get_directional_volumes(Cd)
        
        # Sample radiance and occlusion
        c2, alpha2 = sample_voxel_structure(selected_volumes, Co + Cd * t, V_level)
        
        # Volumetric Accumulation
        accumulated_color = accumulated_alpha * accumulated_color + (1 - accumulated_alpha) * alpha2 * c2
        accumulated_alpha = accumulated_alpha + (1 - accumulated_alpha) * alpha2
        
        # Step size
        step_dist = beta * d
        t += step_dist
        
    final_color = accumulated_color
```

### Parameters
*   **Beta ($\beta$):** Factor modifying the distance $d'$ between cone steps. $\beta = 1$ makes $d'$ equivalent to current diameter $d$. Values $< 1$ produce higher quality but require more samples [S2, unofficial].
*   **Aperture Angle ($\theta$):** Determines the cone's spread.
*   **Specular Exponent:** Determines the aperture of the specular cone in the BRDF matching [S2, unofficial].

### Edge Cases and Pitfalls
*   **Normal Averaging:** Uneven normals within a voxel space can cause artifacts. The normal-weighted attenuation method mitigates this but requires careful implementation of the dominant face selection [S2, unofficial].
*   **Dynamic Updates:** Static voxels must be protected from overwriting. The flag volume mechanism is critical for this. Failure to clear dynamic voxels correctly leads to ghosting [S2, unofficial].
*   **Precision:** Conservative voxelization is required to ensure geometry is fully captured. Atomic averaging can introduce slight blurring compared to hard voxelization [S2, unofficial].
*   **Performance:** Cone tracing is performed per fragment. High resolution and many bounces can be expensive. The anisotropic filtering adds memory bandwidth pressure [S1, unofficial][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dependent on the number of voxels, lights, and cone trace steps. Voxelization is $O(N)$ where $N$ is primitives. Illumination is $O(V \times L)$ where $V$ is voxels and $L$ is lights. Cone tracing is $O(F \times S)$ where $F$ is fragments and $S$ is samples per cone.
*   **Memory Complexity:** High. Requires multiple 3D textures (Albedo, Normal, Emission, Radiance, 6 Directional Volumes, Flag Volume). Resolution scales with scene size.
*   **Strengths:** Handles complex lighting (emissive, indirect shadows, color bleeding) in real-time. Deferred nature avoids overdraw penalties for lighting. Compatible with standard light types.
*   **Limitations:** Voxelization artifacts (jaggies, aliasing). Memory intensive. Quality depends on voxel resolution and cone sampling.
*   **When to use:** Real-time applications requiring global illumination approximations, especially with emissive materials and dynamic scenes.
*   **When not to use:** Memory-constrained environments, or when exact ray-traced accuracy is required.

## Background
Deferred Voxel Shading was presented by Jose Villegas [S2, unofficial]. It expands upon rudimentary voxel cone tracing by merging it with deferred shading techniques. The approach was inspired by Crassin's work on anisotropic voxels and optimized for real-time rendering using OpenGL compute shaders [S1, unofficial][S2, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] Deferred Voxel Shading for Real Time Global Illumination, https://jose-villegas.github.io/post/deferred_voxel_shading/, fetched 2026-10-27 (unofficial)
