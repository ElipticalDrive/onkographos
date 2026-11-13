# Style Transfer and Illustrative Rendering

Also called: Style Transfer Functions (STF), Illustrative Volume Rendering

## Overview
Style Transfer and Illustrative Rendering replaces standard color/opacity mapping with a **Style Transfer Function (STF)** that decouples material identification from visual appearance, allowing artistic styles to be applied based on geometry and view angle [S1, 11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer]. It distinguishes itself from standard transfer function methods by mapping volume density to a material index, then to a style index, and finally retrieving base colors from a 2D array texture based on surface normals (lit-sphere mapping), enabling interactive rotation, zoom, and style assignment [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].

## Shared principles
All methods in this family share the following core mechanism, building on [Transfer Function and Opacity Mapping](2789-transfer-function-and-opacity-mapping.md):

1.  **Material Indexing**: Instead of mapping volume density directly to color, a 1D texture (`transferFunctionTexture`) maps density to a **material index** (often stored in the red channel) and an opacity value [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].
2.  **Style Indexing**: A second 1D texture (`indexFunctionTexture`) maps the material index to a **style index** [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].
3.  **Lit-Sphere Mapping**: Surface normals $\mathbf{n}$ are computed via gradients and transformed to view space. They are projected onto a 2D "lit sphere" coordinate system $\mathbf{u}$:
    $$ \mathbf{u} = \begin{bmatrix} n_x \\ -n_y \end{bmatrix} \cdot 0.5 + 0.5 $$
    [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer]
4.  **2D Array Texture Lookup**: A 2D array texture (`styleTransferTexture`) stores artistic styles. The first two dimensions correspond to the lit-sphere coordinates $\mathbf{u}$, and the third dimension (layer) selects the style index $s$ [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer]. The base color $\mathbf{c}_{base}$ is retrieved as:
    $$ \mathbf{c}_{base} = \text{texture}(\text{styleTransferTexture}, \mathbf{u}, s) $$
    [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer]
5.  **Compositing**: The retrieved base color and opacity are composited using the standard front-to-back equation defined in the parent article [S2789].

## Subtopics

### [Raycasting with Style Transfer and Contour Lines](11409-raycasting-with-style-transfer-and-contour-lines.md)
This method extends direct volume ray casting by replacing standard color/opacity mapping with an STF that maps material indices and surface normals to artistic styles via a 2D array texture [11409-raycasting-with-style-transfer-and-contour-lines]. It distinguishes itself from standard transfer function methods by decoupling material identification from visual appearance, allowing multiple artistic styles to be applied to the same underlying data based on geometry and view angle [11409-raycasting-with-style-transfer-and-contour-lines]. It builds on [Style Transfer and Illustrative Rendering](13031-style-transfer-and-illustrative-rendering.md).

Key features include:
*   **Two-Pass Rendering**: A pre-pass captures exit points into a 2D texture (`ExitPoints`) via back-face rendering, which determines the ray length for the main pass [11409-raycasting-with-style-transfer-and-contour-lines].
*   **Contour Lines**: Optional contour lines are generated using a curvature approximation $\kappa$ based on the change in normal direction between samples [11409-raycasting-with-style-transfer-and-contour-lines].
*   **Noise Jittering**: Optional noise jittering can be applied to the ray start position to reduce banding artifacts [11409-raycasting-with-style-transfer-and-contour-lines].

### [Illustrative Volume Rendering with Style Transfer](11445-illustrative-volume-rendering-with-style-transfer.md)
Illustrative Volume Rendering with Style Transfer applies artistic styles to volumetric data by replacing standard color/opacity lookups with an STF [11445-illustrative-volume-rendering-with-style-transfer]. It maps volume density to a material index, then to a style index, and finally retrieves base colors from a 2D array texture based on surface normals, enabling interactive rotation, zoom, and style assignment [11445-illustrative-volume-rendering-with-style-transfer]. It differs from standard ray casting by decoupling material identification from artistic appearance via a two-stage indexing process and lit-sphere mapping [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].

Key features include:
*   **Back Face Texture**: Uses a 2D texture (`backFaceTexture`) to store exit points, facilitating efficient ray marching [11445-illustrative-volume-rendering-with-style-transfer].
*   **Initialization**: Requires explicit initialization of transfer function textures and style arrays [11445-illustrative-volume-rendering-with-style-transfer].
*   **Precision**: Recommends `GL_RGBA32F` for back face textures to ensure accurate exit point storage [11445-illustrative-volume-rendering-with-style-transfer].

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
Origin: Bruckner and Gröller introduced Style Transfer Functions for Illustrative Volume Rendering in 2007 [S1]. The method was implemented in C++ and OpenGL/GLSL, leveraging Shader Model 3.0 features such as conditional loops and dynamic branching [S1]. It builds upon the concept of lit-sphere mapping used in illustrative rendering to apply artistic styles based on surface orientation [S1].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
