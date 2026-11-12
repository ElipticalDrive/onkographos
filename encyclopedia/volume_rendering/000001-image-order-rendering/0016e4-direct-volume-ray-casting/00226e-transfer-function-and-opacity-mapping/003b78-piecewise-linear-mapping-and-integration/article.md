# Piecewise Linear Mapping and Integration

Also called: Piecewise Linear Transfer Functions, PLTF

## Overview
This family defines opacity $\alpha$ and color $\mathbf{c}$ mappings where the transfer function is constructed from linear segments between control points. It solves the problem of providing intuitive, interactive control over material visibility by allowing artists to define specific opacity and color values at specific scalar thresholds. It differs from [Opacity Correction for Slice Count Invariance](1277-opacity-correction-for-slice-count-invariance.md) by focusing on the definition of the mapping function itself rather than the correction of integration weights, and from [Piecewise Linear Mapping and Integration](9000-piecewise-linear-mapping-and-integration.md) (the specific algorithmic child) by encompassing broader variations like gradient scaling and power function pre-mapping.

## Shared principles
All methods in this family map a scalar input $d$ (the interpolated volume value at a sample point) to a color vector $\mathbf{c}$ and an opacity scalar $\alpha$ [S2, S5, S13, unofficial]. The mapping is defined by a transfer function $T(d) \rightarrow (\mathbf{c}, \alpha)$ [S2, 2789, unofficial]. The opacity $\alpha$ is typically bounded to $[0, 1]$ [G]. The resulting opacity is often corrected for the discretization step size $\Delta t$ before compositing to ensure physical consistency, using the relation $\alpha' = 1 - (1 - \alpha)^{\Delta t}$ or similar approximations [S11, S32, unofficial]. The color $\mathbf{c}$ is usually independent of the step size [G].

## Subtopics

### [Piecewise Density Approximation for Reparameterized Sampling](1139-piecewise-density-approximation-for-reparameterized-sampling.md)
Enables analytical inversion of the opacity function by approximating density as piecewise constant or linear, solving for sample locations without numerical root-finding. Choose when reparameterized sampling is required for gradient-based optimization.

### [Piecewise Linear Opacity Mapping with Gradient Scaling](1335-piecewise-linear-opacity-mapping-with-gradient-scaling.md)
Maps opacity via piecewise linear functions and scales by gradient magnitude to suppress interiors and enhance surfaces. Choose for visualizing thin structures and semi-transparent surface overlays.

### [Linear Opacity Interpolation](7016-linear-opacity-interpolation.md)
The foundational primitive mapping opacity linearly between two thresholds. Choose for simple, computationally inexpensive scalar-intensity-based segmentation.

### Power Function Opacity Transfer
- **Definition**: A simple power function used to map voxel values to opacity before enhancement, allowing illustrative enhancements to supplement existing volume rendering mechanisms [S1].
- **Step**: Calculate the gradient used for enhancement from the dataset *after* the simple power function transfer function has been applied, as this gradient is more effective in capturing subtle features [S1].
- **Formula**: $o_v = (k_{os}v_i)^{k_{oe}}$, where $v_i$ is the volume sample value, $k_{os}$ is the scalar controlling maximum opacity, and $k_{oe}$ is the exponent [S1].
- **Parameters**: $k_{os}$ controls maximum opacity; $k_{oe}$ controls contrast. Values of $k_{oe} < 1$ soften volume differences; values $> 1$ increase contrast [S1].

### Piecewise Linear Transfer Function Integration
- **Definition**: A rendering technique that integrates piecewise linear transfer functions using a 2D texture map to compute correct exponential per-pixel opacity [S2, unofficial].
- **Parameters**: Texture Map: A 2D texture map parameterized by opacity and thickness [S2, unofficial].

## Background
The concept of piecewise linear transfer functions is foundational in direct volume rendering, allowing for interactive definition of material properties. The specific integration technique using 2D texture maps parameterized by opacity and thickness was described by Williams et al. [S2, unofficial]. The use of power functions for opacity mapping to enhance subtle features via gradient calculation was introduced in the context of non-photorealistic volume illustration [S1].

## Sources
[S1] Volume illustration: nonphotorealistic rendering of volume models, https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
