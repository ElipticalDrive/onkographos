# Textured Splat Rendering

Also called: fragment-level texture coordinate computation, oriented quad splatting

## Overview
Textured Splat Rendering is a point cloud visualization technique that renders each point as an oriented quad, computing per-fragment texture coordinates in the fragment shader to retrieve high-frequency color and normal data [S1]. It solves the detail loss inherent in standard scaled splats by decoupling geometry storage from color storage, allowing for aggressive pruning of the point cloud while maintaining visual fidelity [S1]. Unlike flat splats where properties are constant across the primitive, this method recovers high-frequency detail with fewer samples and larger radii [S1].

## How it works
**Inputs:** A set of 3D points $P = \{\mathbf{p}_k\}$ with associated color data stored in 2D texture maps.
**Outputs:** A 2D image with alpha blending.

**Data Structures:**
1.  **Geometry:** Regular $(x, y, z)$ floating-point coordinates for each point. Storing geometry in memory (rather than encoding depth in 16-bit image formats) avoids precision loss [S1].
2.  **Color/Normal Data:** 2D texture maps containing color and optionally normal vectors. The alpha channel of the color texture is used to discard fragments outside the model's silhouette [S1].

**Algorithm Steps:**
1.  **Preprocessing:** Store color data in texture maps. This allows the use of lossy compression formats to reduce disk memory footprint, as color artifacts are generally tolerable [S1]. Geometry is kept as regular $(x,y,z)$ data [S1].
2.  **Rasterization Setup:** For each point $\mathbf{p}_k$ in the cloud, render an oriented quad. The quad is transformed to screen space [S1].
3.  **Fragment Shader Execution:**
    *   The fragment shader receives the interpolated screen position of the fragment and the scan location $\mathbf{s}_k$ of the point center [S1].
    *   Compute the vector from the scan location to the fragment's position: $\bar{\mathbf{x}} = (x, y)$ and $\bar{y}$ (vertical component) [S1].
    *   Compute polar coordinates relative to the scan location:
        $$ \theta = \arctan(\bar{y}, \bar{x}) $$
        $$ \psi = \text{vertical angle component} $$
        *(Note: The source specifies the domain $-\pi/4 < \psi \le \pi/4$ for the calculation of $\theta$ and texture coordinates)* [S1].
    *   Compute per-fragment texture coordinates $(s, t)$ using the inverse mapping equations [S1]:
        $$ s = \frac{\theta / (2\pi) + 0.5}{1.5} $$
        $$ t = \frac{\psi + \pi/4}{2\pi} $$
    *   Retrieve fragment properties (color, normals) from the texture map using $(s, t)$ [S1].
    *   Use the alpha channel of the retrieved color to discard fragments outside the silhouette [S1].
    *   Generate circular or ellipsoidal splats based on the distance to the point center [S1].
4.  **Compositing:** Accumulate fragments using standard alpha blending [S1].

**Parameters:**
*   **Geometry Storage:** Regular $(x,y,z)$ data [S1].
*   **Color Storage:** Texture maps (potentially lossy compressed) [S1].

**Edge Cases and Pitfalls:**
*   **Precision Loss:** Encoding depth in 16-bit image formats (like PNG) causes precision loss. Storing geometry as regular $(x,y,z)$ data avoids this [S1].
*   **Detail Loss in Scaled Splats:** Using regular scaled splats yields poor results because properties (color, normal) are constant across the splat surface [S1].

## Subtopics

### Hardware-Accelerated Textured Polygon Splatting
A variant where pre-computed splat footprints are stored in texture map memory and rendered as 2D textured polygons to leverage hardware acceleration [S2]. It uses mipmaps to handle different resolution levels in hierarchical representations, allowing automatic scaling by drawing larger textured polygons [S2]. A limitation is the dependency on high-end graphics workstations with 3D texture support for potential extensions to 3D hypervolume splats [S2].

## Operating characteristics
*   **Performance:** Achieves almost interactive rates for fairly complex datasets by leveraging hardware acceleration of modern graphics workstations [S2].
*   **Memory:** Reduces disk memory footprint significantly by storing color in compressed texture maps, while keeping geometry in memory [S1].
*   **Quality:** Recovers high-frequency color and normal detail rendering a smaller number of samples with larger radii compared to flat splats [S1].
*   **Limitations:** Requires high-end graphics workstations for 3D texture support if extending to hypervolume splats [S2].

## Background
The method was proposed to improve hierarchical rendering approaches for massive point clouds, enabling more aggressive pruning [S1]. It builds on the principles of Texture Splatting [19892-texture-splatting.md] but introduces fragment-level texture coordinate computation to avoid the constant-property limitation of flat splats [S1]. The approach is orthogonal to hierarchical arrangements of point samples because texture coordinates are computed at runtime from point coordinates [S1].

## Sources
[S1] View-dependent Hierarchical Rendering of Massive Point Clouds through Textured Splats, https://doi.org/10.2312/ceig.20191203, fetched 2026-10-26
[S2] Hypervolume visualization: a challenge in simplicity, https://doi.org/10.1145/288126.288173, fetched 2026-10-26
