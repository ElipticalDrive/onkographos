# Deferred Voxel Shading

Also called: Deferred Voxel Cone Tracing, Villegas Voxel Cone Tracing, Villegas Voxel Shading

## Overview
Deferred Voxel Shading is a real-time global illumination technique that merges voxel cone tracing with deferred shading to compute indirect lighting effects such as diffuse/specular reflection, color blending, emissive materials, and ambient occlusion [S2, unofficial]. Unlike forward rendering approaches, it delays lighting computations until the fragment shader stage, avoiding redundant calculations for overlapping geometry and enabling efficient handling of complex lighting scenarios with multiple light sources [S1, unofficial]. It builds on [Voxel Cone Tracing](1454-voxel-cone-tracing.md) by introducing anisotropic voxel representations and deferred direct illumination integration.

## How it works
The method consists of four sequential stages: Voxelization, Voxel Illumination, Anisotropic Voxel Generation, and Voxel Cone Tracing [S2, unofficial].

### Inputs and Outputs
**Inputs:**
*   Scene geometry (triangles).
*   Material properties: Albedo ($\rho$), Surface Normal ($N$), Emission ($E$), Specular exponent (for BRDF matching).
*   Light sources: Point, Spot, Directional (with intensity $L_i$ and direction $\Psi$).
*   Camera parameters: Position, View Direction, Aperture angle $\theta$ for cone tracing.

**Outputs:**
*   Final fragment color with indirect illumination.
*   Updated 3D textures: Radiance Volume, Directional Volumes, Flag Volume.

### Data Structures
The voxel structure relies on several 3D textures bound as `uimage3D` (unsigned integer 32-bit) or standard 3D textures for interpolation [S2, unofficial]:

1.  **Voxelization Textures (Atomic Write):**
    *   `voxelAlbedo`: Stores averaged albedo per voxel. Format: `r32ui`.
    *   `voxelNormal`: Stores averaged surface normal per voxel. Format: `r32ui` (normalized).
    *   `voxelEmission`: Stores emission intensity per voxel. Format: `r32ui`.
    *   *Note:* Atomic operations (`imageAtomicRGBA8Avg`) are used to average values from fragments covering the same voxel space [S2, unofficial].

2.  **Radiance Volume:**
    *   A 3D texture storing the maximum level of detail (LOD) for the voxelized scene. It holds the result of direct lighting computations per voxel [S2, unofficial].

3.  **Directional Volumes (Anisotropic):**
    *   Six 3D textures at half resolution of the Radiance Volume.
    *   One texture per axis direction: $+X, -X, +Y, -Y, +Z, -Z$.
    *   Mipmap levels store directional values for anisotropic sampling [S2, unofficial].

4.  **Flag Volume:**
    *   A single-value 3D texture used to identify voxels as static. Used to prevent overwriting static voxels during dynamic updates [S2, unofficial].

### Numbered Steps

#### Step 1: Voxelization
1.  **Conservative Rasterization:** Project scene geometry into a 3D grid.
2.  **Atomic Averaging:** For each voxel covered by a fragment, update the three voxelization textures using atomic averaging to handle overlapping fragments [S2, unofficial]:
    ```python
    # Pseudocode for atomic update
    imageAtomicRGBA8Avg(voxelNormal, voxel_pos, normal)
    imageAtomicRGBA8Avg(voxelAlbedo, voxel_pos, albedo)
    imageAtomicRGBA8Avg(voxelEmission, voxel_pos, emission)
    ```
3.  **Dynamic Update Handling:**
    *   Read `flag_volume[voxel_pos]`.
    *   If the flag indicates the position is static, discard writing [S2, unofficial].
    *   To clear previous dynamic voxels, read the flag volume; if a voxel exists and is dynamic, clear its entry in the voxelization textures [S2, unofficial].
    *   After static voxelization, write to `flag_volume` to mark positions as static [S2, unofficial].

#### Step 2: Voxel Illumination (Direct Lighting)
Compute outgoing radiance per voxel using a compute shader. This step calculates direct lighting for all light types (point, spot, directional) [S2, unofficial].

1.  **Extract Voxel Data:** Retrieve averaged normal $N$ and albedo $\rho$ from voxelization textures. Calculate voxel position in world space.
2.  **Normal Weighted Attenuation:** To handle uneven normals within a voxel, calculate attenuation per face based on light direction $\Psi$:
    $$ D_{x,y,z} = (\hat{i}\cdot\Psi, \hat{j}\cdot\Psi, \hat{k}\cdot\Psi) $$
    Select dominant faces based on the sign of the voxel averaged normal vector component $N_\omega$:
    $$ D_{\omega} = \begin{cases} \max\{D_\omega, 0\}, & N_\omega>0 \\ \max\{-D_\omega, 0\}, & \text{otherwise} \end{cases} $$
3.  **Reflectance Model:** Compute the direct radiance contribution $V_r$:
    $$ W = N^2 $$
    $$ V_{r} = L_{i}\frac{\rho}{\pi}(W_x D_x + W_y D_y + W_z D_z) $$
    Where $L_i$ is light intensity, $\rho$ is voxel albedo, and $N$ is the voxel normal vector [S2, unofficial].
4.  **Emission Handling:** Add the emission term from the voxelization process to $V_r$. This allows emissive materials to contribute indirect light even in occluded areas [S2, unofficial].
5.  **Voxel Occlusion (Soft Shadows):**
    *   Trace a ray from the voxel position in the direction of the light $\Psi$.
    *   Sample the voxelized volume (any of the voxelization textures) to check for intersections.
    *   Accumulate soft shadows using:
        $$ \nu = \nu + (1 - \nu)\kappa\div t $$
        Where $\kappa$ is the value accumulated per collision and $t$ is the traced distance. The final occlusion value is $1 - \nu$ [S2, unofficial].
