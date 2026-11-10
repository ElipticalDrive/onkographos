# Deferred Voxel Shading

Also called: Deferred Voxel Cone Tracing, Villegas Voxel Cone Tracing, Villegas Voxel Shading

## Overview
Deferred Voxel Shading is a real-time global illumination technique that merges voxel cone tracing with deferred shading to compute indirect lighting effects such as diffuse/specular reflection, color blending, emissive materials, and ambient occlusion [S2, unofficial]. Unlike forward rendering approaches, it delays lighting computations until after all geometry is rendered, avoiding redundant calculations for overlapping polygons and enhancing performance in scenes with many light sources [S1, unofficial]. It builds on [Voxel Cone Tracing](1454-voxel-cone-tracing.md) by introducing anisotropic voxel filtering and a dedicated voxel illumination pass [S2, unofficial].

## How it works
The method consists of four main stages: Voxelization, Voxel Illumination, Anisotropic Voxel Generation, and Voxel Cone Tracing [S2, unofficial].

### Inputs and Outputs
**Inputs:**
*   Scene geometry (triangles).
*   Material properties: Albedo ($\rho$), Surface Normal ($N$), Emission ($E$), Specular exponent.
*   Light sources: Point, Spot, Directional.
*   Camera parameters: Position, View Direction, Aperture angle ($\theta$).

**Outputs:**
*   Final fragment color with indirect illumination.

### Data Structures
The technique uses several 3D textures bound as `uimage3D` (unsigned integer) or standard floating-point textures for storage [S2, unofficial].

1.  **Voxelization Storage:** Three 3D textures with `r32ui` format (or equivalent unsigned integer format) for atomic averaging [S2, unofficial].
    *   `voxelAlbedo`: Stores averaged albedo.
    *   `voxelNormal`: Stores averaged surface normal.
    *   `voxelEmission`: Stores averaged emission.
2.  **Flag Volume:** A single-value 3D texture used to identify voxels as static. Used to prevent overwriting static voxels during dynamic updates [S2, unofficial].
3.  **Radiance Volume:** A 3D texture storing the maximum level of detail (LOD) for the voxelized scene. It holds the computed outgoing radiance per voxel after the illumination pass [S2, unofficial].
4.  **Directional Volumes:** Six 3D textures at half the resolution of the Radiance Volume. One for each axis direction (positive/negative X, Y, Z). Mipmap levels store directional values for anisotropic sampling [S2, unofficial].

### Step 1: Voxelization
The scene geometry is voxelized using conservative rasterization. Atomic operations are used to average values per voxel to handle overlapping fragments [S2, unofficial].

1.  **Static/Dynamic Separation:**
    *   Initialize `voxelAlbedo`, `voxelNormal`, `voxelEmission`, and `flagVolume` to zero.
    *   For static geometry:
        *   Rasterize triangles into the voxel grid.
        *   For each voxel hit, compute average albedo, normal, and emission.
        *   Write `1` to `flagVolume` at the voxel position to mark it as static.
    *   For dynamic geometry (per frame):
        *   Clear previous dynamic voxels: Read `flagVolume`. If a voxel exists and `flagVolume` is `0` (dynamic), reset `voxelAlbedo`, `voxelNormal`, and `voxelEmission` to zero.
        *   Rasterize dynamic triangles.
        *   For each voxel hit, read `flagVolume`. If `flagVolume` is `1` (static), discard writing. Otherwise, update the voxel values using atomic averaging.
2.  **Atomic Averaging:**
    Use `imageAtomicRGBA8Avg` (or equivalent) to average the incoming fragment values with existing voxel values [S2, unofficial].

### Step 2: Voxel Illumination
Direct lighting is computed per voxel in a compute shader using the data from Step 1. This step calculates outgoing radiance $V_r$ for each voxel [S2, unofficial].

1.  **Voxel Position:** Extract world-space position from voxel 3D coordinates.
2.  **Normal Weighted Attenuation:**
    To handle uneven normals within a voxel, calculate attenuation per face based on the light direction $\Psi$ [S2, unofficial]:
    $$ D_{x,y,z} = (\hat{i}\cdot\Psi, \hat{j}\cdot\Psi, \hat{k}\cdot\Psi) $$
    Select dominant faces based on the sign of the voxel averaged normal vector component $N_\omega$:
    $$ D_{\omega} = \begin{cases} \max\{D_\omega, 0\}, & N_\omega>0 \\ \max\{-D_\omega, 0\}, & \text{otherwise} \end{cases} $$
