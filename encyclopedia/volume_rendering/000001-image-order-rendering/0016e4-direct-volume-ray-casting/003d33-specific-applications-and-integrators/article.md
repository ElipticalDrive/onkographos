# Specific Applications and Integrators

Also called: Domain-specific volume integration, specialized ray casting integrators

## Overview
This family encompasses direct volume ray casting methods adapted for specific physical simulations, medical photorealism, or advanced lighting effects, rather than standard visual inspection. Unlike standard DVR which maps scalar data to color/opacity for aesthetic or diagnostic clarity, these methods integrate physical quantities (polymerization dose), simulate complex light transport (photon scattering), or modify rendering pipelines for specific output constraints (colored shadows). They differ from siblings like [Neural volume rendering](1576-neural-rendering.md) by relying on explicit physical models or deterministic ray tracing rather than learned implicit representations.

## Shared principles
All methods in this family adhere to the core direct volume ray casting loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md): ray generation, sampling, interpolation, classification, and compositing [G]. However, they diverge in the **Classification** and **Compositing** stages:
1.  **Classification**: The transfer function may map to physical quantities (e.g., dose) rather than just color/opacity, or apply position-dependent modifications (e.g., gamma correction) before the standard transfer function [S1, S5291].
2.  **Compositing**: The accumulation equation may be replaced or augmented. For physical integrators, it accumulates dose rather than radiance [S6814]. For cinematic rendering, it uses Monte Carlo path tracing to solve the rendering equation with multiple bounces [S9473]. For shadow mapping, it incorporates probabilistic occlusion factors [S9816].

## Subtopics

### [Customization Volume Integration](5291-customization-volume-integration.md)
A method for applying position-dependent modifications to visualization parameters within a direct volume ray casting pipeline. It solves the problem of static, global transfer function settings by allowing spatially varying adjustments (e.g., local gamma correction) via a secondary "customization volume". Unlike sibling methods that modify the integrand's physical model (e.g., TVAM for polymerization or Cinematic Rendering for light transport), this method operates as a post-sampling, pre-transfer-function modifier, requiring only minor pipeline changes.

### [TVAM Integrator](6814-tvam-integrator.md)
The TVAM Integrator is a specialized volumetric integrator for Tomographic Volumetric Additive Manufacturing (TVAM) that serves as the base class for domain-specific ray casting methods. It solves the problem of accurately simulating light transport and polymerization dose accumulation in a resin vat by handling projector ray sampling, time sampling, and the distinction between regular and stochastic sampling modes. Unlike standard volume rendering integrators that compute color, this method computes physical quantities (such as polymerization dose) and supports complex loss functions for inverse design. It is distinguished from its siblings, such as the [Radon Integrator](7470-radon-integrator.md) and [CornerIntegrator](7445-cornerintegrator.md), by providing the foundational infrastructure for ray generation and scene parsing, which these subclasses extend to handle specific boundary conditions or projection transforms.

### [Cinematic Rendering](9473-cinematic-rendering.md)
Cinematic Rendering (CR) is a physiologically-based volume rendering method that simulates the interaction of visible photons with medical volume data (CT or MRI) to produce photorealistic 3D images. It differs from traditional Direct Volume Ray Casting (DVRC) by tracing hundreds or thousands of photon paths per pixel rather than a single straight ray, thereby accounting for scattering, absorption, multiple bounces, and complex lighting effects like shadows and ambient occlusion. While DVRC relies on local gradient shading and synthetic light sources, CR solves the multi-dimensional rendering equation using path tracing and High Dynamic Range (HDR) light maps to integrate light scattered from all possible directions.

### [Colored Stochastic Shadow Mapping for Volume Rendering](9816-colored-stochastic-shadow-mapping-for-volume-rendering.md)
Colored Stochastic Shadow Mapping (CSSM) is a two-step rendering technique for Direct Volume Rendering (DVR) that generates colored shadows by adapting stochastic shadow mapping concepts to volumetric data. Unlike traditional shadow maps that use binary occlusion or deep shadow maps that store piecewise linear approximations, CSSM uses probabilistic events to represent photon traversal through a volume with changing opacities, allowing translucent objects to cast partial, color-accurate shadows. It sets itself apart by storing only one depth value per color channel per fragment (stochastic sample) rather than multiple control points, resulting in significantly lower memory usage than Deep Shadow Maps (DSM) while producing higher-quality colored shadows than Traditional Shadow Maps (TSM).

### Cutting Plane Integration
- **Definition**: An interactive clipping technique that renders a semi-transparent plane mesh and normal arrow, dynamically updating raycasting uniforms to slice the volume and reveal internal structures [S1, unofficial].
- **Step**: Map normalized height to world Z position: `planeMesh.position.z = 200 - 400 * planeHeight`. Convert degrees to radians and apply rotation: `rotY` includes `Math.PI` offset if `renderAbove` is true [S1, unofficial].
- **Parameter**: `planeHeight`: normalized float (0 to 1) mapping to Z-axis position. `planeRotX`/`planeRotY`: rotation angles in degrees around X and Y axes. `renderAbove`: boolean flag that flips the plane 180° around Y when true [S1, unofficial].

### Two Level Volume Rendering with Layers
- **Definition**: A technique that achieves results similar to traditional two-level volume rendering by rendering different styles (e.g., different transfer functions or rendering modes like MIP vs DVR) into separate layers and compositing them, rather than selecting rendering types per ray segment during integration [S2].
- **Step**: Render each distinct material style or transfer function into a separate layer (e.g., DVR for soft tissue, DVR for bones). Composite the separate layers into a final image using a compositing function that maps the resulting layers to the view volume [S2].
- **Parameter**: Opacity: Layers can be assigned specific opacity values during rendering or compositing (e.g., layer 1 opacity 0.4, layer 3 opacity 0.1, layer 2 opacity 0.5) [S2].
- **Comparison**: Unlike original two-level volume rendering which composites segments in the order the ray encounters materials (e.g., $I = A_1 B_1 A_2 B_2$), this method composites all segments of the same material separately first, then composites the intermediate results (e.g., $I = A_1 A_2 A_3 B_1 B_2$). Because compositing is not commutative, results generally differ [S2].

### Volumetric Ray Tracing Framework (Sobierajski/Kaufman)
- **Definition**: A complete framework for volumetric ray tracing developed by Sobierajski and Kaufman that includes global illumination effects (shadows, reflections, light interaction between volumetric/geometric objects) and volumetric rendering effects (fog, transparency) [S3, unofficial].
- **Usage**: Incorporated into VolVis, a comprehensive volume visualization system developed at Stony Brook University [S3, unofficial].

## Background
The concept of specific applications in volume rendering emerged as the basic direct volume ray casting technique matured in the 1990s. Early systems focused on general visualization [S3, unofficial]. The need for domain-specific integrators arose with the advent of additive manufacturing, requiring accurate simulation of light-polymer interactions (TVAM) [S6814]. Cinematic Rendering was developed for medical diagnostics to overcome the limitations of synthetic lighting in standard DVR, aiming for photorealism comparable to photography [S2][S3, unofficial]. Colored Stochastic Shadow Mapping addressed the lack of realistic shadowing in translucent volumes, adapting techniques from geometric rendering [S1, unofficial]. The Sobierajski/Kaufman framework represented an early attempt to unify global illumination and volumetric rendering in a single system [S3, unofficial].

## Sources
[S1] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
[S2] Layers for Effective Volume Rendering, https://doi.org/10.2312/vg/vg-pbg08/081-088, fetched 2026-10-26
[S3] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
