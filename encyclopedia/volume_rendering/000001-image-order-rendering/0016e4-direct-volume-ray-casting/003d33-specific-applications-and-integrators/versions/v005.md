# Specific Applications and Integrators

Also called: specialized volume rendering integrators, application-specific DVR extensions

## Overview
This family encompasses methods that extend the standard direct volume ray casting pipeline [19-direct-volume-ray-casting.md] to address specific application constraints, such as additive manufacturing simulation, interactive clipping, photorealistic medical imaging, and physically accurate shadowing. Unlike general-purpose renderers that accumulate optical depth along a single ray, these methods integrate auxiliary data structures (customization volumes), simulate complex light-matter interactions (cinematic rendering), or manage multi-pass compositing strategies (layers, shadows) to solve domain-specific visualization problems.

## Shared principles
The methods in this family share the foundational ray casting loop defined in [Image-order rendering](16-image-order-rendering.md) and [Direct volume ray casting](19-direct-volume-ray-casting.md), but modify one or more of the four core stages:

1.  **Ray Generation**: Rays may be generated from non-camera sources (e.g., projector emitters in TVAM [6814-tvam-integrator.md]) or as stochastic photon paths (Cinematic Rendering [9473-cinematic-rendering.md]).
2.  **Sampling**: Sampling may involve fetching from auxiliary volumes (Customization [5291-customization-volume-integration.md]) or querying shadow maps (CSSM [9816-colored-stochastic-shadow-mapping-for-volume-rendering.md]).
3.  **Classification**: The transfer function may be modulated by position-dependent parameters (gamma correction [5291-customization-volume-integration.md]) or applied within a Monte Carlo path tracing context (Cinematic Rendering [9473-cinematic-rendering.md]).
4.  **Compositing**: Accumulation may be split across multiple layers (Two Level with Layers) or modulated by shadow factors (CSSM [9816-colored-stochastic-shadow-mapping-for-volume-rendering.md]).

## Subtopics

### [Customization Volume Integration](5291-customization-volume-integration.md)
A method that integrates a pre-computed volume into the direct volume ray casting pipeline to apply position-dependent modifications to visualization parameters, such as gamma correction of intensity values. It differs from standard ray casting by introducing an auxiliary data structure (the customization volume) that modulates the input data before classification, allowing spatially varying adjustments to the rendering appearance. Choose this when spatially varying non-linear adjustments (e.g., local gamma) are required without re-baking the volume data.

### [TVAM Integrator](6814-tvam-integrator.md)
The TVAM Integrator is a base class for volumetric additive manufacturing (TVAM) simulations that handles projector ray sampling, time sampling, and regular versus stochastic sampling modes. It serves as the foundation for specialized integrators like the Radon Integrator and CornerIntegrator. Unlike general-purpose volume renderers, it is designed to simulate the optical projection of Digital Micromirror Devices (DMDs) onto a photosensitive resin vat, managing the specific constraints of time-multiplexed tomographic exposure. Choose this for simulating light exposure in 3D printing processes.

### [Cutting Plane Integration](8414-cutting-plane-integration.md)
Cutting Plane Integration is an interactive clipping technique that renders a semi-transparent plane mesh and normal arrow to dynamically slice a volume, revealing internal structures. It differs from standard ray casting by modifying the ray integration bounds in real-time based on a user-controlled geometric plane, rather than integrating through the entire volume extent. Choose this for interactive exploration of internal volume structures.

### [Cinematic Rendering](9473-cinematic-rendering.md)
Cinematic Rendering (CR) is a physiologically-based volume rendering method that simulates the interaction of visible photons with medical volume data (CT or MRI) to produce photorealistic 3D images. It differs from traditional Direct Volume Ray Casting (DVRC) by tracing hundreds or thousands of photon paths per pixel using Monte Carlo path tracing, rather than a single straight ray, thereby accounting for complex light-matter interactions such as scattering, absorption, multiple bounces, and global illumination effects. Unlike siblings in the Specific Applications family that accumulate dose (TVAM) or apply probabilistic occlusion (CSSM), CR solves the multi-dimensional, non-continuous rendering equation to integrate light scattered from all possible directions, modeling shadows, ambient occlusion, and subsurface scattering to enhance depth and shape perception. Choose this for photorealistic medical visualization where global illumination effects are critical.

### [Colored Stochastic Shadow Mapping for Volume Rendering](9816-colored-stochastic-shadow-mapping-for-volume-rendering.md)
Colored Stochastic Shadow Mapping (CSSM) is a two-step rendering technique for Direct Volume Rendering (DVR) that generates physically plausible, colored shadows by adapting stochastic shadow mapping concepts to volumetric data. Unlike traditional shadow maps that produce binary occlusion or deep shadow maps that require significant memory for piecewise linear approximations, CSSM stores a single stochastic depth sample per color channel, allowing translucent objects to cast partial, color-accurate shadows with minimal memory overhead. Choose this when colored shadows are required for translucent volumes with limited memory overhead.

### Two Level Volume Rendering with Layers
A technique that achieves results similar to traditional two-level volume rendering by rendering different styles (e.g., different transfer functions or rendering modes like MIP vs DVR) into separate layers and compositing them, rather than selecting rendering types per ray segment during integration [S1].
*   **Step**: Render each distinct material style or transfer function into a separate layer (e.g., DVR for soft tissue, DVR for bones) [S1]. Composite the separate layers into a final image using a compositing function that maps the resulting layers to the view volume [S1].
*   **Parameter**: Opacity: Layers can be assigned specific opacity values during rendering or compositing (e.g., layer 1 opacity 0.4, layer 3 opacity 0.1, layer 2 opacity 0.5) [S1].
*   **Comparison**: Unlike original two-level volume rendering which composites segments in the order the ray encounters materials (e.g., $I = A_1 B_1 A_2 B_2$), this method composites all segments of the same material separately first, then composites the intermediate results (e.g., $I = A_1 A_2 A_3 B_1 B_2$). Because compositing is not commutative, results generally differ [S1].

### Volumetric Ray Tracing Framework (Sobierajski/Kaufman)
A complete framework for volumetric ray tracing developed by Sobierajski and Kaufman that includes global illumination effects (shadows, reflections, light interaction between volumetric/geometric objects) and volumetric rendering effects (fog, transparency) [S2, unofficial]. It is incorporated into VolVis, a comprehensive volume visualization system developed at Stony Brook University [S2, unofficial].

## Background
The need for specific applications arose as volume rendering moved from scientific visualization to specialized domains like medical imaging and additive manufacturing. Cinematic Rendering was developed to address the lack of photorealism in standard DVR for medical diagnostics [S2, S3, unofficial]. CSSM was introduced to provide efficient, colored shadows for translucent volumes [S1]. The TVAM integrator addresses the specific physics of light-curing 3D printing [S1]. The "Layers" approach and Sobierajski/Kaufman framework represent early efforts to decouple material rendering from single-ray integration and incorporate global illumination, respectively [S1][S2, unofficial].

## Sources
[S1] Layers for Effective Volume Rendering, https://doi.org/10.2312/vg/vg-pbg08/081-088, fetched 2026-10-26
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
