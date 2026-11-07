# Extinction-based Shading and Illumination

Also called: Extinction-based shading, SAT-based illumination

## Overview
This method computes ambient occlusion, soft shadows, and color bleeding in direct volume ray casting by aggregating extinction coefficients using a 3D Summed Area Table (SAT) [S1, unofficial]. Unlike traditional shadow ray sampling, it approximates spherical and conical neighborhoods with cuboid shells, enabling distance-based weighting and interactive transfer function updates [S1, unofficial]. It is distinguished from sibling methods by its use of extinction coefficients (rather than opacity) and cuboid shell approximation (rather than octants or fixed cones) [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scalar volume field $V$, Transfer Function (TF), light positions (static or dynamic), view direction.
*   **Output:** Shaded color $C_{shaded}$ and opacity $\alpha$ for each sample point during ray casting.

**Data Structures:**
1.  **Extinction SAT Texture:** A 3D texture storing the summed area table of extinction coefficients $\tau$. Constructed via recursive doubling [S1, unofficial].
2.  **Light Cache Texture:** A ping-pong 3D texture used during SAT generation, storing accumulated Ambient Occlusion (AO) and shadow attenuation factors. It becomes the light cache during rendering [S1, unofficial].
3.  **Color Bleeding SAT:** Stored in the same texture as the extinction SAT (RGBA format) to allow simultaneous construction [S1, unofficial].

**Parameters:**
*   **SAT Resolution:** Arbitrary within OpenGL limits, does not need to match input volume resolution [S1, unofficial].
*   **Number of Shells ($N_{shells}$):** Typically 3 for AO/color bleeding; more if the query radius exceeds 10% of the dataset radius [S1, unofficial].
*   **Cone Sampling:** User-defined sampling frequency and growth rate on the primary axis [S1, unofficial].
*   **Weighting Functions:**
    *   Ambient Occlusion: $w_A = r^{-2}$, where $r$ is the radius of the aggregate query [S1, unofficial].
    *   Directional Shadows: $w_L = V^{-1}$, where $V$ is the aggregate query size [S1, unofficial].

**Steps:**

1.  **Construct Extinction SAT:**
    Compute the extinction coefficient $\tau$ for each voxel based on the current TF. Construct the 3D SAT using a recursive doubling technique with a logarithmic number of passes [S1, unofficial]. Store in the Extinction SAT Texture. Simultaneously construct the SAT for color bleeding vectors $\tau_{C_{RGB}}$ in the same texture (processing 4 values per operation) [S1, unofficial].

2.  **Compute Ambient Occlusion (AO):**
    Approximate the spherical neighborhood $\Omega_A$ using a series of $N_{shells}$ cuboid shells $Sh_i$ [S1, unofficial].
    *   Query the Extinction SAT for each shell to get the aggregate sum of extinction coefficients.
    *   Accumulate AO values iteratively:
        $$ \tau_{Sh_0} = SAT(Sh_0) \cdot r_{Sh_0}^{-2} $$
        $$ \tau_{Sh_{i+1}} = \tau_{Sh_i} + (SAT(Sh_{i+1}) - SAT(Sh_i)) \cdot r_{Sh_{i+1}}^{-2} $$
    *   Store the final accumulated AO result in the Light Cache Texture [S1, unofficial].

3.  **Compute Directional Soft Shadows:**
    *   **Static Lights:** Compute attenuation factors once by querying the extinction SAT for the sampling cone $\Omega_L = \Psi$. Store in the Light Cache Texture. The cone $\Psi$ is approximated by a series of cuboids. The primary cone axis is the coordinate axis with the smallest angle to the vector to the light source [S1, unofficial].
    *   **Dynamic Lights:** Compute attenuation factors every frame by querying the extinction SAT for the sampling cone $\Psi$ [S1, unofficial].

4.  **Rendering (Ray Casting):**
    For each sample point $\mathbf{p}$ along the ray:
    *   Retrieve base color $C_{base}$ from the TF.
    *   Estimate normal $\mathbf{n}$ from the gradient of $V(\mathbf{p})$ [Parent].
    *   Retrieve pre-computed AO and shadow factors from the Light Cache Texture [S1, unofficial].
    *   Compute reflected light $E_i$ using Eq. 8:
        $$ E_i = I_{A,L} \cdot e^{-w_{A,L} \sum_{\Omega_{A,L}} \Delta^{\tau_j} \cdot k} \cdot C_{RGB} $$
        Where $I_{A,L}$ is ambient or light source intensity, $w_{A,L}$ is the weighting function ($r^{-2}$ or $V^{-1}$), $\sum_{\Omega_{A,L}} \Delta^{\tau_j}$ is the sum of extinction coefficients over the query region (obtained via SAT lookup or cache), $k$ is a local illumination factor (e.g., Phong diffuse/specular), and $C_{RGB}$ is the color [S1, unofficial].
    *   Integrate $C_{shaded} = C_{base} \cdot E_i$ into the front-to-back compositing loop [Parent].

**Pitfalls:**
*   The SAT must be updated whenever the Transfer Function changes because extinction coefficients are TF-dependent [S1, unofficial].
*   If the radius of $\Omega_A$ exceeds 10% of the dataset radius, more shells are needed for accurate AO [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** SAT construction is logarithmic in passes [S1, unofficial]. AO computation is constant time per sample after pre-computation (zero cost during rendering) [S1, unofficial]. Dynamic shadow computation requires per-frame SAT queries [S1, unofficial].
*   **Memory Complexity:** Requires two auxiliary 3D textures (SAT and Light Cache). Color bleeding adds memory bandwidth requirements but not significant performance cost on tested hardware [S1, unofficial].
*   **Performance:**
    *   Head (128x256x256, SAT 192^3, 15 shells): 111fps (AO only), 143fps (dynamic illumination) [S1, unofficial].
    *   Engine (256x256x128, SAT 64^3, 50 cone samples): 57fps (AO only), 130fps (dynamic illumination) [S1, unofficial].
    *   Cornell box (256^3, SAT 256^3, 5 shells, 60 cone samples, static light): 133fps (AO only), 161fps (static illumination) [S1, unofficial].
    *   Pelvis (512x512x461, SAT 64^3, 5 shells, 50 cone samples, dynamic light): 15fps (AO only), 26fps (dynamic illumination) [S1, unofficial].
*   **Strengths:** Interactive TF updates; near-zero cost for static illumination during rendering; order-of-magnitude fewer texture fetches than per-voxel sampling [S1, unofficial].
*   **Limitations:** Approximation errors from cuboid shells; requires GPU memory for auxiliary textures; dynamic shadows require per-frame computation [S1, unofficial].

## Background
Originated in the paper "Fast Shadows and Ambient Occlusion in GPU Volume Ray-Casting" by the University of Zurich (UZH) [S1, unofficial]. It addresses the high cost of traditional shadow ray sampling and neighborhood aggregation in volume rendering. It differs from Diaz et al. [4] by using extinction coefficients and cuboid shells instead of opacity and octants [S1, unofficial].

## Sources
[S1] Fast Shadows and Ambient Occlusion in GPU Volume Ray-Casting, https://www.ifi.uzh.ch/dam/jcr:ffffffff-9e7f-2af5-ffff-ffffcf9259bb/FastShadows.pdf, fetched 2026-10-30 (unofficial)
