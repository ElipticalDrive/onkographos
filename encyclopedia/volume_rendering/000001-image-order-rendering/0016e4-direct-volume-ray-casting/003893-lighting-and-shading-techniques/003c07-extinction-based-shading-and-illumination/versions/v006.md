# Extinction-based Shading and Illumination

Also called: Extinction-based shading, SAT-based illumination

## Overview
Extinction-based Shading and Illumination is a GPU-accelerated technique that computes ambient occlusion, color bleeding, and directional soft shadows in direct volume ray casting by aggregating exponential extinction coefficients using a 3D Summed Area Table (SAT) [S1][S2, unofficial]. Unlike traditional ray-marching shadowing or opacity-based occlusion methods, it approximates spherical and conical neighborhoods with axis-aligned cuboid shells, enabling distance-weighted aggregation in constant time per sample [S2, unofficial]. It sets itself apart from siblings like [Direct Volume Rendering with Phong Shading](3686-direct-volume-rendering-with-phong-shading.md) by providing global illumination effects (occlusion, bleeding) rather than just local surface reflection, and from [Extinction-based Shadow and Scattering Approximation](9149-extinction-based-shadow-and-scattering-approximation.md) by unifying these effects into a single SAT-based framework with specific cuboid-shell approximation strategies [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume dataset $V$, Transfer Function (TF) mapping scalar values to extinction coefficient $\tau$ and base color $C_{base}$, light source positions (static or dynamic), camera parameters [S1][S2, unofficial].
*   **Outputs:** Final image with modulated colors $C_i$ at each sample point [S1].

**Data Structures:**
1.  **Extinction SAT Texture:** A 3D texture storing the prefix sums of extinction coefficients $\tau$ [S2, unofficial].
2.  **Light Cache Texture:** A 3D ping-pong texture that becomes a light cache during rendering, storing precomputed illumination factors (AO, color bleeding, static shadows) [S2, unofficial].

**Numbered Steps:**
1.  **Construct Extinction SAT:**
    *   Compute extinction coefficient $\tau(\mathbf{p})$ for each voxel based on the TF [S2, unofficial].
    *   Build the 3D SAT using a recursive doubling technique with a logarithmic number of passes [S2, unofficial].
    *   Simultaneously build the Color Bleeding SAT (storing vector $\tau_{C_{RGB}}$) in the same texture pass [S2, unofficial].
2.  **Compute Ambient Occlusion (AO) and Color Bleeding:**
    *   Approximate the spherical neighborhood $\Omega_A$ using a series of cuboid shells $Sh_i$ [S2, unofficial].
    *   Query the SAT for each shell to get the aggregate sum of extinction coefficients $SAT(Sh_i)$ [S2, unofficial].
    *   Accumulate AO values iteratively:
        $$ \tau_{Sh_0} = SAT(Sh_0) \cdot r_{Sh_0}^{-2} $$
        $$ \tau_{Sh_{i+1}} = \tau_{Sh_i} + (SAT(Sh_{i+1}) - SAT(Sh_i)) \cdot r_{Sh_{i+1}}^{-2} $$
        where $r_{Sh_i}$ is the radius of shell $i$ [S2, unofficial].
    *   Store the result in the Light Cache Texture [S2, unofficial]. This step is independent of light position and only requires recomputation when the TF changes [S2, unofficial].
3.  **Compute Directional Soft Shadows (if dynamic):**
    *   For each voxel and light source, define a cone $\Psi$ towards the light [S1][S2, unofficial].
    *   Determine the **primary cone axis** as the coordinate axis with the smallest angle to the vector towards the light source [S1][S2, unofficial].
    *   Define **two secondary axes** from the remaining four possibilities to form two axis-aligned planes [S1][S2, unofficial].
    *   Approximate the cone $\Psi$ with a series of cuboids placed along these two planes [S1][S2, unofficial].
    *   Query the Extinction SAT for these cuboids to compute attenuation [S2, unofficial].
    *   Weight the sum using $w_L = V^{-1}$, where $V$ is the aggregate query size [S2, unofficial].
    *   Store in Light Cache if static; otherwise compute per frame [S2, unofficial].
4.  **Rendering (Ray Casting Loop):**
    *   For each sample point $\mathbf{p}_i$ along the ray:
        *   Retrieve base color $C_{base}$ and opacity $\alpha$ from TF [S1].
        *   Retrieve precomputed AO/Color Bleeding from Light Cache [S2, unofficial].
        *   If dynamic shadows, query Extinction SAT for the cone $\Psi$ to get shadow attenuation [S2, unofficial].
        *   Compute lighting term $L(\mathbf{p}_i)$ using Eq. 8 below [S2, unofficial].
        *   Compute final color $C_i = C_{base} \cdot L(\mathbf{p}_i)$ [S1].

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
*   **Number of Shells:** As few as 3 shells are sufficient for AO [S2, unofficial]. Increase if radius of $\Omega_A$ exceeds 10% of dataset radius [S2, unofficial].
*   **Cone Sampling:** User-defined sampling frequency and growth rate on the primary axis [S2, unofficial].
*   **SAT Resolution:** Can be lower than volume resolution [S1][S2, unofficial].

**Pitfalls:**
*   **TF Dependency:** The SAT must be updated whenever the Transfer Function changes because extinction coefficients are TF-dependent [S2, unofficial].
*   **Light Count:** Rendering time increases linearly with the number of light sources, as extinction must be estimated for each [S1].
*   **Precision:** Cuboid approximation of cones may introduce artifacts if the cone is highly oblique relative to axes; handled by choosing the primary axis with the smallest angle [S1].

## Subtopics

### Ambient Occlusion via SAT
Approximates spherical occlusion using cuboid shells and inverse-square weighting [S2, unofficial]. Sets it apart by caching AO independently of light position, incurring zero rendering cost [S2, unofficial]. Choose when global shading without light dependency is needed [S2, unofficial]. [Ambient Occlusion](3536-doppler-volume-rendering.md)

### Color Bleeding via SAT
Stores RGB vectors in the SAT alongside extinction [S2, unofficial]. Sets it apart by computing color bleeding simultaneously with the extinction SAT at no extra computation cost [S2, unofficial]. Choose when inter-reflection color transfer is required [S2, unofficial]. [Color Bleeding via SH Projection](12232-color-bleeding-via-sh-projection.md)

### Directional Soft Shadows via SAT
Uses cuboid approximation of light cones [S1][S2, unofficial]. Sets it apart by caching static shadow factors once and querying dynamic shadows per frame [S2, unofficial]. Choose for soft shadows with static or dynamic lights [S1][S2, unofficial]. [Next Event Estimation for Emissive Volumes](3557-next-event-estimation-for-emissive-volumes.md)

## Operating characteristics
*   **Time Complexity:** SAT construction requires $O(\log N)$ passes per dimension [S2, unofficial]. Rendering cost is zero for cached AO/color bleeding and a single texture lookup for static shadows per sample; dynamic shadows require per-frame SAT queries proportional to the number of cuboid samples [S2, unofficial].
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
