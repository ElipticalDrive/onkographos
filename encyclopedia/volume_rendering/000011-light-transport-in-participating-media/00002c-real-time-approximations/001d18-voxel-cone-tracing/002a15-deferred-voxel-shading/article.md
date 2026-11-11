# Deferred Voxel Shading

Also called: Deferred Voxel Cone Tracing, Villegas Voxel Cone Tracing, Villegas Voxel Shading

## Overview
Deferred Voxel Shading is a real-time global illumination technique that merges voxel cone tracing with deferred shading to compute indirect lighting effects such as diffuse/specular reflection, color blending, emissive materials, and shadows [S2, unofficial]. It solves the problem of redundant lighting calculations in scenes with many light sources by delaying lighting computations until after all geometry is rendered, allowing voxel cone tracing to be performed as a screenspace post-process [S1, unofficial]. Unlike forward voxel cone tracing, it integrates direct lighting into the voxel structure during voxelization, enabling emissive surfaces to contribute to indirect illumination even when occluded [S2, unofficial].

## How it works
The method consists of four main stages: Voxelization, Voxel Illumination, Anisotropic Voxel Generation, and Voxel Cone Tracing [S2, unofficial].

### Inputs and Outputs
**Inputs:**
*   Scene geometry (vertices, normals, UVs).
*   Material properties: Albedo ($\rho$), Normal ($N$), Emission ($E$), Specular exponent.
*   Light sources: Point, Spot, Directional (with intensity $L_i$ and direction $\Psi$).
*   Camera parameters: Position, View Matrix, Projection Matrix.

**Outputs:**
*   Final screen color with indirect illumination.

### Data Structures
The voxel structure consists of several 3D textures bound as `uimage3D` [S2, unofficial]:

1.  **Voxelization Storage:** Three 3D textures with `r32ui` format (or equivalent integer/float formats depending on implementation, but sources specify `r32ui` for atomic operations) [S2, unofficial]:
    *   `voxelAlbedo`: Stores averaged albedo.
    *   `voxelNormal`: Stores averaged surface normals.
    *   `voxelEmission`: Stores emission values.
2.  **Radiance Volume:** A 3D texture storing the maximum level of detail (LOD) for the voxelized scene, containing the computed direct lighting radiance per voxel [S2, unofficial].
3.  **Directional Volumes:** Six 3D textures at half the resolution of the radiance volume. One for each axis direction (positive/negative X, Y, Z). Mipmap levels store directional values for anisotropic sampling [S2, unofficial].
4.  **Flag Volume:** A single-value 3D texture used to identify voxels as static. Used to prevent overwriting static voxels during dynamic updates [S2, unofficial].

### Parameters
*   $\beta$: Factor modifying the distance $d'$ between cone steps. $\beta = 1$ makes $d'$ equivalent to the current cone diameter $d$. Values $< 1$ produce higher quality but require more samples [S2, unofficial].
*   $\theta$: Aperture angle of the cone.
*   $\lambda$: Weighting factor for ambient occlusion (mentioned in AO section, value not specified in sources) [S2, unofficial].

### Steps

#### 1. Voxelization
Conservative voxelization is used to generate the initial 3D textures [S2, unofficial]. Atomic operations are used to average values per voxel for fragments surrounding the voxel volume [S2, unofficial].

*   **Static vs Dynamic:** A flag volume tracks static voxels. During dynamic updates, if a voxel position is marked static in the flag volume, writing is discarded [S2, unofficial]. Previous dynamic voxels are cleared using compute shaders by checking the flag volume [S2, unofficial].
*   **Atomic Averaging:**
    ```python
    # Pseudocode for atomic averaging in compute shader
    # position: ivec3 voxel coordinate
    # normal: vec3 fragment normal
    # albedo: vec3 fragment albedo
    # emissive: vec3 fragment emission
    
    imageAtomicRGBA8Avg(voxelNormal, position, normal);
    imageAtomicRGBA8Avg(voxelAlbedo, position, albedo);
    imageAtomicRGBA8Avg(voxelEmission, position, emissive);
    ```
    [S2, unofficial]

#### 2. Voxel Illumination
Direct lighting is computed per voxel in a compute shader using the stored normal and position [S2, unofficial]. The position is extracted by projecting voxel 3D coordinates to world space [S2, unofficial].

*   **Normal Weighted Attenuation:** To handle uneven normals within a voxel, attenuation is calculated per face [S2, unofficial]:
    $$D_{x,y,z} = (\hat{i}\cdot\Psi, \hat{j}\cdot\Psi, \hat{k}\cdot\Psi)$$
    where $\Psi$ is the light direction [S2, unofficial].
    
    Dominant faces are selected based on the sign of the voxel averaged normal vector component $N_\omega$:
    $$D_{\omega} = \begin{cases} \max\{D_\omega, 0\}, & N_\omega>0 \\ \max\{-D_\omega, 0\}, & \text{otherwise} \end{cases}$$
    [S2, unofficial]

*   **Reflectance Model:** The outgoing radiance $V_r$ is computed as:
    $$W = N^2$$
    $$V_{r} = L_{i}\frac{\rho}{\pi}(W_x D_x + W_y D_y + W_z D_z)$$
    where $L_i$ is light intensity, $\rho$ is voxel albedo, and $N$ is the voxel normal vector [S2, unofficial].

