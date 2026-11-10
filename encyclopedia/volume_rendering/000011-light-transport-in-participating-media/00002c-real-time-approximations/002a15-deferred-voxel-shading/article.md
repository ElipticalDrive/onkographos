# Deferred Voxel Shading

Also called: Deferred Voxel Cone Tracing, Villegas Voxel Cone Tracing, Villegas Voxel Shading

## Overview
Deferred Voxel Shading is a real-time global illumination technique that merges voxel cone tracing with deferred shading to approximate indirect lighting effects such as diffuse/specular reflection, color blending, emissive materials, and shadows [S2, unofficial]. It solves the problem of computing multiple light bounces in complex scenes by voxelizing scene data, computing direct lighting per voxel, and using voxel cone tracing for indirect illumination [S2, unofficial]. Unlike standard voxel cone tracing which may struggle with emissive sources or require forward rendering inefficiencies, this method delays lighting computations until the fragment shader stage, avoiding redundant calculations for overlapping polygons and enhancing performance in scenes with many light sources [S1, unofficial].

## How it works
The method consists of four primary stages: Voxelization, Voxel Illumination, Anisotropic Voxel Generation, and Voxel Cone Tracing [S2, unofficial].

### Inputs and Outputs
**Inputs:**
*   Scene geometry (triangles).
*   Material properties: Albedo ($\rho$), Surface Normal ($N$), Emission ($E$), Specular exponent (for BRDF matching).
*   Light sources: Point, Spot, Directional (with intensity $L_i$ and direction $\Psi$).
*   Camera parameters: Position, View Direction, Aperture angle $\theta$.

**Outputs:**
*   Final fragment color with indirect illumination.
*   Updated 3D textures (Radiance Volume, Directional Volumes, Flag Volume).

### Data Structures
The voxel structure relies on several 3D textures bound as `uimage3D` (unsigned integer 3D textures) [S2, unofficial]:

1.  **Voxelization Storage (Input to Illumination):**
    *   `voxelAlbedo`: `r32ui` format. Stores averaged albedo per voxel.
    *   `voxelNormal`: `r32ui` format. Stores averaged surface normals.
    *   `voxelEmission`: `r32ui` format. Stores light emission values.
    *   *Note:* Atomic operations `imageAtomicRGBA8Avg` are used to average values from fragments surrounding the voxel volume [S2, unofficial].

2.  **Flag Volume:**
    *   Single-value 3D texture. Identifies voxels as static (1) or dynamic (0). Used to prevent overwriting static voxels during dynamic updates [S2, unofficial].

3.  **Radiance Volume:**
    *   3D texture storing the maximum level of detail (LOD) for the voxelized scene. Contains the result of direct lighting computations per voxel [S2, unofficial].

4.  **Directional Volumes:**
    *   Six 3D textures at half resolution of the Radiance Volume.
    *   One texture per axis direction (positive/negative X, Y, Z).
    *   Mipmap levels store the directional values for anisotropic sampling [S2, unofficial].

### Steps

#### 1. Voxelization
The scene is voxelized using conservative rasterization. Static and dynamic geometry are handled separately to allow updates [S2, unofficial].

1.  **Static Voxelization:**
    *   Rasterize geometry into the voxel grid.
    *   For each voxel hit, update `voxelAlbedo`, `voxelNormal`, and `voxelEmission` using atomic averaging.
    *   Write `1` to the `Flag Volume` at the voxel position to mark it as static [S2, unofficial].

2.  **Dynamic Voxelization (Per Frame):**
    *   **Clear:** Use compute shaders to clear previous dynamic voxels. Read the `Flag Volume`; if a voxel exists and is dynamic (flag is 0), clear its data in the three storage textures [S2, unofficial].
    *   **Write:** Rasterize dynamic geometry. Before writing to a voxel, read the `Flag Volume`. If the flag indicates static (1), discard writing. Otherwise, update the storage textures and write `0` to the flag [S2, unofficial].

#### 2. Voxel Illumination
Direct lighting is computed per voxel in a compute shader using the data from Voxelization [S2, unofficial].

