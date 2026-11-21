# Style Transfer and Illustrative Rendering

Also called: Style Transfer Functions (STF), Illustrative Volume Rendering

## Overview
This family of methods augments direct volume ray casting with artistic style transfer and illustrative techniques (such as contour lines and non-photorealistic transparency) to produce stylized visualizations rather than photorealistic ones. It differs from standard ray casting by decoupling the material appearance from the scalar data density, mapping surface normals and material indices to 2D array textures of pre-rendered artistic styles, and applying opacity models that simulate illustrative conventions like the 100-percent-rule [S1, 11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].

## Shared principles
All methods in this family share the core ray casting mechanism defined in [Transfer Function and Opacity Mapping](2789-transfer-function-and-opacity-mapping.md). The key extension is the **Style Transfer Function (STF)**, which replaces or augments the standard color/opacity lookup.

1.  **Style Indexing**: Instead of mapping volume density $V$ directly to color $\mathbf{c}$, the method maps $V$ to a **material index** $idx$ (via a transfer function texture) and then maps $idx$ to a **style index** $s$ (via an index function texture) [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].
2.  **Lit Sphere Mapping**: The surface normal $\mathbf{n}$ at each sample is projected onto a 2D "lit sphere" (or normal map) coordinates $\mathbf{u}$ [11409-raycasting-with-style-transfer-and-contour-lines]. This is typically done by mapping the normal vector to the range $[0, 1]$ for the x and y dimensions:
    $$ \mathbf{u} = \begin{bmatrix} n_x \\ -n_y \end{bmatrix} \cdot 0.5 + 0.5 $$
    [11409-raycasting-with-style-transfer-and-contour-lines]
3.  **2D Array Texture Lookup**: The final base color $\mathbf{c}_{base}$ is retrieved from a 2D array texture `styleTransferTexture`, where the first two dimensions correspond to the lit sphere coordinates $\mathbf{u}$ and the third dimension (array layer) corresponds to the style index $s$ [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].
4.  **Exit Point Capture**: To facilitate efficient ray marching, the exit points of rays through the volume are pre-computed using a back-face rendering pass into a 2D texture (`ExitPoints` or `Back Face Texture`) [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].

## Subtopics

### [Raycasting with Style Transfer and Contour Lines](11409-raycasting-with-style-transfer-and-contour-lines.md)
This method performs direct volume ray casting augmented with artistic style transfer and contour line rendering. It differs from its siblings by selectively applying matcap-based contour lines based on local curvature and view angle, using a specific curvature approximation derived from the angle between consecutive normals [11409-raycasting-with-style-transfer-and-contour-lines]. Choose this method when illustrative edge highlighting (contours) is required alongside style transfer.

### [Illustrative Volume Rendering with Style Transfer](11445-illustrative-volume-rendering-with-style-transfer.md)
This method combines direct volume ray casting with a Style Transfer Function (STF) to apply artistic styles. It is distinguished by its two-pass rendering approach: first capturing exit points via back-face rendering, then performing ray casting that utilizes these exit points alongside style transfer textures [11445-illustrative-volume-rendering-with-style-transfer]. Choose this method for general illustrative rendering without specific contour line requirements.

### Illustrative Transparency
A transparency model combining uniform opacity ($\alpha_u$) from the transfer function and directional opacity ($\alpha_d$) from the style map, modulated by curvature and gradient magnitude to simulate the 100-percent-rule [S1].

- If $\alpha_u = 1$, the sample opacity is solely determined by $\alpha_d$ [S1].
- If $\alpha_u = 0$, the sample is completely transparent [S1].
- Compute the term $\max(0, |n \cdot v| - \sqrt{T \kappa_v (2 - T \kappa_v)})$, which is zero on the contour and increases away from it [S1].
- Weight the result by $\alpha_d$ to apply the final opacity [S1].

The overall opacity $\alpha$ is calculated as [S1]:
$$ \alpha = \alpha_d \cdot \alpha_u \cdot \frac{0.5 + \max(0, |n \cdot v| - \sqrt{T \kappa_v (2 - T \kappa_v)}) \cdot (1 - |g|)}{1} $$
where $g$ is the gradient magnitude, $n$ is normal, $v$ is view vector, and $T \kappa_v (2 - T \kappa_v)$ is the contour threshold term [S1].

Parameters:
- $\alpha_u$: Uniform opacity value from the transfer function, controlling overall opacity independent of normal direction [S1].
- $\alpha_d$: Directional opacity tuple (r,g,b,$\alpha$) from the lit sphere map, varying opacity based on normal direction [S1].
- 0.5: Constant restricting maximum opacity enhancement, empirically determined [S1].

## Background
The concept of Style Transfer Functions for Illustrative Volume Rendering was introduced by Stefan Bruckner and Eduard Gröller in 2007 [S1]. The work was published in the journal *Computer Graphics Forum* [S1]. The method was implemented in C++ and OpenGL/GLSL, leveraging Shader Model 3.0 features such as conditional loops and dynamic branching [S1]. It allows users to load volume data, save/load transfer functions with assigned styles, and interactively rotate or zoom around the data [11445-illustrative-volume-rendering-with-style-transfer].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
