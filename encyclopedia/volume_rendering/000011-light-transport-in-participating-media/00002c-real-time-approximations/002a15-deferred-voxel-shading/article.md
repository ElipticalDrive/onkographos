# Deferred Voxel Shading

Also called: Deferred Voxel Cone Tracing, Villegas Voxel Cone Tracing, Villegas Voxel Shading

## Overview
Deferred Voxel Shading is a real-time global illumination technique that merges voxel cone tracing with deferred shading to approximate indirect lighting effects such as diffuse/specular reflections, color bleeding, emissive materials, and ambient occlusion [S2, unofficial]. Unlike forward voxel cone tracing, which computes lighting per fragment during geometry rendering, this method delays lighting calculations to a post-processing stage, avoiding redundant computations for overlapping polygons and enhancing performance in scenes with many light sources [S1, unofficial]. It builds on [Real-time approximations](36-real-time-approximations.md) by utilizing spatial discretization and precomputed voxel structures to evaluate the radiative transfer equation efficiently [S1, S2, unofficial].

## How it works
The method consists of four primary stages: Voxelization, Voxel Illumination, Anisotropic Voxel Generation, and Voxel Cone Tracing [S2, unofficial].

### Inputs and Outputs
**Inputs:**
*   Scene geometry (triangles).
*   Material properties: Albedo ($\rho$), Surface Normal ($N$), Emission ($E$), Specular Exponent ($n$).
*   Light sources: Point, Spot, Directional (with intensity $L_i$ and direction $\Psi$).
*   Camera parameters: Position, View Matrix, Projection Matrix.

**Outputs:**
*   Screen-space indirect illumination buffer (integrated into the deferred shading pipeline).

### Data Structures
The technique relies on several 3D textures bound as `uimage3D` (unsigned integer) or standard 3D textures for interpolation [S2, unofficial].

1.  **Voxelization Storage:** Three 3D textures storing raw scene data.
    *   `voxelAlbedo`: Stores averaged albedo values.
    *   `voxelNormal`: Stores averaged surface normals.
    *   `voxelEmission`: Stores emission values.
    *   *Format:* `r32ui` (or similar unsigned integer format supporting atomic operations).
    *   *Access:* Atomic operations (e.g., `imageAtomicRGBA8Avg`) are used to average values from multiple fragments covering the same voxel [S2, unofficial].

2.  **Flag Volume:** A single-value 3D texture.
    *   *Purpose:* Identifies voxels as static. Used during dynamic updates to prevent overwriting static geometry [S2, unofficial].

3.  **Radiance Volume:** A 3D texture storing the maximum level of detail (LOD) for the voxelized scene.
    *   *Content:* Direct lighting computations per voxel (outgoing radiance).
    *   *Purpose:* Represents the base lighting data before anisotropic filtering [S2, unofficial].

4.  **Directional Volumes:** Six 3D textures, each at half the resolution of the Radiance Volume.
    *   *Content:* One texture per axis direction (positive/negative X, Y, Z).
    *   *Mipmaps:* Store levels of detail for anisotropic sampling. The mipmap levels correspond to the directional coherence of the lighting [S2, unofficial].

### Numbered Steps

#### 1. Voxelization
The scene is voxelized using conservative rasterization to ensure no geometry is missed [S2, unofficial].
1.  For each triangle, determine the bounding box in voxel space.
2.  Iterate through all voxels within the bounding box.
3.  For each voxel, if it is occupied by the triangle:
    *   Read the current value from `voxelAlbedo`, `voxelNormal`, and `voxelEmission` at the voxel position.
    *   Average the new fragment's albedo, normal, and emission with the stored values using atomic operations [S2, unofficial].
    *   If the geometry is static, write `1` to the `Flag Volume` at this position. If dynamic, write `0` (or leave as is if clearing logic handles it) [S2, unofficial].
4.  **Dynamic Update:** Before voxelizing dynamic geometry, clear previous dynamic voxels. Read the `Flag Volume`; if a voxel exists and is marked dynamic (flag is 0 or specific dynamic marker), clear its data in the three storage textures [S2, unofficial].