1.  **Normal Weighted Attenuation:**
    To handle uneven normals within a voxel, calculate attenuation per face of the voxel based on the light direction $\Psi$ [S2, unofficial]:
    $$ D_{x,y,z} = (\hat{i}\cdot\Psi, \hat{j}\cdot\Psi, \hat{k}\cdot\Psi) $$
    Select dominant faces based on the sign of the voxel averaged normal vector component $N_\omega$:
    $$ D_{\omega} = \begin{cases} \max\{D_\omega, 0\}, & N_\omega>0 \\ \max\{-D_\omega, 0\}, & \text{otherwise} \end{cases} $$
    Compute the reflectance model:
    $$ W = N^2 $$
    $$ V_{r} = L_{i}\frac{\rho}{\pi}(W_x D_x + W_y D_y + W_z D_z) $$
    Where $L_i$ is light intensity, $\rho$ is voxel albedo, and $N$ is the voxel normal vector [S2, unofficial].

2.  **Emission Handling:**
    Add the emission term from the voxelization process to the direct illumination value $V_r$ per voxel [S2, unofficial].

3.  **Voxel Occlusion (Soft Shadows):**
    Trace a ray from the voxel position in the direction of the light. Sample the voxelized volume to determine if a voxel exists at the ray position. If true, the voxel is occluded. For soft shadows, accumulate a value $\kappa$ per collision and divide by traced distance $t$:
    $$ \nu = \nu + (1 - \nu)\kappa\div t $$
    Where $1 - \nu$ represents the final occlusion value [S2, unofficial].

4.  **Store Result:**
    Store the computed outgoing radiance (Direct + Emission - Occlusion) in the **Radiance Volume** [S2, unofficial].

#### 3. Generate Anisotropic Voxels
To support precise cone tracing, directional values are computed using volumetric integration in depth [S2, unofficial].

1.  Execute a compute shader thread per voxel at each mipmap level.
2.  Use values from the previous level to compute directional averages.
3.  Store results in the six **Directional Volumes** (one per axis direction) [S2, unofficial].
4.  Generate mipmaps using anisotropic filtering methods for more precise cone trace results [S1, unofficial].

#### 4. Voxel Cone Tracing
Performed in the fragment shader as a screenspace effect. A deferred rendering backend is required [S2, unofficial].

1.  **Cone Definition:**
    Define a cone with origin $C_o$, direction $C_d$, and aperture angle $\theta$. The cone advances a certain length every step, with sampling volume increasing via the diameter $d$ [S2, unofficial]:
    $$ d = 2t \times \tan(\theta \div 2) $$
    Where $t$ is the traced distance [S2, unofficial].

2.  **Mipmap Level Selection:**
    Determine which mipmap level to sample based on the cone diameter:
    $$ V_{level} = \log_2(d \div V_{size}) $$
    Where $V_{size}$ is the size of a voxel at the maximum level of detail. Linear interpolation is used between samples of the Radiance Volume and Directional Volumes when the required level ranges between the maximum and the first filtered level [S2, unofficial].

3.  **Directional Sampling:**
    The cone direction determines which three of the six Directional Volumes are sampled. The directional sample is obtained by linearly interpolating the three samples [S2, unofficial].

4.  **Volumetric Accumulation:**
    Use quadrilinear interpolation for smooth variation. Update accumulated color $c$ and occlusion $\alpha$ using front-to-back accumulation:
    $$ c = \alpha c + (1-\alpha)\alpha_2 c_2 $$
    $$ \alpha = \alpha + (1-\alpha)\alpha_2 $$
    Where $c_2$ and $\alpha_2$ are the retrieved outgoing radiance and occlusion from the voxel structure [S2, unofficial].

5.  **Step Size:**
    Modify the distance $d'$ between cone steps by a factor $\beta$. With $\beta = 1$, $d'$ is equivalent to the current diameter $d$. Values $< 1$ produce higher quality but require more samples [S2, unofficial].

