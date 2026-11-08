# Piecewise Linear Mapping and Integration

Also called: Piecewise Linear Transfer Function Integration

## Overview
This family defines opacity $\alpha$ as a piecewise linear function of the scalar volume data $d$, often combined with gradient information to modulate visibility. It solves the problem of defining smooth, continuous opacity transitions between distinct tissue types or material classes in direct volume ray casting. Unlike [SDF-based Opacity Derivation via Logistic CDF](3233-sdf-based-opacity-derivation-via-logistic-cdf.md) which uses probabilistic cumulative distribution functions, or [Classified Volume Rendering](3590-classified-volume-rendering.md) which uses discrete hard thresholds, this family provides computationally inexpensive, linear interpolation between defined control points, optionally scaled by local geometric features [S1, unofficial][S2, unofficial].

## Shared principles
All methods in this family map a scalar input $d$ to an opacity $\alpha$ using linear segments. The core mechanism involves defining control points $(d_i, \alpha_i)$ and interpolating between them. When gradient scaling is employed, the base linear opacity is multiplied by the local gradient magnitude $|\nabla d|$ to emphasize boundaries [S1, unofficial][S2, unofficial]. The resulting opacity is corrected for the discretization step size $\Delta t$ before compositing to ensure physical consistency, using the relation $\alpha' = 1 - (1 - \alpha)^{\Delta t}$ or similar approximations [S11, S32, unofficial]. The color $C$ is usually independent of the step size [G].

## Subtopics

### [Piecewise Linear Opacity Mapping with Gradient Scaling](1335-piecewise-linear-opacity-mapping-with-gradient-scaling.md)
This method maps voxel scalar values to opacity using piecewise linear functions defined by tissue types, then scales the resulting opacity by the magnitude of the local gradient vector to suppress tissue interiors and enhance bounding surfaces. Unlike [Linear Opacity Interpolation](7016-linear-opacity-interpolation.md), which maps values directly to opacity without geometric weighting, this technique ensures that thin structures appear as faint wisps rather than holes, and that surface boundaries are visually emphasized over homogeneous regions.

### [Linear Opacity Interpolation](7016-linear-opacity-interpolation.md)
Linear Opacity Interpolation maps scalar volume data values to opacity using linear interpolation between two threshold values. It serves as the foundational opacity mapping technique within the Piecewise Linear Mapping and Integration family, providing a simple, computationally inexpensive method to define opacity transitions. Unlike its sibling [Piecewise Linear Opacity Mapping with Gradient Scaling](1335-piecewise-linear-opacity-mapping-with-gradient-scaling.md), which incorporates gradient magnitude to modulate opacity, this method relies solely on the scalar data value.

### Piecewise Linear Transfer Function Integration
A rendering technique that integrates piecewise linear transfer functions using a 2D texture map to compute correct exponential per-pixel opacity [S1, unofficial]. The method uses a 2D texture map parameterized by opacity and thickness to determine the final integrated opacity for a ray segment [S1, unofficial].

## Background
The concept of piecewise linear transfer functions has been widely used in volume rendering to provide intuitive control over material appearance. The integration of these functions, particularly using hardware-accelerated texture maps for exponential opacity calculation, was described by Williams et al. [S1, unofficial]. This approach allows for efficient rendering on GPU architectures by leveraging fragment processing capabilities.

## Sources
[S1] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
