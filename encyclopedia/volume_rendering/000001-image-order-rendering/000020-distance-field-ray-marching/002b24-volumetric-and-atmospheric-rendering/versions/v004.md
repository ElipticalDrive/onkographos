# Volumetric and Atmospheric Rendering

Also called: Volume Ray Marching, Atmospheric Ray Marching, Single Scattering Approximation

## Overview
Volumetric and Atmospheric Rendering is a family of image-order ray marching techniques that simulate light transport through participating media, such as clouds, smoke, fog, and planetary atmospheres, by integrating density and scattering properties along view rays [S1, unofficial][S2]. It differs from [SDF Cloud Rendering](2064-sdf-cloud-rendering.md) by treating the volume as a sampled or procedural density field rather than an implicit surface, and from [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md) by focusing on localized, animated volumetric structures rather than global spherical atmospheric scattering [S1, unofficial][S2].

## Shared principles
The family relies on the discretization of the Radiative Transfer Equation (RTE) along a ray $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the direction vector. The core mechanism involves marching through a 3D density field $\rho(\mathbf{p})$ and accumulating color and opacity using the Beer-Lambert law for extinction and a phase function for scattering [S1, unofficial][S2].

1.  **Ray Setup**: For each pixel, a view ray is constructed from the camera position and direction [G].
2.  **Marching Loop**: The ray is stepped through the volume with a step size $\Delta s$. At each sample point $\mathbf{p}$, the density $\rho(\mathbf{p})$ is evaluated [S1, unofficial][S2].
3.  **Extinction**: The transmittance $T$ is updated using the extinction coefficient $\sigma_T = \sigma_S + \sigma_A$ (scattering + absorption):
    $$ T \leftarrow T \cdot e^{-\sigma_T \rho(\mathbf{p}) \Delta s} $$
    [S1, unofficial].
4.  **Scattering**: If single scattering is approximated, the inscattered light $L_{in}$ from a light source is computed by marching towards the light to determine optical depth, then attenuated by the phase function $P(\theta)$ and scattering coefficient $\sigma_S$ [S1, unofficial]:
    $$ L_{out} \leftarrow L_{out} + T \cdot \sigma_S \cdot \rho(\mathbf{p}) \cdot P(\theta) \cdot L_{in} \cdot \Delta s $$
    [S1, unofficial].
5.  **Accumulation**: Color is accumulated using alpha blending:
    $$ C_{accum} \leftarrow C_{accum} + C_{sample} \cdot \alpha \cdot (1 - C_{accum}.a) $$
    $$ \alpha \leftarrow \alpha + \alpha_{sample} \cdot (1 - C_{accum}.a) $$
    [S1, unofficial].

## Subtopics

### [Ray Marching with Single Scattering](2057-ray-marching-with-single-scattering.md)
A real-time technique for rendering translucent, animated volumetric clouds using procedural noise or 3D textures for density and physically-based single scattering illumination. It targets localized cloud volumes with explicit bounding or spherical boundaries and wind animation.

### [SDF Cloud Rendering](2064-sdf-cloud-rendering.md)
A technique for rendering volumetric clouds using Signed Distance Fields (SDFs) via ray marching, utilizing warped sphere primitives and smooth unions to create complex shapes. It represents the cloud volume as an implicit surface defined by analytic distance functions rather than a sampled density grid.

### [Volume Ray Marching with Temporal Accumulation](4804-volume-ray-marching-with-temporal-accumulation.md)
A real-time rendering technique that discretizes view rays through a 3D density field to compute color and opacity, enhanced by blending the current frame's result with a history texture to reduce noise and stabilize the image over time. It distinguishes itself by incorporating temporal feedback, allowing for higher visual fidelity with fewer spatial samples.

### [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md)
A real-time technique for rendering planetary atmospheres using single scattering, exponential density falloff, and the Rayleigh phase function. It solves the problem of visualizing the sky and horizon glow around a spherical planet by integrating the Radiative Transfer Equation along view rays.

### Wax Volume Raymarching
*   **Step**: Raymarch the interior volume of the glass container. Compute surface normal using the gradient of the density field. Set fragment depth explicitly using `gl_FragDepth` in the opaque pass. Shading model includes: wrap lighting, soft Fresnel, fake subsurface scattering based on traversed thickness, and emission driven by temperature and proximity to the light bulb [S1, unofficial].
*   **Number**: Artifacts: 0.00% of surface voxels cause a round-trip to the threshold, even at 60 Hz with ±30% jitter [S1, unofficial].

### X-Ray Visualization
*   **Definition**: A visualization method that computes average density values along a ray cast through the data, visualizing the result as a grey scale image similar to a medical X-ray [S2].
*   **Step**: Cast a ray through the data set in the desired viewing direction. Calculate the average density values along the ray. Visualize the average density values as a grey scale image [S2].
*   **Parameter**: View rotation: The view of the data set can be rotated in any of the 3 directions (X, Y, Z axes) in degrees (-360 ... 0 ... 360) [S2].

## Background
The method originates from early volume rendering applications in medical visualization, where ray casting through 3D datasets was used to generate X-ray-like visualizations and shaded 3D views [S2]. The specific implementation of wax volume raymarching with liquid simulation was developed by Nathaniel D. Dr. Lidy in the context of a lava lamp simulation, utilizing GLSL shaders for real-time rendering on the GPU [S1, unofficial].

## Sources
[S1] Étape 4 : surface de cire raymarchée et liquide volumique, https://github.com/nathanddrl/lava-lamp/pull/4, fetched 2026-10-27 (unofficial)
[S2] VOLUME RENDERING Application by Thomas Lidy, https://www.cg.tuwien.ac.at/courses/Visualisierung/2001-2002/Ergebnisse/Beispiel1/LidyT/description.html, fetched 2026-10-29
