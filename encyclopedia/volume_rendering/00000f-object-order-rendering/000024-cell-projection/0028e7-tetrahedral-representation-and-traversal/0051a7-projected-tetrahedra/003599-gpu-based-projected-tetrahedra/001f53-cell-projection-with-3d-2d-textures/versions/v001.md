# Cell Projection with 3D/2D Textures

Also called: Röttger et al. Cell Projection with 3D/2D Textures

## Overview
This method extends GPU-based Projected Tetrahedra by pre-computing volume rendering integrals into texture maps, enabling hardware-accelerated compositing of semi-transparent triangles. It solves the limitation of standard projected tetrahedra by supporting arbitrary transfer functions via pre-integration, at the cost of increased memory usage for 3D textures or approximation errors for 2D textures. It differs from [Sorted Rasterization of Projected Tetrahedra](3018-sorted-rasterization-of-projected-tetrahedra.md) by relying on texture-based integral approximation rather than explicit sorting of all triangles, and from [On-the-fly Shell Geometry Construction](13927-on-the-fly-shell-geometry-construction.md) by using pre-computed textures instead of dynamic geometry generation.

## How it works
**Inputs:** A volume dataset, a transfer function, and a viewing direction.
**Outputs:** A rendered image.

**Data Structures:**
1.  **3D Texture Map (Optional/Original):** Stores pre-integrated color and opacity. Dimensions correspond to cell entry values, exit values, and distance through the cell.
2.  **2D Texture Map 1 (Approximation):** Stores dependencies of the volume rendering integral with respect to the length between two sample points, approximated in linear terms.
3.  **2D Texture Map 2 (Residual):** Stores remaining parts of the integral that depend only on sample values.
4.  **Projected Tetrahedra Geometry:** Each tetrahedron is decomposed into triangles. Vertices are assigned all associated triangles.

**Steps:**
1.  **Pre-processing:** Compute the volume rendering integral for each cell.
    *   If using 3D textures: Store color and opacity directly in a 3D texture map [S1, unofficial].
    *   If using 2D textures: Approximate dependencies on sample point distance linearly and store in the first 2D texture map. Store remaining sample-value-dependent integral parts in the second 2D texture map [S1, unofficial].
2.  **Decomposition:** Decompose each volume cell into tetrahedra, then into triangles.
3.  **Projection:** Project the tetrahedra onto the screen.
4.  **Triangle Assignment:** Assign all associated triangles to each vertex of each projected tetrahedron [S1, unofficial].
5.  **Rendering:**
    *   Compute opacity and color at the thickest point for each tetrahedron.
    *   Perform linear interpolation between the triangle faces pierced by the ray [S1, unofficial].
    *   Blend the triangles into the frame-buffer using back-to-front compositing (Painter's Algorithm) [S1, unofficial].

**Parameters:**
*   Texture resolution: Not given in the sources.
*   Transfer function: Arbitrary (if 3D texture) or Linear (if 2D texture approximation is used without pre-integration extensions) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Usage:** 3D textures are memory-consuming [S1, unofficial].
*   **Transfer Function Limitations:** The 2D texture approximation method can only handle linear transfer functions without artifacts [S2, unofficial].
*   **Approximation Error:** Linear approximation of integral dependencies may introduce errors for non-linear volume properties.

## Subtopics

### Cell Sorting for Projection
Sorts the cells before they are split into tetrahedra to ensure correct depth ordering. Utilizes 2D texture mapping hardware to accelerate opacity interpolation and provide correct per-pixel opacity values. This method can only handle linear transfer functions without artifacts [S2, unofficial].

## Operating characteristics
**Time Complexity:** Pre-processing is required to compute texture maps. Rendering time is reduced to texture lookups and compositing.
**Memory Complexity:** High for 3D texture approach; moderate for 2D texture approach.
**Measured Performance:** Interactive rendering times were achieved on an SGI Octane MXE with up to 5 fps [S1, unofficial].
**Strengths:** Supports arbitrary transfer functions (with 3D textures); leverages hardware texture interpolation.
**Limitations:** High memory consumption for 3D textures; 2D approximation limited to linear transfer functions; requires pre-processing.
**When to use:** When arbitrary transfer functions are needed and memory is sufficient, or when linear transfer functions are acceptable and memory is constrained.
**When not to use:** When memory is extremely limited and linear transfer functions are insufficient, or when dynamic transfer functions are required without pre-computation.

## Background
Originated with Röttger et al. [Röttger00], who introduced pre-integrated volume rendering for projected tetrahedra. The method was extended by Guthe et al. [Guthe02] to add hardware-accelerated pre-integration [Engel01]. It builds on earlier parallel cell projection work by Lucas [Lucas92] and Stein et al. [243].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
