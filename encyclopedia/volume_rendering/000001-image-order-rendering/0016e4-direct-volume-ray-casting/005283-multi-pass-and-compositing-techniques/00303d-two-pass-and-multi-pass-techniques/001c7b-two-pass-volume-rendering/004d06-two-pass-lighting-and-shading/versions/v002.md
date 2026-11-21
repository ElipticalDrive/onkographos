# Two-Pass Lighting and Shading

Also called: Illumination Pass, Light/Volume two-pass, Precomputed Lighting Pass

## Overview
Two-Pass Lighting and Shading is a volume rendering technique that decouples the computation of light transport (attenuation and scattering) from the final image compositing [S1]. It solves the performance bottleneck of calculating complex illumination for every pixel in real-time by pre-computing light accumulation in a voxel grid during a dedicated pass, then using this data for a fast, real-time rendering pass [S1].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Volume density field (particle density $\rho(s)$) [S1].
    *   Extinction coefficient $\sigma_t$ (derived from density and particle area $A$) [S1].
    *   Light source position $I$ and intensity $L_{source}$ [S1].
    *   Camera parameters for the render pass [G].
*   **Outputs:**
    *   **Light Buffer:** A 3D voxel grid storing pre-computed incident light values $L_{incident}(v)$ for each voxel [S1].
    *   **Frame Buffer:** The final 2D image with accumulated color and opacity [G].

### Data Structures
1.  **Voxel Grid (Light Buffer):**
    *   Layout: 3D array of floating-point scalars [S1].
    *   Size: Matches the resolution of the volume density field [S1].
    *   Content: $L_{incident}(v)$, the total light intensity collected at voxel $v$ from the light source, including transmission and forward scattering [S1].
2.  **Volume Density Field:**
    *   Layout: 3D array of density values $\rho(s)$ [S1].
    *   Content: Particle density at each spatial location [S1].

### Steps

#### Pass 1: Illumination (CPU)
This pass is executed only when the light source position or intensity changes [S1].

1.  **Initialize Light Buffer:** Create a 3D voxel grid of floating-point scalars, initialized to zero [S1].
2.  **Iterate Voxels:** For each voxel $v$ inside the cloud volume:
    a.  **Trace Light Ray:** Cast a ray $l$ from voxel $v$ towards the light source $I$ [S1].
    b.  **Ray Marching:** Step along the ray from $v$ to the cloud surface, covering distance $D$ [S1].
    c.  **Compute Transmittance:** Calculate $T_{surface \to v}$ using the extinction coefficient $\sigma_t$ along the path [S1].
    d.  **Compute Incident Light:**
        i.  Calculate direct transmission term: $L_{direct} = L_{source} \cdot T_{surface \to v}$ [S1].
        ii. Calculate scattered contribution: Sum the light scattered from each intermediate point $i$ along the path. Use the approximation that 50% of light scatters forward [S1].
        iii. Sum terms: $L_{incident}(v) = L_{direct} + L_{scatter\_term}$ [S1].
    e.  **Store Result:** Write $L_{incident}(v)$ to the corresponding voxel in the Light Buffer [S1].

#### Pass 2: Rendering (GPU)
This pass runs in real-time for every frame [S1].

1.  **Initialize Frame Buffer:** Clear color and opacity buffers [G].
2.  **Iterate Pixels:** For each pixel in the frame buffer:
    a.  **Cast View Ray:** Cast a ray $s$ from the camera through the pixel, traversing the cloud from 0 to $P$ [S1].
    b.  **Ray Marching:** Step along the ray with step size $\Delta s$ [S1].
    c.  **Accumulate Color:** For each step at position $s$:
        i.  **Sample Light:** Retrieve $L_{incident}(s)$ from the Light Buffer (Pass 1 output) [S1].
        ii. **Compute Emission/Reflection:** Determine the color contribution $C(s)$ based on $L_{incident}(s)$ and the volume's reflective/emissive properties [S1].
        iii. **Compute Transmittance:** Calculate $T(s, s+\Delta s)$ using $\sigma_t$ [S1].
        iv. **Composite:** Use front-to-back compositing with an 'under' blend to accumulate color and opacity into the frame buffer [S1].