6.  **Store Result:** Write the computed radiance $V_r$ (modified by occlusion) to the Radiance Volume [S2, unofficial].

#### Step 3: Generate Anisotropic Voxels
Generate directional volumes to support anisotropic cone tracing [S2, unofficial].

1.  **Volumetric Integration:** For each mipmap level (from highest to lowest resolution):
    *   Execute a compute shader thread per voxel.
    *   Perform volumetric integration in depth to compute directional values.
    *   Average directional values to obtain the resulting value for a specific direction [S2, unofficial].
2.  **Storage:** Store the six directional values (one per axis direction) in the corresponding Directional Volume textures. Mipmap levels store the levels of detail for anisotropic sampling [S2, unofficial].

#### Step 4: Voxel Cone Tracing (Indirect Lighting)
Performed in the fragment shader as a screenspace effect. A deferred rendering backend is required [S2, unofficial].

1.  **Cone Definition:** Define a cone by origin $C_o$, direction $C_d$, and aperture angle $\theta$.
2.  **Cone Diameter:** Calculate the diameter $d$ at traced distance $t$:
    $$ d=2t\times\tan(\theta\div 2) $$
3.  **Mipmap Level Selection:** Determine the LOD to sample:
    $$ V_{level} = log_2(d\div V_{size}) $$
    Where $V_{size}$ is the size of a voxel at the maximum level of detail [S2, unofficial].
4.  **Ray Marching:** Advance the cone along $C_d$.
    *   **Step Size:** Modify the distance $d'$ between steps by a factor $\beta$. $\beta = 1$ makes $d'$ equivalent to current diameter $d$. Values $< 1$ produce higher quality but require more samples [S2, unofficial].
    *   **Sampling:**
        *   If $V_{level}$ is within the maximum LOD, sample the Radiance Volume.
        *   If $V_{level}$ is between the maximum LOD and the first filtered LOD, use linear interpolation between samples from the Radiance Volume and Directional Volumes [S2, unofficial].
        *   Use quadrilinear interpolation for smooth variation [S2, unofficial].
    *   **Accumulation:** Update accumulated color $c$ and occlusion $\alpha$ using volumetric front-to-back accumulation:
        $$ c =\alpha c + (1-\alpha)\alpha_2c_2 $$
        $$ \alpha=\alpha+(1-\alpha)\alpha_2 $$
        Where $c_2$ and $\alpha_2$ are the retrieved outgoing radiance and occlusion from the voxel structure [S2, unofficial].
5.  **BRDF Matching:**
    *   **Diffuse:** Use a few large cones distributed over the normal-oriented hemisphere.
    *   **Specular:** Use a single cone in the reflected direction, where aperture depends on the specular exponent [S2, unofficial].
6.  **Ambient Occlusion:** Approximate using the same cones as diffuse reflection. Accumulate occlusion value $\alpha_2$ multiplied by weighting function $f(r) = \frac{1}{1+\lambda r}$, where $r$ is the current radius and $\lambda$ is a parameter [S2, unofficial].

### Parameters
*   **$\beta$ (Beta):** Factor modifying the distance $d'$ between cone steps. $\beta = 1$ is standard; $\beta < 1$ increases quality at the cost of performance [S2, unofficial].
*   **$\theta$ (Theta):** Aperture angle of the cone. Determines the spread of the sampling volume [S2, unofficial].
*   **$\lambda$:** Parameter for ambient occlusion weighting function [S2, unofficial].

### Edge Cases and Pitfalls
*   **Normal Averaging:** Uneven normals within a voxel can cause artifacts. The normal-weighted attenuation model mitigates this [S2, unofficial].
*   **Dynamic Updates:** Static voxels must be protected from overwriting. The Flag Volume mechanism is critical for this [S2, unofficial].
*   **Precision:** Quadrilinear interpolation is used to ensure smooth variation, but coarse voxelization may lead to blocky artifacts.
*   **Performance:** Cone tracing is expensive. The number of steps and $\beta$ value significantly impact performance.

## Operating characteristics
*   **Time Complexity:** Depends on the number of lights (for voxel illumination) and the number of cone steps (for cone tracing). Cone tracing is a screenspace effect, so it scales with screen resolution and cone aperture.
*   **Memory Complexity:** Requires multiple 3D textures: 3 for voxelization, 1 for radiance, 6 for directional volumes, and 1 for flags. Resolution is typically lower than screen space to manage memory.
*   **Strengths:** Handles complex lighting effects (emissive, indirect shadows, multiple bounces) in real-time. Deferred shading avoids redundant lighting calculations. Compatible with standard light types.
*   **Limitations:** Requires a deferred rendering pipeline. Voxelization can be memory-intensive. Approximations (cone tracing, soft shadows) may lack the precision of path tracing.
*   **When to Use:** Scenes with many light sources, emissive materials, and where real-time global illumination is desired.
*   **When Not to Use:** Forward rendering pipelines, scenes with very high-frequency geometry details that cannot be captured by the voxel resolution.

## Background
Deferred Voxel Shading was presented by Jose Villegas [S2, unofficial]. It expands upon rudimentary voxel cone tracing by merging it with deferred shading techniques [S1, unofficial]. The approach allows for the computation of lighting effects such as emissive materials, indirect shadows, and multiple light bounces on a voxel cone tracing basis. It was developed to provide enhanced performance in scenes with a vast amount of light sources by delaying lighting computations until the fragment shader stage [S1, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] Deferred Voxel Shading for Real Time Global Illumination, https://jose-villegas.github.io/post/deferred_voxel_shading/, fetched 2026-10-27 (unofficial)
