# Deferred Voxel Shading

Also called: Deferred Voxel Cone Tracing, Villegas Voxel Cone Tracing, Villegas Voxel Shading

## Overview
Deferred Voxel Shading is a real-time global illumination technique that merges voxel cone tracing with deferred shading to compute indirect lighting effects such as diffuse/specular reflection, color blending, emissive materials, and ambient occlusion [S2, unofficial]. Unlike forward voxel cone tracing, it delays lighting computations until the fragment shader stage, avoiding redundant calculations for overlapping geometry and enabling efficient handling of complex lighting scenarios with multiple light sources [S1, unofficial].

## How it works
The method consists of four distinct stages: Voxelization, Voxel Illumination, Anisotropic Voxel Generation, and Voxel Cone Tracing [S2, unofficial].

### Inputs and Outputs
*   **Inputs:** Scene geometry (vertices, normals, albedo, emission), light sources (point, spot, directional), camera parameters.
*   **Outputs:** Final screen-space color with indirect illumination.

### Data Structures
The technique relies on several 3D textures bound as `uimage3D` [S2, unofficial]:

1.  **Voxelization Storage:** Three 3D textures storing raw scene data.
    *   `voxelAlbedo`: Stores albedo values.
    *   `voxelNormal`: Stores surface normals.
    *   `voxelEmission`: Stores light emission values.
    *   Format: `r32ui` (unsigned integer, 32-bit) [S2, unofficial].
    *   Operations: Atomic averaging (`imageAtomicRGBA8Avg`) is used to combine values from overlapping fragments [S2, unofficial].

2.  **Radiance Volume:** A 3D texture storing the computed outgoing radiance per voxel (direct lighting + emission). This represents the maximum level of detail (LOD 0) for the voxelized scene [S2, unofficial].

3.  **Directional Volumes:** Six 3D textures, each at half the resolution of the Radiance Volume.
    *   One texture per axis direction (positive/negative X, Y, Z).
    *   Mipmap levels store directional values for anisotropic sampling [S2, unofficial].

4.  **Flag Volume:** A single-value 3D texture used to identify voxels as static. Used to prevent overwriting static voxels during dynamic updates [S2, unofficial].

### Step 1: Voxelization
The scene geometry is voxelized into the Voxelization Storage textures [S2, unofficial].

1.  **Conservative Voxelization:** Geometry is rasterized into 3D space.
2.  **Atomic Averaging:** For each voxel covered by a fragment, atomic operations average the albedo, normal, and emission values [S2, unofficial].
    ```python
    # Pseudocode for voxelization fragment/compute shader
    # position: voxel coordinates
    # normal: fragment normal
    # albedo: fragment albedo
    # emissive: fragment emission
    
    imageAtomicRGBA8Avg(voxelNormal, position, normal)
    imageAtomicRGBA8Avg(voxelAlbedo, position, albedo)
    imageAtomicRGBA8Avg(voxelEmission, position, emissive)
    ```
3.  **Dynamic Updates:**
    *   A Flag Volume tracks static voxels.
    *   Before writing a dynamic voxel, read the Flag Volume at the target position.
    *   If the flag indicates the voxel is static, discard the write [S2, unofficial].
    *   To clear previous dynamic voxels, use a compute shader to read the Flag Volume; clear voxels if they exist and are marked dynamic [S2, unofficial].

### Step 2: Voxel Illumination
Direct lighting is computed per voxel in a compute shader using the data from Voxelization Storage [S2, unofficial].

1.  **Position Extraction:** Extract the voxel's 3D coordinates and project them to world space to determine position [S2, unofficial].
2.  **Normal Retrieval:** Retrieve the averaged normal from `voxelNormal` [S2, unofficial].
3.  **Direct Lighting Calculation:** Compute outgoing radiance $V_r$ for each light source [S2, unofficial].

    **Normal Weighted Attenuation:**
    To handle uneven normals within a voxel, calculate attenuation per face:
    $$ D_{x,y,z} = (\hat{i}\cdot\Psi, \hat{j}\cdot\Psi, \hat{k}\cdot\Psi) $$
    where $\Psi$ is the light direction [S2, unofficial].

    Select dominant faces based on the sign of the voxel averaged normal vector component $N_\omega$:
    $$ D_{\omega} = \begin{cases} \max\{D_\omega, 0\}, & N_\omega>0 \\ \max\{-D_\omega, 0\}, & \text{otherwise} \end{cases} $$
    where $\omega \in \{x, y, z\}$ [S2, unofficial].

    **Reflectance Model:**
    $$ W = N^2 $$
    $$ V_{r} = L_{i}\frac{\rho}{\pi}(W_x D_x + W_y D_y + W_z D_z) $$
    where:
    *   $L_i$: Light source intensity.
    *   $\rho$: Voxel albedo.
    *   $N$: Normal vector of the voxel.
    *   $W_x, W_y, W_z$: Weights derived from $N^2$ [S2, unofficial].

