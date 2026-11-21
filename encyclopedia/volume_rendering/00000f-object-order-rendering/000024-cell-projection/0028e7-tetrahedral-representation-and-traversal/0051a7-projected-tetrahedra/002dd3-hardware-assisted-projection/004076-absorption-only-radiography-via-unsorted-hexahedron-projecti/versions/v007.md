# Absorption-Only Radiography via Unsorted Hexahedron Projection

Also called: Unsorted Hexahedron Projection, Additive Optical Thickness Accumulation

## Overview
This method simulates absorption-only X-ray radiographs for curvilinear hexahedral meshes by accumulating optical thickness additively, thereby eliminating the need for a global back-to-front visibility sort [S1, unofficial]. It differs from its parent [Hardware-Assisted Projection](5625-hardware-assisted-projection.md) by relaxing the sorting requirement through the commutative property of addition in the absorption-only regime, and from [Hardware-Assisted Polyhedron Projection](1453-hardware-assisted-polyhedron-projection.md) by supporting direct projection of hexahedral cells rather than requiring tetrahedral decomposition for all cases.

## How it works
**Inputs:** A curvilinear hexahedral mesh; extinction coefficients $K_\nu$ (possibly wavelength-dependent); camera/view parameters.
**Outputs:** A radiograph image representing integrated attenuation.

**Data Structures:**
*   **Mesh:** Curvilinear hexahedra defined by vertex positions and face connectivity.
*   **P-buffer:** A floating-point off-screen buffer (texture) storing the accumulated optical thickness $\tau_{acc}$ per pixel. Initialized to zero.
*   **Vertex Data:** For each cell, precomputed vertex attenuation values $A_v = K_\nu \cdot l_v$, where $l_v$ is the geometric thickness at vertex $v$.

**Algorithm Steps:**

1.  **Cell Classification and Tessellation:**
    For each hexahedral cell in the mesh:
    *   Project the cell's faces onto the image plane.
    *   Check if the projection of every face is a convex quadrilateral and the hexahedron is non-degenerate (no coinciding vertices).
    *   If valid, define the rasterizable regions using triangle fans or strips as described by Max [12].
    *   If invalid (degenerate or non-convex face projections), subdivide the hexahedron into tetrahedra. Use the method of Shirley and Tuchman [16] to define triangle fans for the tetrahedra's projections. [G] Note: Subdividing into five tetrahedra is a common strategy for hexahedra [S1, unofficial].

2.  **Thickness and Attenuation Computation (Software):**
    For each vertex $v$ of the current cell (or subdivided tetrahedron):
    *   Compute the geometric thickness $l_v$ along the viewing ray passing through $v$. This is the length of the intersection segment of the ray with the cell.
    *   Compute the vertex attenuation $A_v = K_\nu \cdot l_v$. If $K_\nu$ varies spatially within the cell, $K_\nu$ should be the value at vertex $v$ or interpolated appropriately.

3.  **Rasterization and Accumulation (Hardware):**
    Render the tessellated regions (triangles) of the current cell:
    *   Interpolate the vertex attenuation $A_v$ across the triangle fragments using standard rasterization.
    *   In the fragment shader, read the current accumulated attenuation $\tau_{current}$ from the P-buffer texture at the fragment's screen position.
    *   Compute the new accumulated attenuation: $\tau_{new} = \tau_{current} + A_{interpolated}$.
    *   Write $\tau_{new}$ back to the P-buffer.

4.  **Texture Synchronization:**
    Because writes to output buffers are not guaranteed to be in polygon order and texture caches do not account for on-the-fly changes, the P-buffer must be copied to a texture resource before it can be reliably read in subsequent passes or cells. [S1, unofficial]

5.  **Final Image Generation:**
    After all cells are processed, the P-buffer contains the total optical thickness $\tau_{total}$ for each pixel. The final intensity $I$ is computed from the incident source intensity $I_0$ using the Beer-Lambert law:
    $$ I = I_0 \cdot e^{-\tau_{total}} $$
    This step may be performed in a final post-processing pass or via a lookup texture if $I_0$ is constant.

**Parallelism:**
The method can accumulate attenuation for four different X-ray wavelength bands in parallel by using the Red, Green, Blue, and Alpha channels of the floating-point P-buffer [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Cells:** Hexahedra with coinciding vertices or non-convex face projections must be subdivided. If these subdivided tetrahedra share non-planar faces with adjacent non-subdivided cells, gaps or overlaps in viewing ray segments may occur, introducing small errors in total absorption [S1, unofficial].
*   **Precision:** Standard framebuffer compositing lacks the required floating-point accuracy. A dedicated floating-point P-buffer and fragment program are necessary [S1, unofficial].
*   **Performance:** Copying the P-buffer to a texture is slow. Rendering non-overlapping cells in multiple passes can minimize the number of copies [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of cells and their projected screen area. The cost is dominated by rasterization and memory bandwidth for the P-buffer updates.
*   **Memory Complexity:** Linear with respect to the image resolution (for the P-buffer) and mesh size (for vertex data).
*   **Strengths:** Eliminates the expensive global back-to-front sort required by standard volume rendering. Supports direct hexahedral projection, reducing mesh storage and preprocessing time compared to tetrahedral-only methods.
*   **Limitations:** Only applicable to absorption-only scenarios ($J_\nu = 0$). Precision is limited by the floating-point format of the P-buffer. Subdivision of degenerate cells can introduce geometric artifacts. Texture synchronization overhead can be significant.
*   **When to use:** Industrial and medical X-ray simulation where emissivity is negligible and performance is critical.
*   **When not to use:** Scenes with significant self-emission or scattering, or when high-precision sorting is required for other rendering effects.

## Background
This technique was developed for industrial and medical settings where X-ray self-emission is negligible [S1, unofficial]. It builds on the polyhedron projection method [S1, unofficial] and utilizes tessellation strategies by Shirley and Tuchman [16] and Max [12]. The approach addresses the computational bottleneck of sorting in volume rendering by leveraging the commutative property of addition in the absorption-only regime.

## Sources
[S1] Hardware-Accelerated Simulated Radiography, https://digital.library.unt.edu/ark:/67531/metadc876978/?q=front-to-back compositing, fetched 2026-10-29 (unofficial)