3.  **Reflectance Model:**
    Compute the weight $W$ and outgoing radiance $V_r$ [S2, unofficial]:
    $$ W = N^2 $$
    $$ V_{r} = L_{i}\frac{\rho}{\pi}(W_x D_x + W_y D_y + W_z D_z) $$
    Where $L_i$ is light intensity, $\rho$ is voxel albedo, and $N$ is the voxel normal vector.
4.  **Emission:**
    Add the emission term from the voxelization process to $V_r$. This allows emissive materials to contribute indirect light even in occluded areas [S2, unofficial].
5.  **Voxel Occlusion (Soft Shadows):**
    Trace a ray from the voxel position in the direction of the light. Sample the voxelized volume to determine if a voxel exists at the ray position. If true, the voxel is occluded. For soft shadows, accumulate value $\kappa$ per collision and divide by traced distance $t$ [S2, unofficial]:
    $$ \nu = \nu + (1 - \nu)\kappa\div t $$
    Where $1 - \nu$ represents the final occlusion value. Store this occlusion value in the Radiance Volume alongside $V_r$ [S2, unofficial].

### Step 3: Generate Anisotropic Voxels
To improve cone tracing precision, anisotropic filtering is applied using volumetric integration in depth [S2, unofficial].

1.  **Directional Volumes:**
    For each of the six axis directions (positive/negative X, Y, Z), execute a compute shader thread per voxel at each mipmap level.
2.  **Volumetric Integration:**
    Use values from the previous mipmap level to compute directional values. Average the values to obtain the resulting directional value for a certain direction [S2, unofficial].
3.  **Mipmap Levels:**
    Store these directional values in the corresponding Directional Volumes. The mipmap levels store the levels of detail for anisotropic sampling [S2, unofficial].

### Step 4: Voxel Cone Tracing
Performed in the fragment shader as a screenspace effect. Cones advance, sampling volume increases via mipmap levels. Quadrilinear interpolation is used for smooth variation [S2, unofficial].

1.  **Cone Definition:**
    Define cone origin $C_o$, direction $C_d$, and aperture angle $\theta$ [S2, unofficial].
2.  **Cone Diameter:**
    Calculate diameter $d$ at traced distance $t$ [S2, unofficial]:
    $$ d=2t\times\tan(\theta\div 2) $$
3.  **Mipmap Level Selection:**
    Determine the mipmap level $V_{level}$ to sample [S2, unofficial]:
    $$ V_{level} = log_2(d\div V_{size}) $$
    Where $V_{size}$ is the size of a voxel at the maximum level of detail. If $V_{level}$ ranges between the maximum level and the first filtered level of detail, use linear interpolation between samples from the Radiance Volume and Directional Volumes [S2, unofficial].
4.  **Directional Sampling:**
    The cone direction determines which three Directional Volumes are sampled. Linearly interpolate the three samples obtained from the selected directional volumes [S2, unofficial].
5.  **Volumetric Front-to-Back Accumulation:**
    Track occlusion $\alpha$ and color $c$ (indirect light) [S2, unofficial]. Retrieve occlusion $\alpha_2$ and outgoing radiance $c_2$ from the voxel structure. Update values [S2, unofficial]:
    $$ c =\alpha c + (1-\alpha)\alpha_2c_2 $$
    $$ \alpha=\alpha+(1-\alpha)\alpha_2 $$
6.  **Step Size:**
    Modify the distance $d'$ between cone steps by a factor $\beta$. With $\beta = 1$, $d'$ is equivalent to the current diameter $d$. Values $< 1$ produce higher quality but require more samples [S2, unofficial].
7.  **Indirect Illumination Approximation:**
    Use a crude Monte Carlo approximation. Partition the hemisphere integral into a sum of integrals resembling cones. For Blinn-Phong materials, use few large cones distributed over the normal-oriented hemisphere for diffuse reflection. Use a single cone in the reflected direction for specular reflection, where aperture depends on the specular exponent [S2, unofficial].