6.  **BRDF Matching (Indirect Illumination):**
    Approximate the hemisphere integral using a crude Monte Carlo approach. Partition the hemisphere into cones [S2, unofficial]:
    *   **Diffuse:** Use a few large cones distributed over the normal-oriented hemisphere.
    *   **Specular:** Use a single cone in the reflected direction, where aperture depends on the specular exponent [S2, unofficial].
    *   **Ambient Occlusion:** Use the same cones as diffuse reflection. Accumulate only the occlusion value $\alpha_2$, multiplied by weighting function $f(r) = \frac{1}{1+\lambda r}$, where $r$ is the current radius and $\lambda$ is a parameter [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Voxelization (Compute Shader)
def voxelize_geometry(geometry, flag_volume, albedo_vol, normal_vol, emission_vol):
    for triangle in geometry:
        voxels_hit = conservative_rasterize(triangle)
        for voxel_pos in voxels_hit:
            is_static = flag_volume.read(voxel_pos)
            
            if is_static:
                # Static voxelization only happens once, or we skip if already static
                # Assuming this call is for initial static setup
                pass 
            else:
                # Dynamic update
                # Clear previous dynamic data if needed (handled in clear step)
                pass
            
            # Atomic average
            frag_normal = compute_normal(triangle, voxel_pos)
            frag_albedo = compute_albedo(triangle, voxel_pos)
            frag_emission = compute_emission(triangle, voxel_pos)
            
            normal_vol.atomic_avg(voxel_pos, frag_normal)
            albedo_vol.atomic_avg(voxel_pos, frag_albedo)
            emission_vol.atomic_avg(voxel_pos, frag_emission)
            
            # Mark as static if this is the first write or based on logic
            # Note: Logic for marking static vs dynamic depends on implementation specifics
            # [G] Assuming a separate flag update step or initial pass sets static flags
            pass

def clear_dynamic_voxels(flag_volume, albedo_vol, normal_vol, emission_vol):
    for voxel_pos in grid:
        if flag_volume.read(voxel_pos) == 0: # Dynamic
            albedo_vol.write(voxel_pos, 0)
            normal_vol.write(voxel_pos, 0)
            emission_vol.write(voxel_pos, 0)

# Step 2: Voxel Illumination (Compute Shader)
def compute_voxel_radiance(albedo_vol, normal_vol, emission_vol, lights, radiance_vol, flag_volume):
    for voxel_pos in grid:
        if flag_volume.read(voxel_pos) == 0: # Skip empty or dynamic if not updated
            continue
            
        N = normal_vol.read(voxel_pos)
        rho = albedo_vol.read(voxel_pos)
        E = emission_vol.read(voxel_pos)
        
        radiance = 0.0
        
        for light in lights:
            Psi = normalize(light.position - voxel_pos) # Direction to light
            Li = light.intensity
            
            # Normal Weighted Attenuation
            D_x = dot(N, vec3(1,0,0)) # Simplified: using axes
            D_y = dot(N, vec3(0,1,0))
            D_z = dot(N, vec3(0,0,1))
            
            # Dominant faces
            N_x = sign(N.x)
            N_y = sign(N.y)
            N_z = sign(N.z)
            
            D_x_dom = max(D_x, 0) if N_x > 0 else max(-D_x, 0)
            D_y_dom = max(D_y, 0) if N_y > 0 else max(-D_y, 0)
            D_z_dom = max(D_z, 0) if N_z > 0 else max(-D_z, 0)
            
            W = N * N
            Vr = Li * (rho / PI) * (W.x * D_x_dom + W.y * D_y_dom + W.z * D_z_dom)
            
            # Occlusion
            nu = 0.0
            t = 0.0
            ray_pos = voxel_pos
            while t < max_distance:
                ray_pos += Psi * step_size
                if voxel_exists(ray_pos): # Sample voxel volume
                    nu = nu + (1 - nu) * kappa / t
                t += step_size
            
            radiance += Vr * (1 - nu)
            
        radiance += E
        radiance_vol.write(voxel_pos, radiance)

# Step 3: Anisotropic Voxels (Compute Shader)
def generate_anisotropic_voxels(radiance_vol, directional_volumes):
    for level in mip_levels:
        for voxel_pos in grid_at_level(level):
            # Volumetric integration in depth from previous level
            # [G] Specific integration logic from Crassin's method
            values = []
            for axis in [X_pos, X_neg, Y_pos, Y_neg, Z_pos, Z_neg]:
                val = integrate_direction(radiance_vol, voxel_pos, axis, level-1)
                values.append(val)
            
            for i, axis in enumerate([X_pos, X_neg, Y_pos, Y_neg, Z_pos, Z_neg]):
                directional_volumes[axis].write(voxel_pos, values[i])

# Step 4: Voxel Cone Tracing (Fragment Shader)
def fragment_shader(camera, radiance_vol, directional_volumes):
    # Screen space position to world space origin Co
    Co = screen_to_world(camera, gl_FragCoord)
    Cd = normalize(camera.direction)
    theta = camera.aperture_angle
    
    c = vec3(0.0)
    alpha = 0.0
    t = 0.0
    
    while t < max_trace_distance:
        d = 2 * t * tan(theta / 2)
        V_level = log2(d / V_size)
        
        # Determine which directional volumes to sample based on Cd
        # [G] Logic to select 3 dominant axes from 6 volumes
        selected_volumes = get_dominant_directional_volumes(Cd, directional_volumes)
        
        # Sample radiance and occlusion
        c2, alpha2 = sample_voxel_structure(Co, d, V_level, selected_volumes, radiance_vol)
        
        # Accumulate
        c = alpha * c + (1 - alpha) * alpha2 * c2
        alpha = alpha + (1 - alpha) * alpha2
        
        # Step size
        t += beta * d
        
    return c
```

### Parameters
*   **Beta ($\beta$):** Factor modifying the distance $d'$ between cone steps. $\beta = 1$ makes $d'$ equivalent to current diameter $d$. Values $< 1$ produce higher quality but require more samples [S2, unofficial].
*   **Aperture Angle ($\theta$):** Determines the spread of the cone. Depends on the specular exponent for specular reflections [S2, unofficial].
*   **Lambda ($\lambda$):** Parameter in the ambient occlusion weighting function $f(r) = \frac{1}{1+\lambda r}$ [S2, unofficial].

### Edge Cases and Pitfalls
*   **Normal Averaging:** Uneven normals within a voxel can cause incorrect attenuation. The normal-weighted attenuation method mitigates this [S2, unofficial].
*   **Dynamic Updates:** Static voxels must be protected from being overwritten by dynamic voxelization. The Flag Volume is essential for this [S2, unofficial].
*   **Precision:** Voxel cone tracing can suffer from precision issues at large distances or small apertures. Quadrilinear interpolation helps smooth variations [S2, unofficial].
*   **Performance:** High $\beta$ values reduce sample count but may introduce artifacts. Low $\beta$ values improve quality but increase computational cost [S2, unofficial].

## Subtopics

### Voxel Cone Tracing
The core technique of tracing cones through a voxel grid to approximate indirect lighting. It differs from ray tracing by using structured voxel data for faster traversal and interpolation. Choose when real-time global illumination is needed with moderate complexity. [Voxel Cone Tracing](1454-voxel-cone-tracing.md)

### Deferred Shading
A rendering technique that delays lighting calculations until after the geometry pass. It differs from forward rendering by avoiding redundant lighting computations for overlapping fragments. Choose when scenes have many light sources or complex geometry. [Deferred Shading](https://en.wikipedia.org/wiki/Deferred_shading) (External Link)

## Operating characteristics
*   **Time Complexity:** Dependent on the number of voxels, lights, and cone trace steps. Voxelization is $O(N)$ where $N$ is the number of triangles. Illumination is $O(V \times L)$ where $V$ is voxels and $L$ is lights. Cone tracing is $O(F \times S)$ where $F$ is fragments and $S$ is samples per cone.
*   **Memory Complexity:** $O(V)$ for the voxel structures. Six directional volumes at half resolution plus the radiance volume and flag volume.
*   **Measured Performance:** Enhanced performance in scenes with many light sources compared to forward rendering [S1, unofficial].
*   **Strengths:** Handles emissive materials, indirect shadows, and multiple light bounces. Compatible with point, spot, and directional lights. No shadow maps needed for voxel occlusion [S2, unofficial].
*   **Limitations:** Requires a deferred rendering backend. Quality depends on voxel resolution and cone trace steps. Artifacts can occur at boundaries or with high-frequency details.
*   **When to Use:** Real-time global illumination in deferred rendering pipelines.
*   **When Not to Use:** Forward rendering pipelines, scenes with extremely high-frequency details requiring higher resolution than feasible, or when strict memory constraints exist.

## Background
Deferred Voxel Shading was presented by Jose Villegas [S2, unofficial]. It expands upon rudimentary voxel cone tracing by merging it with deferred shading [S1, unofficial]. The technique was developed to provide a real-time solution for complex lighting effects including indirect diffuse, specular reflectance, color-blending, emissive materials, indirect shadows, and ambient occlusion [S2, unofficial]. It builds on the principles of voxel cone tracing [S1, unofficial] and deferred rendering [S1, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] Deferred Voxel Shading for Real Time Global Illumination, https://jose-villegas.github.io/post/deferred_voxel_shading/, fetched 2026-10-27 (unofficial)
