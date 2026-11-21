# Two-Pass Lighting and Shading

Also called: Illumination Pass, Light/Volume two-pass, Precomputed Lighting Pass

## Overview
Two-Pass Lighting and Shading is a volume rendering technique that decouples the computation of light transport (attenuation and scattering) from the final image compositing. It solves the performance bottleneck of calculating complex illumination for every pixel in real-time by pre-computing light accumulation in a voxel grid during a dedicated pass, then using this data for a fast, real-time rendering pass. It differs from standard direct volume ray casting by moving the expensive light integration step off the critical rendering path, and from single-pass methods by requiring a two-stage data dependency between the light buffer and the frame buffer.

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Volume density field (particle density $\rho(s)$).
    *   Extinction coefficient $\sigma_t$ (derived from density and particle area $A$).
    *   Light source position $I$ and intensity $L_{source}$.
    *   Camera parameters for the render pass.
*   **Outputs:**
    *   **Light Buffer:** A 3D voxel grid storing pre-computed incident light values $L_{incident}(v)$ for each voxel.
    *   **Frame Buffer:** The final 2D image with accumulated color and opacity.

### Data Structures
1.  **Voxel Grid (Light Buffer):**
    *   Layout: 3D array of floating-point scalars.
    *   Size: Matches the resolution of the volume density field.
    *   Content: $L_{incident}(v)$, the total light intensity collected at voxel $v$ from the light source, including transmission and forward scattering.
2.  **Volume Density Field:**
    *   Layout: 3D array of density values $\rho(s)$.
    *   Content: Particle density at each spatial location.

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
    1.  Light propagated directly from the source to $v$, attenuated by total transmittance from the cloud surface to $v$.
    2.  Light scattered in the forward direction from intermediate points $s$ along the ray, attenuated by transmittance from $s$ to $v$ [S1].

*   **Forward Scattering Approximation:**
    It is assumed that 50% of the scattered light is directed forward within a small solid angle (~0.0001 steradians). The remaining 50% is ignored for computational efficiency [S1].

*   **Incident Light (Discrete Ray Marching):**
    $$ L_{incident}(v) = L_{source} \cdot T_{surface \to v} + \sum_{i} L_{scatter}(i) \cdot T_{i \to v} \cdot \Delta s $$
    Where the sum iterates over discrete steps $i$ from the cloud surface to $v$, with step size $\Delta s$ [S1].

### Parameters
*   **$\sigma_t$:** Extinction coefficient. Typical values depend on the medium (e.g., water vapor density).
*   **Scattering Fraction:** 0.5 (50% of light scattered forward) [S1].
*   **Solid Angle:** ~0.0001 steradians for forward scattering approximation [S1].

### Steps

#### Pass 1: Illumination (CPU)
This pass is executed only when the light source position or intensity changes.

1.  **Initialize Light Buffer:** Create a 3D voxel grid of floating-point scalars, initialized to zero.
2.  **Iterate Voxels:** For each voxel $v$ inside the cloud volume:
    a.  **Trace Light Ray:** Cast a ray $l$ from voxel $v$ towards the light source $I$.
    b.  **Ray Marching:** Step along the ray from $v$ to the cloud surface (or light source), covering distance $D$.
    c.  **Compute Transmittance:** Calculate $T_{surface \to v}$ using the extinction coefficient $\sigma_t$ along the path [S1].
    d.  **Compute Incident Light:**
        i.  Calculate direct transmission term: $L_{direct} = L_{source} \cdot T_{surface \to v}$.
        ii. Calculate scattered contribution: Sum the light scattered from each intermediate point $i$ along the path. Use the approximation that 50% of light scatters forward [S1].
        $$ L_{scatter\_term} = \sum_{i} (0.5 \cdot \sigma_s(i) \cdot L_{local}(i)) \cdot T_{i \to v} \cdot \Delta s $$
        *(Note: $\sigma_s$ is scattering coefficient, often approximated or derived from $\sigma_t$ if not explicitly given. The source implies $L_{scatter}$ is computed based on local illumination and scattering density [S1]).*
        iii. Sum terms: $L_{incident}(v) = L_{direct} + L_{scatter\_term}$.
    e.  **Store Result:** Write $L_{incident}(v)$ to the corresponding voxel in the Light Buffer.

