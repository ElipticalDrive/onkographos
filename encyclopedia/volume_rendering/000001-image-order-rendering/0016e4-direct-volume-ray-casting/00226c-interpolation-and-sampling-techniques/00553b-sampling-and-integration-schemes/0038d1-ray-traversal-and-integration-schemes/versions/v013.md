# Ray Traversal and Integration Schemes

Also called: Ray Casting and Compositing, Volume Ray Marching, Image-Order Integration

## Overview
Ray Traversal and Integration Schemes define the computational framework for sampling volumetric data along camera rays and aggregating those samples into final pixel colors. This family solves the problem of mapping discrete 3D scalar or vector fields to 2D screen-space images by numerically integrating the volume rendering equation. It is distinguished from object-order methods (like ray tracing of implicit surfaces) by its pixel-parallel, image-order nature, and from Monte Carlo methods by its deterministic, grid-aligned traversal logic.

## Shared principles
All methods in this family operate on the principle that a continuous function $V(\mathbf{p})$ must be approximated from a discrete set of samples $V_i$ located at grid positions $\mathbf{p}_i$. The core mechanism involves:

1.  **Query Positioning**: Determining the 3D coordinate $\mathbf{p}$ along the ray $r(t) = \mathbf{o} + t\mathbf{d}$ where a value is needed. This may involve converting depth parameters to physical distances or warping coordinates for neural encodings.
2.  **Kernel/Weight Application**: Computing weights $w_i(\mathbf{p})$ based on the distance between $\mathbf{p}$ and the sample centers $\mathbf{p}_i$. The interpolated value is:
    $$
    V(\mathbf{p}) = \sum_{i} w_i(\mathbf{p}) V_i
    $$
    The choice of kernel (e.g., linear, cubic, inverse distance) determines the spatial extent (number of neighbors) and frequency response (aliasing vs. blurring).
3.  **Coordinate Transformation**: Mapping world-space or camera-space coordinates to the local coordinate system of the volume data (e.g., normalized $[-1, 1]$ or $[0, 1]$ ranges) before interpolation.

## Subtopics

### [Ray Marching and Discretization](2788-ray-marching-and-discretization.md)
Iterative numerical algorithm approximating the volume integral via discrete steps and alpha compositing. Chosen for real-time performance on GPUs.

### [Image-order Rendering Loop](2938-image-order-rendering-loop.md)
Fundamental control structure iterating over pixels to generate rays. Chosen for correct perspective projection and single-pass execution.

### [Front-to-Back Volume Ray Casting with HSV Shading and SOS/Attenuation/Reflection Filtering](4324-front-to-back-volume-ray-casting-with-hsv-shading-and-sos-at.md)
Direct volume rendering with strict filtering on SOS, Attenuation, and Reflection values, and HSV color mapping. Chosen for specific scientific visualization pipelines requiring component-based filtering.

### [LOS-integration for visualization](4807-los-integration-for-visualization.md)
Computes scalar physical quantities (e.g., column density) via direct line-of-sight integration. Chosen for solar physics to avoid chromospheric artifacts.

### [Monte Carlo Path Tracing for Volume Rendering](4980-monte-carlo-path-tracing-for-volume-rendering.md)
Stochastic framework using Woodcock tracking for light attenuation and shadows. Chosen for photo-realistic rendering with complex lighting and depth-of-field.

### [Ray Traversal Schemes](6890-ray-traversal-schemes.md)
Defines aggregation logic (e.g., Maximum, Average) for voxel samples. Chosen for non-compositional visualizations or X-ray simulation.

### [Ray Casting and Compositing Loop](9826-ray-casting-and-compositing-loop.md)
Single-pass GPU method managing traversal and compositing state explicitly. Chosen for deterministic, uniform step-size rendering without surface extraction.

### [Ray Casting with Ray Subdividing](11392-ray-casting-with-ray-subdividing.md)
Dynamically splits rays to maintain sampling density under perspective divergence. Chosen to prevent aliasing in distant volume slices.

### [Cone Casting with Inscribed Sphere Sampling](11501-cone-casting-with-inscribed-sphere-sampling.md)
Emits cones from the camera to sample isotropic spheres, avoiding distance-agnostic aliasing. Chosen for neural field rendering with pre-filtered encodings.

### [Raycasting Phase](16581-raycasting-phase.md)
Two-pass geometry-based traversal using back-face depth buffering. Chosen for parallel implementations where entry/exit points are determined via geometry.

### Adaptive Segment Length
**Definition**: A strategy to optimize speed while reducing artifacts by varying segment length based on contribution to ray color [S1, unofficial].
**Steps**: Use smaller segments when the contribution to the ray color is large, and longer segments when the contribution is low [S1, unofficial].

## Background
Originating from the foundational work on direct volume rendering in the late 1980s and early 1990s, ray traversal methods evolved from simple fixed-step marching to adaptive and stochastic techniques. Early implementations relied on fixed step sizes and front-to-back compositing [S1, unofficial]. The introduction of GPU fragment shaders enabled real-time image-order rendering loops . Later developments incorporated adaptive subdivision for perspective aliasing [S1, unofficial] and Monte Carlo techniques for realistic lighting [S1, unofficial].

## Sources
[S1] Common Artifacts in Volume Rendering, https://doi.org/10.48550/arxiv.2109.13704, fetched 2026-10-26 (unofficial)
