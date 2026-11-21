# Style Transfer and Illustrative Rendering

Also called: Style Transfer Functions, STF

## Overview
Style Transfer and Illustrative Rendering applies non-photorealistic artistic styles to volumetric data by mapping volume values to discrete style indices and surface normals to 2D style textures (e.g., lit spheres or matcaps) [S1, 11445-illustrative-volume-rendering-with-style-transfer.md]. It distinguishes itself from standard direct volume raycasting by replacing fixed color transfer functions with a Style Transfer Texture lookup driven by normal-to-UV mapping, and from basic illustrative rendering by integrating curvature-based contour detection and illustrative transparency models directly into the raycasting loop [S1, 11409-raycasting-with-style-transfer-and-contour-lines.md].

## Shared principles
All methods in this family share the following core mechanism:

1.  **Input Domain**: The input is a scalar value $V$ retrieved from the volume at a sample position. This value may be raw, normalized, or derived from a Signed Distance Function (SDF) [S1, S2, S3, S4, unofficial][S6].
2.  **Mapping Function**: A function $T(V)$ computes the optical properties. The most common form is a piecewise function defined by control points $(v_k, \mathbf{c}_k, \alpha_k)$ [S1, S2, S9000].
3.  **Opacity Correction**: The raw opacity $\alpha_{raw}$ derived from $T(V)$ often requires correction for the sampling step size $\Delta t$ to approximate continuous attenuation. The Beer-Lambert law is the standard physical model: $\alpha_i = 1 - e^{-\sigma_i \Delta t}$, where $\sigma_i$ is the extinction coefficient [S26, S33, S39, unofficial]. Alternative corrections include linear scaling for slice count invariance [S1277] or logistic CDF mapping for SDFs [S3233].
4.  **Gradient Modulation**: Many methods augment the base opacity and color with information derived from the local gradient $\nabla V(\mathbf{p}_i)$, which serves as a surface normal. This is used for shading [S8281], silhouette enhancement [S8222], or gradient-based opacity [S13088].
5.  **Output**: The result is a tuple $(\mathbf{c}_i, \alpha_i)$ passed to the compositing stage [S2789].

## Subtopics

### [Raycasting with Style Transfer and Contour Lines](11409-raycasting-with-style-transfer-and-contour-lines.md)
Raycasting with Style Transfer and Contour Lines is a direct volume rendering technique that applies artistic styles to volumetric data based on surface normals and material indices, while optionally rendering contour lines to emphasize structural features. It distinguishes itself from standard direct volume raycasting by replacing fixed color transfer functions with a Style Transfer Texture lookup driven by normal-to-UV mapping, and from basic illustrative rendering by integrating curvature-based contour detection directly into the raycasting loop.

### [Illustrative Volume Rendering with Style Transfer](11445-illustrative-volume-rendering-with-style-transfer.md)
This method applies artistic styles to volumetric data by combining standard direct volume ray casting with a Style Transfer Function (STF). It solves the problem of visualizing scalar fields with non-photorealistic aesthetics by mapping volume values to discrete style indices and surface normals to 2D style textures. It builds on [Style Transfer and Illustrative Rendering](13031-style-transfer-and-illustrative-rendering.md) by implementing a specific two-pass rendering pipeline that captures exit points for accurate ray length calculation.

### Illustrative Transparency
#### definition
- [S1] A transparency model combining uniform opacity ($\alpha_u$) from the transfer function and directional opacity ($\alpha_d$) from the style map, modulated by curvature and gradient magnitude to simulate the 100-percent-rule.
#### step
- [S1] If $\alpha_u = 1$, the sample opacity is solely determined by $\alpha_d$.
- [S1] If $\alpha_u = 0$, the sample is completely transparent.
- [S1] Compute the term $\max(0, |n \cdot v| - \sqrt{T \kappa_v (2 - T \kappa_v)})$, which is zero on the contour and increases away from it.
- [S1] Weight the result by $\alpha_d$ to apply the final opacity.
#### formula
- [S1] Overall opacity $\alpha = \alpha_d \cdot \alpha_u \cdot \frac{0.5 + \max(0, |n \cdot v| - \sqrt{T \kappa_v (2 - T \kappa_v)}) \cdot (1 - |g|)}{1}$, where $g$ is the gradient magnitude, $n$ is normal, $v$ is view vector, and $T \kappa_v (2 - T \kappa_v)$ is the contour threshold term.
#### parameter
- [S1] $\alpha_u$: Uniform opacity value from the transfer function, controlling overall opacity independent of normal direction.
- [S1] $\alpha_d$: Directional opacity tuple (r,g,b,$\alpha$) from the lit sphere map, varying opacity based on normal direction.
- [S1] 0.5: Constant restricting maximum opacity enhancement, empirically determined.

## Background
The concept of Style Transfer Functions for Illustrative Volume Rendering was introduced by Stefan Bruckner and Eduard Gröller in 2007 [S1]. The method was implemented in C++ and OpenGL/GLSL, leveraging Shader Model 3.0 features such as conditional loops and dynamic branching [S1]. It allows for the integration of illustrative techniques, such as contour lines and variable transparency, into existing GPU-based ray casting renderers with minimal effort [S1].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