#### Pass 2: Rendering (GPU)
This pass runs in real-time for every frame.

1.  **Initialize Frame Buffer:** Clear color and opacity buffers.
2.  **Iterate Pixels:** For each pixel in the frame buffer:
    a.  **Cast View Ray:** Cast a ray $s$ from the camera through the pixel, traversing the cloud from near plane to far plane (0 to $P$).
    b.  **Ray Marching:** Step along the ray with step size $\Delta s$.
    c.  **Accumulate Color:** For each step at position $s$:
        i.  **Sample Light:** Retrieve $L_{incident}(s)$ from the Light Buffer (Pass 1 output).
        ii. **Compute Emission/Reflection:** Determine the color contribution $C(s)$ based on $L_{incident}(s)$ and the volume's reflective/emissive properties. The source mentions "reflection contributions" but does not provide a specific BRDF or phase function, implying a simplified model where incident light drives the color [S1].
        iii. **Compute Transmittance:** Calculate $T(s, s+\Delta s)$ using $\sigma_t$ [S1].
        iv. **Composite:** Use front-to-back compositing with an 'under' blend (or standard alpha compositing) to accumulate color and opacity into the frame buffer.
        $$ C_{out} = C_{in} + C(s) \cdot T(s, s+\Delta s) $$
        $$ \alpha_{out} = \alpha_{in} + \alpha(s) \cdot T(s, s+\Delta s) \cdot (1 - \alpha_{in}) $$
        *(Note: Standard alpha compositing is implied by "under blend" and "accumulates color and opacity" in the parent article [Parent].)*
3.  **Display:** Output the final frame buffer.

### Edge Cases and Pitfalls
*   **Light Source Changes:** The Light Buffer becomes invalid if the light source moves. The system must detect this and re-run Pass 1. This can cause stuttering if not handled asynchronously [S1].
*   **Precision:** Floating-point precision in the Light Buffer may lead to banding or noise if light values vary significantly.
*   **Boundary Conditions:** The ray marching in Pass 1 must correctly identify the cloud surface to start the transmittance calculation from the correct point.
*   **Scattering Approximation:** Ignoring 50% of scattered light reduces accuracy, potentially leading to darker or less realistic volumes compared to full scattering models [S1].

## Operating characteristics

*   **Time Complexity:**
    *   Pass 1: $O(V \cdot N_l)$, where $V$ is the number of voxels and $N_l$ is the number of steps in the light ray. Executed only on light change.
    *   Pass 2: $O(P \cdot N_r)$, where $P$ is the number of pixels and $N_r$ is the number of steps in the view ray. Executed every frame.
*   **Memory Complexity:** $O(V)$ for the Light Buffer, plus $O(V)$ for the volume density field.
*   **Performance:** Significantly faster than real-time full light integration per pixel. The bottleneck is the initial light computation if the light moves frequently.
*   **Strengths:** Real-time rendering capability for complex lighting; decouples lighting from view changes.
*   **Limitations:** Static lighting assumption (or expensive re-computation); approximation errors in scattering; requires extra memory for the light buffer.
*   **When to Use:** Real-time applications with static or slowly moving light sources (e.g., clouds, smoke).
*   **When Not to Use:** Dynamic lighting with frequent changes; high-fidelity offline rendering where full scattering is required.

## Background
The method is based on the principles of transmittance and scattering described by Max [12], with optimizations from Harris [13] and Tessendorf [33]. It was implemented in GPU code for real-time volumetric cloud rendering, using a pseudo-spheroid-based approach instead of a particle system [S1]. The specific implementation described in [S1] uses a CPU for the illumination pass and a GPU for the rendering pass, leveraging a pre-calculated voxel grid.

## Sources
[S1] Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry, https://doi.org/10.3390/sym10040125, fetched 2026-10-26
