# Specific Material and Geometry Rendering

Also called: Material Parameter Setting, Surface and DRR Rendering Modes

## Overview
Specific Material and Geometry Rendering provides mechanisms to define volumetric material properties (ambient, diffuse, specular) and rendering modes (Surface, DRR, MIP) within volume rendering pipelines, particularly in the VolPack API and real-time ray-marched implementations. It solves the problem of mapping scalar or vector volume data to visual appearance and handling geometric occlusion or specific projection requirements. It differs from general ray marching by explicitly exposing material parameter interfaces and distinct rendering modes that alter the fragment shader's output logic, rather than relying solely on the integration of the radiative transfer equation.

## Shared principles
The family shares the concept of decoupling the physical density/scattering data from the visual appearance rules. This is achieved through:
1.  **Material Parameterization:** Defining optical properties (reflection coefficients, shininess) that are applied to voxels based on their geometric orientation (gradient direction) relative to the viewer.
2.  **Rendering Mode Selection:** Using a uniform variable to switch the fragment shader's computation between different output definitions (e.g., Phong shading vs. attenuation-only vs. maximum intensity).
3.  **Gradient-Based Classification:** Determining surface properties (exterior vs. interior) based on the sign of the volume gradient vector at the sample point.

## Subtopics

### [Procedural Volumetric Cloud Rendering](3480-procedural-volumetric-cloud-rendering.md)
Procedural volumetric cloud rendering is a real-time technique that ray-marches a spherical cloud layer built from tileable 3D noise fields, lighting it with multiple-scattering approximations and Mie phase functions. It solves the problem of rendering physically plausible, large-scale cloudscapes with dynamic lighting and shadows at interactive frame rates. What sets it apart from [Mr-Voxel Volume Rendering](3874-mr-voxel-volume-rendering.md) is the use of procedural noise for density generation rather than pre-computed voxel grids, and from [Ray-marched Volumetric Fog](4248-ray-marched-volumetric-fog.md) by employing specific cloud physics (Mie scattering, multiple scattering approximations, and vertical density profiles) rather than generic volumetric fog.

### [Mr-Voxel Volume Rendering](3874-mr-voxel-volume-rendering.md)
Mr-Voxel Volume Rendering is a direct volume ray casting technique implemented in the `VolumeIntegrator::Li` function that performs ray marching through a single volume region, accumulating emitted light and computing transmittance and scattering contributions from a single light source. It is distinguished from its siblings by its specific implementation within the Mr-Voxel library's integrator, which combines an accelerated voxel data structure with a ray marching loop that handles single-volume, single-light scenarios with Russian Roulette termination. Unlike [Procedural Volumetric Cloud Rendering](3480-procedural-volumetric-cloud-rendering.md) which often relies on procedural noise for density, or [Ray-marched Volumetric Fog](4248-ray-marched-volumetric-fog.md) which may use simplified scattering models, this method explicitly computes optical depth and direct lighting transmittance per step for a defined volume object.

### [Ray-marched Volumetric Fog](4248-ray-marched-volumetric-fog.md)
Ray-marched volumetric fog simulates light absorption, scattering, soft shadows, and light shafts (god rays) by treating fog as a 3D volume of participating media traced via ray-marching from the camera to the scene. It differs from backward ray-marching (scene to camera) by enabling early-out optimizations when transmittance becomes negligible, and it differs from voxel-based methods by operating directly in screen space using depth reconstruction rather than explicit volume data structures.

### Material Parameter Setting
Material parameter setting allows the configuration of volumetric material properties such as ambient, diffuse, specular reflection, and shininess. The function `vpSetMaterial` is used to assign these properties to specific material identifiers (`VP_MATERIAL0` through `VP_MATERIAL5`). Properties include `VP_AMBIENT`, `VP_DIFFUSE`, `VP_SPECULAR` (taking RGB values), and `VP_SHINYNESS` (taking a specular exponent, ignoring G and B). The application of these properties is controlled by `surface_side`, which can be `VP_EXTERIOR` (voxels where the gradient points towards the viewer), `VP_INTERIOR` (gradient points away), or `VP_BOTH_SIDES`. The meaning of exterior and interior can be reversed using `VP_REVERSE_SURFACE_SIDES` [S1].

### Surface and DRR Rendering Modes
Rendering modes define how the final color $C(s)$ is computed for a sample point $s$ along the ray [S2, unofficial].
*   **Surface Rendering:** $C(s)$ is computed according to the Phong equation using the volume gradient [S2, unofficial].
*   **Digitally Reconstructed Radiographs (DRRs):** $C(s) = (0,0,0)$, and only the backlight is attenuated by the accumulated transmittance $t$ [S2, unofficial].
*   **Maximum Intensity Projections (MIPs):** The output is the opacity $\alpha$ of the maximum-opacity interval along the ray [S2, unofficial].
A uniform variable tells the fragment shader which mode to use [S2, unofficial].

## Background
The VolPack API, documented in the VolPack User's Guide, introduced specific material parameter setting and rendering modes (Surface, DRR, MIP) for volume rendering [S1]. These features allow for flexible visualization of volumetric data, ranging from surface-like shading to projection-based analyses like DRRs and MIPs, commonly used in medical imaging and scientific visualization. The concepts of material parameterization and mode switching have been adapted in various real-time implementations, such as procedural cloud rendering and volumetric fog shaders, to balance physical accuracy with performance.

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S2] CISC/CMPE 454, GPU Volume Rendering, https://watkins.cs.queensu.ca/~jstewart/454/notes/44/44-gpu-volume-rendering.html, fetched 2026-10-30 (unofficial)
