# Extinction-based Shading and Illumination

Also called: Extinction-based shading, SAT-based illumination

## Overview
Extinction-based Shading and Illumination is a GPU-accelerated technique that computes ambient occlusion, color bleeding, and directional soft shadows in direct volume ray casting by aggregating exponential extinction coefficients using a 3D Summed Area Table (SAT) [S1][S2, unofficial]. Unlike traditional ray-marching shadowing or opacity-based occlusion methods, it approximates spherical and conical neighborhoods with axis-aligned cuboid shells, enabling distance-weighted aggregation in constant time per sample [S2, unofficial]. It sets itself apart from siblings like [Direct Volume Rendering with Phong Shading](3686-direct-volume-rendering-with-phong-shading.md) by providing global illumination effects (occlusion, bleeding) rather than just local surface reflection, and from [Extinction-based Shadow and Scattering Approximation](9149-extinction-based-shadow-and-scattering-approximation.md) by unifying these effects into a single SAT-based framework with specific cuboid-shell approximation strategies [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume dataset $V$, Transfer Function (TF) mapping scalar values to extinction coefficient $\tau$ and base color $C_{base}$, light source positions (static or dynamic), camera parameters.
*   **Outputs:** Final image with modulated colors $C_i$ at each sample point.

**Data Structures:**
1.  **Extinction SAT Texture:** A 3D texture storing the prefix sums of extinction coefficients $\tau$. Resolution is arbitrary (typically lower than volume) [S2, unofficial].
2.  **Light Cache Texture:** A 3D texture (ping-pong buffer during construction) storing precomputed illumination factors (AO, color bleeding, static shadows) [S2, unofficial].
3.  **Color Bleeding SAT:** Stored within the same texture as the extinction SAT, processing 4 values (RGB + $\tau$) per operation [S2, unofficial].

**Numbered Steps:**

1.  **Construct Extinction SAT:**
    *   Compute extinction coefficient $\tau(\mathbf{p}) = 1 - e^{-\alpha(\mathbf{p}) \cdot \Delta d}$ for each voxel, where $\alpha$ is from the TF and $\Delta d$ is step size [S1][S2, unofficial].
    *   Build the 3D SAT using recursive doubling in $O(\log N)$ passes [S2, unofficial].
    *   Simultaneously build the Color Bleeding SAT (storing vector $\tau_{C_{RGB}}$) in the same texture pass [S2, unofficial].

2.  **Compute Ambient Occlusion (AO) and Color Bleeding:**
    *   Approximate the spherical neighborhood $\Omega_A$ using a series of $N$ concentric cuboid shells $Sh_i$ centered at the voxel [S2, unofficial].
    *   Query the SAT for each shell to get the aggregate sum of extinction coefficients $SAT(Sh_i)$ [S2, unofficial].
    *   Accumulate AO values iteratively:
        $$ \tau_{Sh_0} = SAT(Sh_0) \cdot r_{Sh_0}^{-2} $$
        $$ \tau_{Sh_{i+1}} = \tau_{Sh_i} + (SAT(Sh_{i+1}) - SAT(Sh_i)) \cdot r_{Sh_{i+1}}^{-2} $$
        where $r_{Sh_i}$ is the radius of shell $i$ [S2, unofficial].
    *   Store the result in the Light Cache Texture. This step is independent of light position and only needs recomputation when the TF changes [S2, unofficial].

3.  **Compute Directional Soft Shadows (if dynamic):**
    *   For each voxel and light source, define a cone $\Psi$ towards the light [S1][S2, unofficial].
    *   Determine the **primary cone axis** as the coordinate axis with the smallest angle to the vector towards the light source [S1][S2, unofficial].
    *   Define **two secondary axes** from the remaining four possibilities to form two axis-aligned planes [S1][S2, unofficial].
    *   Approximate the cone $\Psi$ with a series of cuboids placed along these two planes [S1][S2, unofficial].
    *   Query the Extinction SAT for these cuboids to compute attenuation.
    *   Weight the sum using $w_L = V^{-1}$, where $V$ is the aggregate query size [S2, unofficial].
    *   Store in Light Cache if static; otherwise compute per frame [S2, unofficial].

4.  **Rendering (Ray Casting Loop):**
    *   For each sample point $\mathbf{p}_i$ along the ray:
        *   Retrieve base color $C_{base}$ and opacity $\alpha$ from TF [S1, S8307].
        *   Retrieve precomputed AO/Color Bleeding from Light Cache [S2, unofficial].
        *   If dynamic shadows, query Extinction SAT for the cone $\Psi$ to get shadow attenuation [S2, unofficial].
        *   Compute lighting term $L(\mathbf{p}_i)$ using Eq. 8 below.
        *   Compute final color $C_i = C_{base} \cdot L(\mathbf{p}_i)$ [S1, S8307].

**Formula:**
The reflected light $E_i$ for a voxel due to a light source (ambient or directional) is:
$$ E_i = I_{A,L} \cdot e^{-w_{A,L} \sum_{\Omega_{A,L}} \Delta^{\tau_j} \cdot k} \cdot C_{RGB} $$
Where:
*   $I_{A,L}$: Ambient or light source intensity [S2, unofficial].
*   $w_{A,L}$: Weighting function. $w_A = r^{-2}$ for AO, $w_L = V^{-1}$ for shadows [S2, unofficial].
*   $\sum_{\Omega_{A,L}} \Delta^{\tau_j}$: Sum of extinction coefficients over the query region (obtained via SAT) [S2, unofficial].
*   $k$: Local illumination factor (e.g., gradient-based normal) [S2, unofficial].
*   $C_{RGB}$: Color vector (for color bleeding) or base color [S2, unofficial].

**Parameters:**
*   **Number of Shells:** As few as 3 shells are sufficient for AO [S2, unofficial]. Increase if radius of $\Omega_A$ > 10% of dataset radius [S2, unofficial].
*   **Cone Sampling:** User-defined sampling frequency and growth rate on the primary axis [S2, unofficial].
*   **SAT Resolution:** Can be lower than volume resolution [S1][S2, unofficial].

**Pitfalls:**
*   **TF Dependency:** The SAT must be updated whenever the Transfer Function changes because extinction coefficients are TF-dependent [S2, unofficial].
*   **Light Count:** Rendering time increases linearly with the number of light sources, as extinction must be estimated for each [S1].
*   **Precision:** Cuboid approximation of cones may introduce artifacts if the cone is highly oblique relative to axes; handled by choosing the primary axis with the smallest angle [S1].

## Subtopics

### Ambient Occlusion via SAT
Uses cuboid shells and inverse-square weighting to approximate spherical occlusion. Chosen when global shading is needed without light position dependency. [S2, unofficial]

### Color Bleeding via SAT
Stores RGB vectors in the SAT alongside extinction. Chosen when inter-reflection color transfer is required. [S2, unofficial]

### Directional Soft Shadows via SAT
Uses cuboid approximation of light cones. Chosen for soft shadows; static lights are cached, dynamic lights are queried per frame. [S1][S2, unofficial]

## Operating characteristics
*   **Time Complexity:** SAT construction is $O(\log N)$ passes per dimension [S2, unofficial]. Rendering is $O(1)$ per sample for static effects (cache lookup) and $O(K)$ for dynamic shadows (K cuboid queries) [S2, unofficial].
*   **Memory Complexity:** Requires two auxiliary 3D textures (SAT and Light Cache) [S2, unofficial].
*   **Performance:**
    *   Head (128x256x256, SAT 192^3, 15 shells): 111fps (AO only), 143fps (dynamic) [S2, unofficial].
    *   Engine (256x256x128, SAT 64^3, 50 cone samples): 57fps (AO only), 130fps (dynamic) [S2, unofficial].
    *   Cornell Box (256^3, SAT 256^3, 5 shells, 60 cone samples, static): 133fps (AO only), 161fps (static) [S2, unofficial].
    *   Pelvis (512x512x461, SAT 64^3, 5 shells, 50 cone samples, dynamic): 15fps (AO only), 26fps (dynamic) [S2, unofficial].
*   **Strengths:** Interactive frame rates for global illumination effects; decouples lighting from view changes (AO); efficient constant-time neighborhood queries [S1][S2, unofficial].
*   **Limitations:** Memory intensive for high-resolution SATs; requires SAT rebuild on TF change; performance degrades with many light sources [S1][S2, unofficial].

## Background
Developed by Schlegel et al. in 2011 [SMP11], this technique builds on earlier SAT-based ambient occlusion work by Desgranges and Engel [DE07] [S1]. It was introduced to provide interactive global illumination in volume ray casting by leveraging the order-independent summation property of extinction coefficients [S1]. It is used in scientific visualization and medical imaging where realistic lighting enhances depth perception [S1].

## Sources
[S1] Interactive Volume Rendering with Volumetric Illumination, https://doi.org/10.2312/conf/eg2012/stars/053-074, fetched 2026-10-26
[S2] PDF Extinction-based Shading and Illumination in GPU Volume Ray-Casting - UZH, https://www.ifi.uzh.ch/dam/jcr:ffffffff-9e7f-2af5-ffff-ffffcf9259bb/FastShadows.pdf, fetched 2026-10-30 (unofficial)