8.  **Ambient Occlusion:**
    Approximate using the same cones as diffuse reflection. Accumulate only the occlusion value $\alpha_2$, multiplied by weighting function $f(r) = \frac{1}{1+\lambda r}$, where $r$ is the current radius of the cone and $\lambda$ is a parameter [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Voxelization (Compute Shader)
def voxelize_geometry(geometry, flag_volume, voxel_albedo, voxel_normal, voxel_emission):
    for triangle in geometry:
        for voxel_hit in conservative_rasterize(triangle):
            pos = voxel_hit.position
            frag_normal = triangle.normal_at(voxel_hit.uv)
            frag_albedo = triangle.albedo_at(voxel_hit.uv)
            frag_emission = triangle.emission_at(voxel_hit.uv)
            
            # Check if static
            is_static = flag_volume.read(pos) == 1
            
            if is_static:
                # Static voxelization (once)
                voxel_albedo.atomic_avg(pos, frag_albedo)
                voxel_normal.atomic_avg(pos, frag_normal)
                voxel_emission.atomic_avg(pos, frag_emission)
                flag_volume.write(pos, 1)
            else:
                # Dynamic voxelization (per frame)
                # First, clear previous dynamic voxels if they exist
                if voxel_albedo.read(pos) != 0:
                    voxel_albedo.write(pos, 0)
                    voxel_normal.write(pos, 0)
                    voxel_emission.write(pos, 0)
                
                # Write new dynamic data
                voxel_albedo.atomic_avg(pos, frag_albedo)
                voxel_normal.atomic_avg(pos, frag_normal)
                voxel_emission.atomic_avg(pos, frag_emission)

# Step 2: Voxel Illumination (Compute Shader)
def compute_voxel_radiance(voxel_albedo, voxel_normal, voxel_emission, radiance_volume, lights):
    for voxel_pos in voxel_grid:
        normal = voxel_normal.read(voxel_pos)
        albedo = voxel_albedo.read(voxel_pos)
        emission = voxel_emission.read(voxel_pos)
        
        if normal == 0:
            continue
            
        total_radiance = emission
        total_occlusion = 0.0
        
        for light in lights:
            light_dir = normalize(light.position - voxel_pos)
            light_intensity = light.intensity
            
            # Normal Weighted Attenuation
            i_hat = vec3(1, 0, 0)
            j_hat = vec3(0, 1, 0)
            k_hat = vec3(0, 0, 1)
            
            D_x = dot(i_hat, light_dir)
            D_y = dot(j_hat, light_dir)
            D_z = dot(k_hat, light_dir)
            
            N_x = normal.x
            N_y = normal.y
            N_z = normal.z
            
            # Dominant faces
            D_x_dom = max(D_x, 0) if N_x > 0 else max(-D_x, 0)
            D_y_dom = max(D_y, 0) if N_y > 0 else max(-D_y, 0)
            D_z_dom = max(D_z, 0) if N_z > 0 else max(-D_z, 0)
            
            # Reflectance Model
            W_x = normal.x ** 2
            W_y = normal.y ** 2
            W_z = normal.z ** 2
            
            radiance_contribution = light_intensity * (albedo / PI) * (W_x * D_x_dom + W_y * D_y_dom + W_z * D_z_dom)
            total_radiance += radiance_contribution
            
            # Soft Voxel Shadows
            nu = 0.0
            t = 0.0
            step_size = 0.1 # Example step size
            max_dist = 10.0
            while t < max_dist:
                t += step_size
                ray_pos = voxel_pos + light_dir * t
                if voxel_exists(ray_pos): # Check any voxel texture
                    kappa = 1.0
                    nu = nu + (1 - nu) * kappa / t
                if nu >= 1.0:
                    break
            total_occlusion += nu
            
        radiance_volume.write(voxel_pos, vec4(total_radiance, total_occlusion))

# Step 3: Generate Anisotropic Voxels (Compute Shader)
def generate_anisotropic_voxels(radiance_volume, directional_volumes):
    # Six directions: +X, -X, +Y, -Y, +Z, -Z
    directions = [
        vec3(1, 0, 0), vec3(-1, 0, 0),
        vec3(0, 1, 0), vec3(0, -1, 0),
        vec3(0, 0, 1), vec3(0, 0, -1)
    ]
    
    for dir_idx, direction in enumerate(directions):
        for mipmap_level in range(1, max_mipmaps):
            for voxel_pos in voxel_grid_at_level(mipmap_level):
                # Volumetric integration from previous level
                prev_values = []
                for offset in get_voxel_neighbors(voxel_pos, direction):
                    prev_val = directional_volumes[dir_idx].read_at_mipmap(mipmap_level - 1, offset)
                    prev_values.append(prev_val)
                
                # Average
                new_val = average(prev_values)
                directional_volumes[dir_idx].write_at_mipmap(mipmap_level, voxel_pos, new_val)

# Step 4: Voxel Cone Tracing (Fragment Shader)
def voxel_cone_tracing(camera_pos, camera_dir, aperture_angle, radiance_volume, directional_volumes):
    cone_origin = camera_pos
    cone_dir = camera_dir
    theta = aperture_angle
    
    alpha = 0.0
    color = vec3(0.0)
    
    t = 0.0
    beta = 1.0 # Step size factor
    max_dist = 10.0
    
    while t < max_dist:
        # Cone diameter
        d = 2 * t * tan(theta / 2)
        
        # Mipmap level
        V_size = 1.0 # Size of voxel at max LOD
        V_level = log2(d / V_size)
        
        # Determine which directional volumes to sample based on cone_dir
        # Select 3 volumes closest to cone_dir
        selected_volumes = get_closest_directional_volumes(cone_dir)
        
        # Sample radiance and occlusion
        # Interpolate between radiance_volume and directional_volumes if needed
        c_2 = sample_voxel_structure(selected_volumes, V_level, cone_origin + cone_dir * t)
        alpha_2 = sample_occlusion(selected_volumes, V_level, cone_origin + cone_dir * t)
        
        # Front-to-back accumulation
        color = alpha * color + (1 - alpha) * alpha_2 * c_2
        alpha = alpha + (1 - alpha) * alpha_2
        
        # Step size
        d_prime = beta * d
        t += d_prime
        
        if alpha >= 1.0:
            break
            
    return color
```

### Parameters
*   **Beta ($\beta$):** Factor modifying the distance $d'$ between cone steps. $\beta = 1$ makes $d'$ equivalent to current diameter $d$. Values $< 1$ produce higher quality but require more samples [S2, unofficial].
*   **Aperture Angle ($\theta$):** Determines the cone's spread. Depends on the specular exponent for specular reflection [S2, unofficial].
*   **Lambda ($\lambda$):** Parameter for ambient occlusion weighting function $f(r) = \frac{1}{1+\lambda r}$ [S2, unofficial].

### Edge Cases and Pitfalls
*   **Normal Averaging:** Uneven normals within a voxel can cause artifacts. The normal-weighted attenuation method mitigates this [S2, unofficial].
*   **Dynamic Updates:** Static voxels must be protected from being overwritten by dynamic voxelization. The flag volume is essential for this [S2, unofficial].
*   **Precision:** Soft voxel shadows rely on raycasting within the volume. Precision may vary depending on voxel resolution [S2, unofficial].
*   **Performance:** Cone tracing is a screenspace effect. High aperture angles or small $\beta$ values increase sample count and reduce performance [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of voxels, lights, and cone trace steps. Voxelization is $O(N)$ where $N$ is the number of triangles. Voxel illumination is $O(V \times L)$ where $V$ is the number of voxels and $L$ is the number of lights. Cone tracing is $O(F \times S)$ where $F$ is the number of fragments and $S$ is the number of cone steps [S2, unofficial].
*   **Memory Complexity:** Requires storage for voxelization textures, radiance volume, directional volumes, and flag volume. Memory usage scales with voxel grid resolution and mipmap levels [S2, unofficial].
*   **Strengths:** Handles complex lighting effects like emissive materials and multiple light bounces. Efficient in scenes with many light sources due to deferred shading. No shadow maps needed for voxel occlusion [S1, S2, unofficial].
*   **Limitations:** Approximate method. Quality depends on voxel resolution and cone trace parameters. Artifacts can occur with uneven normals or low resolution [S2, unofficial].
*   **When to Use:** Real-time global illumination in deferred rendering pipelines. Scenes with many light sources and emissive materials [S1, S2, unofficial].
*   **When Not to Use:** High-fidelity offline rendering. Scenes with very fine geometric details that require higher resolution than feasible for real-time voxelization [S2, unofficial].

## Background
Deferred Voxel Shading was presented by Jose Villegas [S1, S2, unofficial]. It expands upon rudimentary voxel cone tracing by merging it with deferred shading [S1, unofficial]. The technique is inspired by voxel cone tracing and deferred rendering [S2, unofficial]. It allows for the computation of lighting effects such as emissive materials, indirect shadows, and multiple light bounces on a voxel cone tracing basis [S1, unofficial]. The generation of mipmaps utilizes anisotropic filtering methods for more precise cone trace results [S1, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] Deferred Voxel Shading for Real Time Global Illumination, https://jose-villegas.github.io/post/deferred_voxel_shading/, fetched 2026-10-27 (unofficial)