*   **Emission:** The emission term from voxelization is added to the direct illumination value per voxel [S2, unofficial].

*   **Voxel Occlusion (Soft Shadows):** A ray is traced from the voxel position in the direction of the light. Soft shadows are approximated by accumulating a value $\kappa$ per collision and dividing by traced distance $t$:
    $$\nu = \nu + (1 - \nu)\kappa\div t$$
    where $1 - \nu$ represents the final occlusion value [S2, unofficial].

#### 3. Generate Anisotropic Voxels
Anisotropic filtering is used to generate mipmaps for more precise cone trace results [S1, unofficial]. Volumetric integration in depth is used to compute directional values [S2, unofficial].

*   A compute shader executes one thread per voxel at each mipmap level, using values from the previous level [S2, unofficial].
*   Six directional volumes store values for positive/negative axes. The directional sample is obtained by linearly interpolating the three samples from the selected directional volumes based on the cone direction [S2, unofficial].

#### 4. Voxel Cone Tracing
Performed in the fragment shader as a screenspace effect [S2, unofficial].

*   **Cone Definition:** Each cone is defined by origin $C_o$, direction $C_d$, and aperture angle $\theta$ [S2, unofficial].
*   **Cone Diameter:**
    $$d=2t\times\tan(\theta\div 2)$$
    where $t$ is the traced distance [S2, unofficial].
*   **Mipmap Level Selection:**
    $$V_{level} = log_2(d\div V_{size})$$
    where $V_{size}$ is the size of a voxel at the maximum level of detail [S2, unofficial].
    Linear interpolation is used between samples of the radiance volume and directional volumes when the required mipmap level ranges between the maximum level and the first filtered level of detail [S2, unofficial].
*   **Volumetric Front-to-Back Accumulation:**
    In each step, retrieve occlusion $\alpha_2$ and outgoing radiance $c_2$ from the voxel structure. Update accumulated color $c$ and occlusion $\alpha$:
    $$c =\alpha c + (1-\alpha)\alpha_2c_2$$
    $$\alpha=\alpha+(1-\alpha)\alpha_2$$
    [S2, unofficial]
*   **Step Size:** The distance $d'$ between steps is modified by $\beta$:
    $$d' = \beta \times d$$
    [S2, unofficial]
*   **Indirect Illumination Approximation:** A crude Monte Carlo approximation partitions the hemisphere integral into cones [S2, unofficial].
    *   **Diffuse:** Few large cones distributed over the normal-oriented hemisphere [S2, unofficial].
    *   **Specular:** A single cone in the reflected direction, where aperture depends on the specular exponent [S2, unofficial].
*   **Ambient Occlusion:** Approximated using the same cones as diffuse reflection. The accumulated occlusion value $\alpha_2$ is multiplied with a weighting function $f(r) = \frac{1}{1+\lambda r}$, where $r$ is the current radius of the cone [S2, unofficial].

### Edge Cases and Pitfalls
*   **Normal Averaging:** Uneven normal vectors within a voxel space can cause artifacts. The normal-weighted attenuation method mitigates this [S2, unofficial].
*   **Dynamic Updates:** Static voxels must be protected from overwriting by dynamic geometry using the flag volume [S2, unofficial].
*   **Performance:** Values of $\beta < 1$ increase quality but require more samples, reducing performance [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of cone steps, which is influenced by $\beta$ and scene density. Voxelization is $O(N)$ where $N$ is the number of fragments. Cone tracing is performed per fragment in the fragment shader [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** Requires storage for multiple 3D textures: Albedo, Normal, Emission, Radiance, 6 Directional Volumes, and Flag Volume. Resolution of directional volumes is half that of the radiance volume [S2, unofficial].
*   **Strengths:** Handles complex lighting effects including emissive materials and multiple light bounces. Enhanced performance in scenes with many light sources due to deferred nature. No shadow maps needed for voxel occlusion [S1, unofficial][S2, unofficial].
*   **Limitations:** Approximate results. Crude Monte Carlo approximation for indirect lighting. Requires deferred rendering backend for efficiency [S1, unofficial][S2, unofficial].
*   **When to use:** Scenes with many light sources, emissive materials, and where real-time global illumination is desired [S1, unofficial][S2, unofficial].
*   **When not to use:** Forward rendering pipelines where deferred shading is not feasible, or when exact shadow precision is required without additional techniques [S1, unofficial][S2, unofficial].

## Background
Deferred Voxel Shading was presented by Jose Villegas [S1, unofficial]. It expands upon rudimentary voxel cone tracing by merging it with deferred shading [S1, unofficial]. The approach allows for the computation of lighting effects such as emissive materials, indirect shadows, and multiple light bounces on a voxel cone tracing basis [S1, unofficial]. The technique is inspired by voxel cone tracing and deferred rendering [S2, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] Deferred Voxel Shading for Real Time Global Illumination, https://jose-villegas.github.io/post/deferred_voxel_shading/, fetched 2026-10-27 (unofficial)
