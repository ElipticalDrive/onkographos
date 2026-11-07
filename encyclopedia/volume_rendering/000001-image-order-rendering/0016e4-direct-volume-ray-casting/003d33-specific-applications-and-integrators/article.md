# Specific Applications and Integrators

Also called: Application-specific DVR, Domain-specific Integrators

## Overview
This family encompasses direct volume ray casting methods adapted for specialized domains—such as medical imaging, manufacturing simulation, and interactive visualization—where standard optical compositing is insufficient or where the rendering pipeline requires domain-specific physical models. Unlike generic ray casting, these methods modify the integrand, sampling strategy, or compositing logic to solve problems like polymerization dose calculation, photorealistic light transport, or spatially varying parameter control.

## Shared principles
All methods in this family adhere to the fundamental direct volume ray casting loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md). However, they diverge in the evaluation of the integrand or the structure of the compositing pass. The core distinction lies in the replacement or augmentation of the standard optical properties $(C_i, \alpha_i)$ with domain-specific quantities (e.g., polymerization dose, photon radiance, or customized intensity) before or during the accumulation step.

## Subtopics

### [Customization Volume Integration](5291-customization-volume-integration.md)
A method for applying position-dependent modifications to visualization parameters (e.g., gamma correction, opacity scaling) via a secondary "customization volume." It differs from standard DVR by modulating input data intensity before the transfer function is applied, allowing spatial control without altering the underlying scalar data. It is distinct from Cinematic Rendering, which focuses on physical light transport, and TVAM Integrators, which focus on manufacturing physics.

### [TVAM Integrator](6814-tvam-integrator.md)
A volumetric integrator designed for Tomographic Volumetric Additive Manufacturing (TVAM). It simulates light transport from projector emitters through photosensitive resin to calculate polymerization doses. Unlike camera-based ray casting, it samples rays from projectors and incorporates manufacturing physics (polymerization thresholds, inhibitor dynamics) into the integrand. It serves as a base class for specialized variants like the CornerIntegrator.

### [Cinematic Rendering](9473-cinematic-rendering.md)
A physiologically-based volume rendering method that simulates visible photon interactions with medical data (CT/MRI) to produce photorealistic images. Unlike standard DVR, which uses single straight rays and local gradient shading, Cinematic Rendering uses path tracing to simulate multiple scattering, absorption, and global illumination (soft shadows, subsurface scattering). This accounts for lighting effects from neighboring voxels, enhancing depth perception.

### Cutting Plane Integration
An interactive clipping technique that renders a semi-transparent plane mesh and normal arrow to dynamically slice the volume and reveal internal structures [S1, unofficial].
*   **Step:** Map normalized height to world Z position: `planeMesh.position.z = 200 - 400 * planeHeight`. Convert degrees to radians and apply rotation, including a $\pi$ offset if `renderAbove` is true [S1, unofficial].
*   **Parameters:** `planeHeight` (normalized float 0–1), `planeRotX`/`planeRotY` (rotation angles in degrees), `renderAbove` (boolean flag flipping the plane 180° around Y) [S1, unofficial].

### Two Level Volume Rendering with Layers
A technique that achieves results similar to traditional two-level volume rendering by rendering different styles (e.g., MIP vs. DVR) into separate layers and compositing them, rather than selecting rendering types per ray segment during integration [S2].
*   **Step:** Render each distinct material style or transfer function into a separate layer (e.g., DVR for soft tissue, DVR for bones). Composite the separate layers into a final image using a compositing function that maps the resulting layers to the view volume [S2].
*   **Parameters:** Opacity values can be assigned to layers during rendering or compositing (e.g., layer 1 opacity 0.4, layer 3 opacity 0.1, layer 2 opacity 0.5) [S2].
*   **Comparison:** Unlike original two-level volume rendering which composites segments in the order the ray encounters materials (e.g., $I = A_1 B_1 A_2 B_2$), this method composites all segments of the same material separately first, then composites the intermediate results (e.g., $I = A_1 A_2 A_3 B_1 B_2$). Because compositing is not commutative, results generally differ [S2].

### Volumetric Ray Tracing Framework (Sobierajski/Kaufman)
A complete framework for volumetric ray tracing that includes global illumination effects (shadows, reflections, light interaction between volumetric/geometric objects) and volumetric rendering effects (fog, transparency) [S3, unofficial]. It was incorporated into VolVis, a comprehensive volume visualization system developed at Stony Brook University [S3, unofficial].

## Background
The need for specific integrators arose as volume rendering moved from simple visualization to domain-specific applications. Early work by Sobierajski and Kaufman established a framework for volumetric ray tracing with global illumination, integrated into the VolVis system at Stony Brook University [S3, unofficial]. Later, specialized integrators emerged for additive manufacturing, such as the TVAM Integrator, which models the physics of resin polymerization [S1, unofficial]. In medical imaging, Cinematic Rendering was developed to provide photorealistic visualization by simulating complex light transport, moving beyond the local shading approximations of traditional DVR [S2][S3, unofficial].

## Sources
[S1] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
[S2] Layers for Effective Volume Rendering, https://doi.org/10.2312/vg/vg-pbg08/081-088, fetched 2026-10-26
[S3] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