#### 2. Voxel Illumination
Direct lighting is computed per voxel in a compute shader using the data from the Voxelization stage [S2, unofficial].
1.  Iterate through all voxels in the grid.
2.  Retrieve the averaged Normal ($N$), Albedo ($\rho$), and Emission ($E$) for the voxel.
3.  Calculate the voxel's world-space position from its 3D coordinates.
4.  For each light source:
    *   Determine light direction $\Psi$ and intensity $L_i$.
    *   Compute **Normal Weighted Attenuation** to handle uneven normals within a voxel:
        *   Calculate face attenuations:
            $$ D_{x,y,z} = (\hat{i}\cdot\Psi, \hat{j}\cdot\Psi, \hat{k}\cdot\Psi) $$
            where $\hat{i}, \hat{j}, \hat{k}$ are the axis unit vectors and $\Psi$ is the light direction [S2, unofficial].
        *   Select dominant faces based on the sign of the voxel averaged normal vector components ($N_\omega$):
            $$ D_{\omega} = \begin{cases} \max\{D_\omega, 0\}, & N_\omega > 0 \\ \max\{-D_\omega, 0\}, & \text{otherwise} \end{cases} $$
            where $\omega \in \{x, y, z\}$ [S2, unofficial].
        *   Compute weights $W = N^2$ (component-wise square of the normal vector).
        *   Calculate reflected radiance contribution $V_r$:
            $$ V_{r} = L_{i}\frac{\rho}{\pi}(W_x D_x + W_y D_y + W_z D_z) $$
    *   **Voxel Occlusion:** Trace a ray from the voxel position in direction $\Psi$. Sample the voxelized volume (e.g., using `voxelAlbedo` or a dedicated occupancy texture). If a voxel exists at the ray position, the voxel is occluded.
        *   **Soft Shadows:** Instead of binary occlusion, accumulate soft shadow value $\nu$:
            $$ \nu = \nu + (1 - \nu)\kappa\div t $$
            where $\kappa$ is the value accumulated per collision and $t$ is the traced distance. The final occlusion factor is $1 - \nu$ [S2, unofficial].
    *   Update the voxel's radiance in the **Radiance Volume** with the computed $V_r$ (modulated by occlusion) and add the voxel's Emission term $E$ [S2, unofficial].

#### 3. Generate Anisotropic Voxels
To support precise cone tracing, directional information is stored in mipmaps of the **Directional Volumes** [S2, unofficial].
1.  For each mipmap level (from base to max):
    *   For each of the six directional volumes (±X, ±Y, ±Z):
        *   Execute a compute shader thread per voxel.
        *   Perform volumetric integration in depth using values from the previous mipmap level (or the Radiance Volume for the base level).
        *   Average the directional values to obtain the resulting value for that specific axis direction.
        *   Store the result in the corresponding Directional Volume at the current mipmap level [S2, unofficial].

#### 4. Voxel Cone Tracing
Performed in the fragment shader as a screen-space effect. It approximates indirect illumination using a crude Monte Carlo method where the hemisphere integral is partitioned into cones [S2, unofficial].

**Parameters:**
*   $\theta$: Aperture angle of the cone.
*   $\beta$: Step size factor ($\beta=1$ means step size equals current diameter; $\beta < 1$ increases quality but requires more samples) [S2, unofficial].

**Algorithm:**
1.  **Cone Definition:** Define a cone with origin $C_o$ (fragment position), direction $C_d$ (view direction or BRDF lobe direction), and aperture $\theta$.
2.  **BRDF Matching:**
    *   **Diffuse:** Use a few large cones distributed over the normal-oriented hemisphere.
    *   **Specular:** Use a single cone in the reflected direction, where aperture $\theta$ depends on the specular exponent [S2, unofficial].
3.  **Trace Loop:**
    *   Initialize accumulated color $c = 0$ and accumulated occlusion $\alpha = 0$.
    *   Initialize traced distance $t = 0$.
    *   While $t$ is within max distance:
        *   Calculate current cone diameter:
            $$ d = 2t \times \tan(\theta \div 2) $$
        *   Determine mipmap level to sample:
            $$ V_{level} = \log_2(d \div V_{size}) $$
            where $V_{size}$ is the size of a voxel at the maximum level of detail [S2, unofficial].
        *   **Sample Directional Volume:**
            *   Select the three directional volumes corresponding to the cone's dominant axes.
            *   Retrieve outgoing radiance $c_2$ and occlusion $\alpha_2$ from the selected Directional Volume at $V_{level}$ using quadrilinear interpolation for smooth variation [S2, unofficial].
        *   **Volumetric Front-to-Back Accumulation:**
            $$ c = \alpha c + (1-\alpha)\alpha_2 c_2 $$
            $$ \alpha = \alpha + (1-\alpha)\alpha_2 $$
            where $c$ and $\alpha$ are the accumulated color and occlusion, and $c_2$ and $\alpha_2$ are the retrieved values [S2, unofficial].
        *   Advance $t$ by step size $d' = \beta \times d$.
