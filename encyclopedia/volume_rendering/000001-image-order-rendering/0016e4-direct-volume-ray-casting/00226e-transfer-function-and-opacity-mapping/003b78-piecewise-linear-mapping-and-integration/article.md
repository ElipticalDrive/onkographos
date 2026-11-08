# Piecewise Linear Mapping and Integration
Also called: Piecewise Linear Transfer Function Integration

## Overview
This family defines methods for mapping scalar volume data to color and opacity using linear segments, often incorporating gradient information or specific integration strategies to handle discretization and surface enhancement. It distinguishes itself from non-linear alternatives (like [SDF-based Opacity Derivation via Logistic CDF](3233-sdf-based-opacity-derivation-via-logistic-cdf.md)) by relying on explicit linear interpolation between defined tissue thresholds, and from pure density approximations by often integrating opacity corrections directly into the transfer function evaluation or compositing step.

## Shared principles
All methods in this family map a scalar input $d$ (the interpolated volume value at a sample point) to a color vector $C$ and an opacity scalar $\alpha$ [S2, S5, S13, unofficial]. The mapping is defined by a transfer function $T(d) \rightarrow (C, \alpha)$ [S2, S2789]. The opacity $\alpha$ is typically bounded to $[0, 1]$ [G]. The resulting opacity is often corrected for the discretization step size $\Delta t$ before compositing to ensure physical consistency, using the relation $\alpha' = 1 - (1 - \alpha)^{\Delta t}$ or similar approximations [S11, S32, unofficial]. The color $C$ is usually independent of the step size [G].

## Subtopics

### [Piecewise Linear Opacity Mapping with Gradient Scaling](1335-piecewise-linear-opacity-mapping-with-gradient-scaling.md)
A technique that maps scalar voxel values to opacity using tissue-specific linear segments and then suppresses interior opacity by scaling with the local gradient magnitude, thereby enhancing bounding surfaces. Unlike [Linear Opacity Interpolation](7016-linear-opacity-interpolation.md), which maps values directly to opacity without gradient weighting, this method ensures that thin structures appear as faint wisps rather than holes and that semi-transparent surfaces (e.g., skin over bone) are clearly distinguished from solid interiors. See [Piecewise Linear Opacity Mapping with Gradient Scaling](1335-piecewise-linear-opacity-mapping-with-gradient-scaling.md).

### [Linear Opacity Interpolation](7016-linear-opacity-interpolation.md)
Maps scalar volume data values to opacity using linear interpolation between two threshold values. It serves as the foundational opacity mapping primitive in piecewise linear transfer functions, providing a simple, computationally efficient way to define opacity ramps. Unlike its sibling [Piecewise Linear Opacity Mapping with Gradient Scaling](1335-piecewise-linear-opacity-mapping-with-gradient-scaling.md), this method operates solely on scalar values without incorporating gradient magnitude scaling, making it suitable for homogeneous regions or when boundary emphasis is handled separately. See [Linear Opacity Interpolation](7016-linear-opacity-interpolation.md).

### Piecewise Linear Transfer Function Integration
A rendering technique that integrates piecewise linear transfer functions using a 2D texture map to compute correct exponential per-pixel opacity [S1, unofficial]. The method uses a 2D texture map parameterized by opacity and thickness to determine the final integrated opacity for a ray segment [S1, unofficial].

## Background
The concept of piecewise linear transfer functions has been a staple in volume rendering since the early days of direct volume rendering, allowing for intuitive user interaction by defining opacity ramps between known tissue types (e.g., CT numbers). The integration of these functions, particularly the handling of the exponential nature of light attenuation through discrete samples, has been refined to support hardware acceleration. Early implementations relied on pre-computed tables or simple linear interpolation, while later approaches, such as those described in GPU-based rendering surveys, utilized 2D texture maps parameterized by opacity and thickness to accelerate the integration of piecewise linear transfer functions [S1, unofficial]. This family of methods remains fundamental in interactive volume rendering systems due to its computational efficiency and interpretability.

## Sources
[S1] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