4.  **Emission Addition:** Add the emission term from `voxelEmission` to $V_r$ [S2, unofficial].
5.  **Voxel Occlusion (Shadows):**
    *   Trace a ray from the voxel position in the direction of the light.
    *   Sample the voxelized volume to check for intersections.
    *   **Soft Shadows:** Accumulate value $\kappa$ per collision. Update occlusion $\nu$:
        $$ \nu = \nu + (1 - \nu)\kappa\div t $$
        where $t$ is the traced distance. The final occlusion value is $1 - \nu$ [S2, unofficial].
    *   Store the resulting radiance in the Radiance Volume [S2, unofficial].

### Step 3: Generate Anisotropic Voxels
Compute directional values for anisotropic filtering [S2, unofficial].

1.  **Volumetric Integration:** For each mipmap level, execute a compute shader thread per voxel.
2.  **Directional Values:** Compute six directional values (positive/negative X, Y, Z) by integrating in depth.
3.  **Averaging:** Average the directional values to obtain the resulting value for each direction.
4.  **Storage:** Store these values in the corresponding Directional Volumes. This process is repeated for each mipmap level, using values from the previous level [S2, unofficial].

### Step 4: Voxel Cone Tracing
Performed in the fragment shader as a screen-space effect [S2, unofficial].

1.  **Cone Definition:** Each cone is defined by origin $C_o$, direction $C_d$, and aperture angle $\theta$ [S2, unofficial].
2.  **Cone Diameter:** Calculate diameter $d$ at traced distance $t$:
    $$ d=2t\times\tan(\theta\div 2) $$
3.  **Mipmap Level Selection:** Determine the LOD to sample:
    $$ V_{level} = log_2(d\div V_{size}) $$
    where $V_{size}$ is the size of a voxel at maximum LOD [S2, unofficial].
4.  **Interpolation:** Use quadrilinear interpolation for smooth variation between samples [S2, unofficial].
5.  **Directional Sampling:**
    *   Determine which three Directional Volumes to sample based on the cone direction $C_d$.
    *   Linearly interpolate the three samples from the selected volumes [S2, unofficial].
6.  **Front-to-Back Accumulation:**
    *   Track accumulated color $c$ and occlusion $\alpha$.
    *   Retrieve outgoing radiance $c_2$ and occlusion $\alpha_2$ from the voxel structure (Radiance Volume or Directional Volumes, interpolated if $V_{level}$ is between max and first filtered level) [S2, unofficial].
    *   Update values:
        $$ c =\alpha c + (1-\alpha)\alpha_2c_2 $$
        $$ \alpha=\alpha+(1-\alpha)\alpha_2 $$
7.  **Step Size:** Modify the distance $d'$ between cone steps by a factor $\beta$:
    *   $\beta = 1$: $d' = d$ (current diameter).
    *   $\beta < 1$: Higher quality, more samples [S2, unofficial].

### Indirect Illumination Approximation
*   **Monte Carlo:** Partition the hemisphere integral into a sum of cone integrals [S2, unofficial].
*   **BRDF Matching (Blinn-Phong):**
    *   **Diffuse:** Use few large cones distributed over the normal-oriented hemisphere.
    *   **Specular:** Use a single cone in the reflected direction, with aperture depending on the specular exponent [S2, unofficial].
*   **Ambient Occlusion:** Approximated using the same cones as diffuse reflection. Accumulate only $\alpha_2$, weighted by $f(r) = \frac{1}{1+\lambda r}$, where $r$ is the current radius and $\lambda$ is a parameter [S2, unofficial].

## Operating characteristics
*   **Performance:** Enhanced performance in scenes with many light sources compared to forward rendering, as lighting is computed once per pixel rather than per light [S1, unofficial].
*   **Quality:** Anisotropic filtering and normal-weighted attenuation provide more precise results than basic voxel cone tracing [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires a deferred rendering backend. Dynamic voxelization requires careful management of static/dynamic flags to avoid artifacts [S2, unofficial].
*   **Complexity:** The method involves multiple compute shader passes and complex data structure management (six directional volumes, flag volume) [S2, unofficial].

## Background
Deferred Voxel Shading was presented by Jose Villegas [S1, unofficial]. It expands upon rudimentary voxel cone tracing by integrating deferred shading principles, allowing for the computation of complex lighting effects like emissive materials and multiple light bounces in real-time [S1, unofficial]. The technique is inspired by Crassin's work on anisotropic voxel generation [S2, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] Deferred Voxel Shading for Real Time Global Illumination, https://jose-villegas.github.io/post/deferred_voxel_shading/, fetched 2026-10-27 (unofficial)