3.  **Display:** Output the final frame buffer [S1].

### Formulas
*   **Extinction Coefficient:**
    $$ \sigma_t(s) = \rho(s) \cdot A $$
    Where $\rho(s)$ is the particle density at location $s$, and $A$ is the area occluded by a single particle projected on the plane perpendicular to the ray [S1].

*   **Transmittance:**
    $$ T(s, s') = e^{-\int_{s}^{s'} \sigma_t(s'') ds''} $$
    Represents the fraction of light passing from point $s$ to $s'$ without being absorbed or scattered out of the ray [S1].

*   **Incident Light (Continuous):**
    $$ L_{incident}(v) = L_{source} \cdot T_{surface \to v} + \int_{surface}^{v} L_{scatter}(s) \cdot T_{s \to v} ds $$
    The light collected at point $v$ is the sum of:
    1.  Light propagated directly from the source to $v$, attenuated by total transmittance from the cloud surface to $v$ [S1].
    2.  Light scattered in the forward direction from intermediate points $s$ along the ray, attenuated by transmittance from $s$ to $v$ [S1].

*   **Forward Scattering Approximation:**
    It is assumed that 50% of the scattered light is directed forward within a small solid angle (~0.0001 steradians). The remaining 50% is ignored for computational efficiency [S1].

*   **Incident Light (Discrete Ray Marching):**
    $$ L_{incident}(v) = L_{source} \cdot T_{surface \to v} + \sum_{i} L_{scatter}(i) \cdot T_{i \to v} \cdot \Delta s $$
    Where the sum iterates over discrete steps $i$ from the cloud surface to $v$, with step size $\Delta s$ [S1].

### Parameters
*   **$\sigma_t$:** Extinction coefficient. Typical values depend on the medium (e.g., water vapor density) [S1].
*   **Scattering Fraction:** 0.5 (50% of light scattered forward) [S1].
*   **Solid Angle:** ~0.0001 steradians for forward scattering approximation [S1].

### Edge Cases and Pitfalls
*   **Light Source Changes:** The Light Buffer becomes invalid if the light source moves. The system must detect this and re-run Pass 1 [S1].
*   **Precision:** Floating-point precision in the Light Buffer may lead to banding or noise if light values vary significantly [S1].
*   **Boundary Conditions:** The ray marching in Pass 1 must correctly identify the cloud surface to start the transmittance calculation from the correct point [S1].
*   **Scattering Approximation:** Ignoring 50% of scattered light reduces accuracy, potentially leading to darker or less realistic volumes compared to full scattering models [S1].

## Operating characteristics

*   **Time Complexity:**
    *   Pass 1: $O(V \cdot N_l)$, where $V$ is the number of voxels and $N_l$ is the number of steps in the light ray. Executed only on light change [S1].
    *   Pass 2: $O(P \cdot N_r)$, where $P$ is the number of pixels and $N_r$ is the number of steps in the view ray. Executed every frame [S1].
*   **Memory Complexity:** $O(V)$ for the Light Buffer, plus $O(V)$ for the volume density field [S1].
*   **Performance:** Significantly faster than real-time full light integration per pixel. The bottleneck is the initial light computation if the light moves frequently [S1].
*   **Strengths:** Real-time rendering capability for complex lighting; decouples lighting from view changes [S1].
*   **Limitations:** Static lighting assumption (or expensive re-computation); approximation errors in scattering; requires extra memory for the light buffer [S1].
*   **When to Use:** Real-time applications with static or slowly moving light sources (e.g., clouds, smoke) [S1].
*   **When Not to Use:** Dynamic lighting with frequent changes; high-fidelity offline rendering where full scattering is required [S1].

## Background
The method is based on the principles of transmittance and scattering described by Max [12], with optimizations from Harris [13] and Tessendorf [33] [S1]. It was implemented in GPU code for real-time volumetric cloud rendering, using a pseudo-spheroid-based approach instead of a particle system [S1]. The specific implementation described in [S1] uses a CPU for the illumination pass and a GPU for the rendering pass, leveraging a pre-calculated voxel grid [S1].

## Sources
[S1] Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry, https://doi.org/10.3390/sym10040125, fetched 2026-10-26
