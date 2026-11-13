# Style Transfer and Illustrative Rendering

Also called: Style Transfer Functions (STF), Illustrative Volume Rendering

## Overview
This family extends direct volume ray casting by decoupling visual appearance from physical density, mapping scalar values to artistic styles via surface normals and material indices rather than photorealistic lighting [S1, 11409-raycasting-with-style-transfer-and-contour-lines.md, 11445-illustrative-volume-rendering-with-style-transfer.md]. It distinguishes itself from standard ray casting by using 2D array textures (lit-sphere maps) to apply pre-computed artistic styles and optional contour lines, creating non-photorealistic illustrations [11409-raycasting-with-style-transfer-and-contour-lines.md, 11445-illustrative-volume-rendering-with-style-transfer.md].

## Shared principles
All methods in this family share the following core mechanism, building on [Transfer Function and Opacity Mapping](2789-transfer-function-and-opacity-mapping.md):

1.  **Input Domain**: The method operates on scalar volume values $V(\mathbf{p})$ retrieved at sample positions $\mathbf{p}$ along the ray [S2, 2789].
2.  **Mapping Function**: A transfer function $T$ maps the scalar value $V$ to a color vector $\mathbf{c} \in [0,1]^3$ and an opacity scalar $\alpha \in [0,1]$ [S2, 2789].
    $$ T: V \rightarrow (\mathbf{c}, \alpha) $$
3.  **Composition**: The resulting $\mathbf{c}$ and $\alpha$ are used in the front-to-back compositing equation defined in the parent article [19-direct-volume-ray-casting.md] to compute the final pixel color [S2, 2789].
4.  **Parameterization**: Transfer functions are typically parameterized by control points $(V_k, \mathbf{c}_k, \alpha_k)$, with interpolation (linear, cubic, or piecewise) between points [S2, 2789].

## Subtopics

### [Raycasting with Style Transfer and Contour Lines](11409-raycasting-with-style-transfer-and-contour-lines.md)
A direct volume rendering technique that applies artistic styles based on surface normals and material indices, optionally rendering contour lines to emphasize structural features [11409-raycasting-with-style-transfer-and-contour-lines.md]. It distinguishes itself by decoupling material appearance from density via a two-stage indexing system (material index to style index) and enhancing results with curvature-based contour detection [11409-raycasting-with-style-transfer-and-contour-lines.md]. Choose this when structural edges need explicit highlighting alongside artistic styling [11409-raycasting-with-style-transfer-and-contour-lines.md]. [11409-raycasting-with-style-transfer-and-contour-lines.md](11409-raycasting-with-style-transfer-and-contour-lines.md)

### [Illustrative Volume Rendering with Style Transfer](11445-illustrative-volume-rendering-with-style-transfer.md)
Combines standard direct volume ray casting with a Style Transfer Function (STF) to apply artistic, non-photorealistic styles to volumetric data [11445-illustrative-volume-rendering-with-style-transfer.md]. It distinguishes itself by decoupling visual appearance from physical density using a 2D array texture to map surface normals and material types to pre-computed artistic styles [11445-illustrative-volume-rendering-with-style-transfer.md]. Choose this for general illustrative rendering without specific contour line requirements [11445-illustrative-volume-rendering-with-style-transfer.md]. [11445-illustrative-volume-rendering-with-style-transfer.md](11445-illustrative-volume-rendering-with-style-transfer.md)

### Illustrative Transparency
A transparency model combining uniform opacity ($\alpha_u$) from the transfer function and directional opacity ($\alpha_d$) from the style map, modulated by curvature and gradient magnitude to simulate the 100-percent-rule [S1].
- If $\alpha_u = 1$, the sample opacity is solely determined by $\alpha_d$ [S1].
- If $\alpha_u = 0$, the sample is completely transparent [S1].
- Compute the term $\max(0, |n \cdot v| - \sqrt{T \kappa_v (2 - T \kappa_v)})$, which is zero on the contour and increases away from it [S1].
- Weight the result by $\alpha_d$ to apply the final opacity [S1].
- Overall opacity $\alpha = \alpha_d \cdot \alpha_u \cdot \frac{0.5 + \max(0, |n \cdot v| - \sqrt{T \kappa_v (2 - T \kappa_v)}) \cdot (1 - |g|)}{1}$, where $g$ is the gradient magnitude, $n$ is normal, $v$ is view vector, and $T \kappa_v (2 - T \kappa_v)$ is the contour threshold term [S1].
- Parameters: $\alpha_u$ (uniform opacity from transfer function), $\alpha_d$ (directional opacity tuple from lit sphere map), and 0.5 (constant restricting maximum opacity enhancement, empirically determined) [S1].

## Background
Originated with Bruckner and Gröller in 2007 with the paper "Style Transfer Functions for Illustrative Volume Rendering" [S1]. The method was implemented in C++ and OpenGL/GLSL, leveraging Shader Model 3.0 for conditional loops and dynamic branching [S1]. It allows users to load volumetric datasets, define transfer functions mapping density to material indices and styles, and interactively rotate and zoom the rendered result [11445-illustrative-volume-rendering-with-style-transfer.md].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