4.  **Ambient Occlusion (Optional):** For AO, accumulate only $\alpha_2$ weighted by $f(r) = \frac{1}{1+\lambda r}$, where $r$ is the current radius and $\lambda$ is a falloff parameter [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Deferred Voxel Shading Implementation

# Data Structures
class VoxelGrid:
    def __init__(self, size):
        self.size = size
        self.voxel_albedo = Image3D(size, format='r32ui')
        self.voxel_normal = Image3D(size, format='r32ui')
        self.voxel_emission = Image3D(size, format='r32ui')
        self.flag_volume = Image3D(size, format='r32ui') # 1 for static, 0 for dynamic
        self.radiance_volume = Image3D(size, format='rgba32f')
        self.directional_volumes = [Image3D(size/2, format='rgba32f') for _ in range(6)] # ±X, ±Y, ±Z

# Step 1: Voxelization
def voxelize_scene(geometry, is_static):
    for triangle in geometry:
        bbox = get_voxel_bbox(triangle)
        for voxel_pos in bbox:
            # Check flag for dynamic updates
            if not is_static:
                if read_flag(voxel_pos) == 1: # Static
                    continue
            
            # Conservative rasterization: determine if triangle covers voxel
            if triangle_intersects_voxel(triangle, voxel_pos):
                # Atomic average
                frag_albedo = get_fragment_albedo(triangle, voxel_pos)
                frag_normal = get_fragment_normal(triangle, voxel_pos)
                frag_emission = get_fragment_emission(triangle, voxel_pos)
                
                atomic_avg(self.voxel_albedo, voxel_pos, frag_albedo)
                atomic_avg(self.voxel_normal, voxel_pos, frag_normal)
                atomic_avg(self.voxel_emission, voxel_pos, frag_emission)
                
                if is_static:
                    write_flag(voxel_pos, 1)

# Step 2: Voxel Illumination
def compute_voxel_lighting():
    for voxel_pos in grid:
        N = read_normalized(self.voxel_normal, voxel_pos)
        rho = read(self.voxel_albedo, voxel_pos)
        E = read(self.voxel_emission, voxel_pos)
        pos_world = voxel_to_world(voxel_pos)
        
        radiance = E # Start with emission
        
        for light in lights:
            Psi = normalize(light.position - pos_world) # For point/spot
            Li = light.intensity
            
            # Normal Weighted Attenuation
            D_x = dot(vec3(1,0,0), Psi)
            D_y = dot(vec3(0,1,0), Psi)
            D_z = dot(vec3(0,0,1), Psi)
            
            # Dominant faces
            D_x_dom = max(D_x, 0) if N.x > 0 else max(-D_x, 0)
            D_y_dom = max(D_y, 0) if N.y > 0 else max(-D_y, 0)
            D_z_dom = max(D_z, 0) if N.z > 0 else max(-D_z, 0)
            
            W_x = N.x ** 2
            W_y = N.y ** 2
            W_z = N.z ** 2
            
            Vr = Li * (rho / pi) * (W_x * D_x_dom + W_y * D_y_dom + W_z * D_z_dom)
            
            # Occlusion (Soft Shadow)
            nu = 0.0
            t = 0.0
            max_dist = 10.0 # Example max shadow distance
            step = 0.1
            while t < max_dist:
                ray_pos = pos_world + Psi * t
                voxel_ray_pos = world_to_voxel(ray_pos)
                if is_occupied(voxel_ray_pos):
                    nu = nu + (1 - nu) * 1.0 / t # kappa=1 for single collision
                t += step
            
            occlusion = 1 - nu
            radiance += Vr * occlusion
            
        write(self.radiance_volume, voxel_pos, radiance)

# Step 3: Anisotropic Voxels
def generate_anisotropic_voxels():
    for level in range(0, max_mip_levels):
        for dir_idx in range(6):
            for voxel_pos in grid:
                # Volumetric integration from previous level
                prev_val = read_mipmap(self.directional_volumes[dir_idx], voxel_pos, level-1) if level > 0 else read(self.radiance_volume, voxel_pos)
                # Integrate along depth axis for this direction
                dir_val = integrate_depth(prev_val, voxel_pos, dir_idx)
                write_mipmap(self.directional_volumes[dir_idx], voxel_pos, level, dir_val)

# Step 4: Voxel Cone Tracing (Fragment Shader)
def trace_cone(origin, direction, theta, beta):
    c = vec3(0)
    alpha = 0.0
    t = 0.0
    V_size = 1.0 # Voxel size at LOD 0
    
    while t < max_trace_distance:
        d = 2 * t * tan(theta / 2)
        V_level = log2(d / V_size)
        
        # Select directional volumes based on cone direction
        # Simplified: assume we pick the 3 dominant axes
        samples = []
        for axis in dominant_axes(direction):
            val = read_mipmap_quadrilinear(self.directional_volumes[axis], origin, V_level)
            samples.append(val)
        
        # Interpolate samples to get c2 and alpha2
        c2, alpha2 = interpolate_directional_samples(samples)
        
        # Accumulate
        c = alpha * c + (1 - alpha) * alpha2 * c2
        alpha = alpha + (1 - alpha) * alpha2
        
        t += beta * d
        
    return c
```

### Edge Cases and Pitfalls
*   **Precision:** Atomic averaging of normals requires careful normalization after accumulation to prevent drift [S2, unofficial].
*   **Boundaries:** Cone tracing must handle cases where the cone extends beyond the voxel grid boundaries; samples should be clamped or treated as empty [S2, unofficial].
*   **Degenerate Input:** Voxels with zero normal (empty) should be skipped during illumination and cone tracing to avoid division by zero or invalid samples [S2, unofficial].
*   **Performance:** The number of cone steps is inversely proportional to $\beta$. Low $\beta$ values significantly increase fragment shader cost [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Voxelization is $O(N)$ where $N$ is the number of triangles (conservative rasterization). Voxel Illumination is $O(V \cdot L)$ where $V$ is the number of voxels and $L$ is the number of lights. Cone Tracing is $O(F \cdot S)$ where $F$ is the number of fragments and $S$ is the number of cone steps (dependent on $\beta$ and aperture) [S1, S2, unofficial].
*   **Memory Complexity:** Requires storage for 3 base textures, 1 flag volume, 1 radiance volume, and 6 directional volumes (mipmapped). Memory usage scales with the voxel grid resolution and the number of mipmap levels [S2, unofficial].
*   **Strengths:** Handles complex lighting effects (emissive, indirect shadows, color bleeding) in real-time. Deferred nature avoids overdraw penalties for lighting [S1, unofficial]. Compatible with standard light types without shadow maps [S2, unofficial].
*   **Limitations:** High memory footprint due to multiple 3D textures. Quality depends on voxel resolution; low resolution leads to blocky artifacts. Cone tracing can be expensive if many samples are required [S1, S2, unofficial].
*   **When to use:** Scenes with dynamic geometry and complex lighting where real-time global illumination is desired, and hardware supports efficient 3D texture sampling and atomic operations.
*   **When not to use:** Memory-constrained environments, or scenes where high-frequency lighting details are critical and voxel resolution cannot be increased [S2, unofficial].

## Background
Deferred Voxel Shading was introduced by Jose Villegas as an expansion upon rudimentary voxel cone tracing [S1, unofficial]. It merges voxel cone tracing with deferred shading to handle complex lighting effects like emissive materials and multiple light bounces [S1, unofficial]. The technique was presented in a blog post detailing a real-time global illumination implementation [S2, unofficial]. It builds on the principles of voxel cone tracing (Crassin) and deferred rendering, aiming to provide a comprehensive set of indirect illumination effects in real-time applications [S1, S2, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] Deferred Voxel Shading for Real Time Global Illumination, https://jose-villegas.github.io/post/deferred_voxel_shading/, fetched 2026-10-27 (unofficial)
