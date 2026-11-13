# Style Transfer and Illustrative Rendering

Also called: Style Transfer Functions (STF), Illustrative Volume Rendering with Style Transfer

## Overview
This family replaces physical lighting models with artistic style maps (e.g., lit spheres, matcaps) to determine surface color, decoupling visual appearance from geometric illumination [S1]. It distinguishes itself from standard ray casting by mapping surface normals to 2D texture coordinates for color retrieval, and from siblings by offering specific architectural choices for boundary handling (two-pass exit point textures) and contour integration (curvature-based masking within the ray loop) [11409-raycasting-with-style-transfer-and-contour-lines.md, 11445-illustrative-volume-rendering-with-style-transfer.md].

## Shared principles
All methods in this family share the following core mechanism, extending the parent's classification and compositing steps:

1.  **Style Transfer Function (STF) Lookup**: Instead of a standard transfer function returning a fixed color $\mathbf{c}$, the STF returns a **style index** $k$ and potentially a **directional opacity** $\alpha_d$ [S1, 11445-illustrative-volume-rendering-with-style-transfer.md]. The color $\mathbf{c}_i$ is derived by sampling a **Style Transfer Texture** (typically a 2D array of lit sphere or matcap images) using coordinates derived from the surface normal $\mathbf{n}_i$ [S1, 11409-raycasting-with-style-transfer-and-contour-lines.md].
2.  **Normal-to-UV Mapping**: The surface normal $\mathbf{n}_i$ (computed from the volume gradient) is mapped to 2D UV coordinates for the style texture. A common mapping for lit spheres is:
    $$
    \mathbf{uv} = (\mathbf{n}_i.x, -\mathbf{n}_i.y) \cdot 0.5 + 0.5
    $$
    This maps the hemisphere of normals to the unit square [11409-raycasting-with-style-transfer-and-contour-lines.md].
3.  **Index Mapping**: A scalar volume value $V(\mathbf{p}_i)$ is mapped to a material index $i$ via a 1D **Index Function Texture**, which then maps $i$ to a discrete style index $k$ [S1, 11445-illustrative-volume-rendering-with-style-transfer.md].
4.  **Compositing**: The sampled style color $\mathbf{c}_{base}$ is combined with opacity $\alpha_i$ and composited using the standard front-to-back equation [11409-raycasting-with-style-transfer-and-contour-lines.md].

## Subtopics

### [Raycasting with Style Transfer and Contour Lines](11409-raycasting-with-style-transfer-and-contour-lines.md)
This method performs direct volume ray casting where surface color is derived from 2D style textures mapped via surface normals [11409-raycasting-with-style-transfer-and-contour-lines.md]. It distinguishes itself by explicitly integrating curvature-based contour masking into the ray loop to simulate illustrative effects like the "100-percent-rule" [11409-raycasting-with-style-transfer-and-contour-lines.md]. Choose this when you need single-pass rendering with integrated contour lines.

### [Illustrative Volume Rendering with Style Transfer](11445-illustrative-volume-rendering-with-style-transfer.md)
This method combines direct volume ray casting with a Style Transfer Function (STF) to apply artistic styles [11445-illustrative-volume-rendering-with-style-transfer.md]. It is distinguished by its specific two-pass architecture using a back-face exit point texture to drive ray length, ensuring precise boundary handling [11445-illustrative-volume-rendering-with-style-transfer.md]. Choose this when precise ray length calculation via back-face intersection is required or when separating boundary detection from volume integration improves performance.

### Illustrative Transparency
A transparency model combining uniform opacity ($\alpha_u$) from the transfer function and directional opacity ($\alpha_d$) from the style map, modulated by curvature and gradient magnitude to simulate the 100-percent-rule [S1].

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
Originating from Bruckner and Gröller's work on Style Transfer Functions for Illustrative Volume Rendering [S1], this approach emerged to address the limitations of physical lighting in visualizing complex volumetric structures. It allows artists and scientists to apply non-photorealistic styles (e.g., cartoon shading, matcaps) directly to volume data, enhancing the perception of shape and transparency through illustrative techniques like the 100-percent-rule [S1]. The method is implemented using GPU-based ray casting, leveraging Shader Model 3.0 features for dynamic branching and texture lookups [S1].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
