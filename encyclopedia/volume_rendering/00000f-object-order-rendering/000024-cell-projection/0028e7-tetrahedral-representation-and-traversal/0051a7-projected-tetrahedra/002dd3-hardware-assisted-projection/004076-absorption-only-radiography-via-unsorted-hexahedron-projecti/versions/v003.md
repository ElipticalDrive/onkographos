# Absorption-Only Radiography via Unsorted Hexahedron Projection

Also called: Unsorted Hexahedron Projection, Absorption-Only Radiography

## Overview
A hardware-accelerated technique to simulate radiographs in the absorption-only regime (emissivity $J=0$) for curvilinear hexahedral meshes [S1, unofficial]. It avoids the need for a global back-to-front visibility sort by accumulating optical thickness additively, leveraging the commutativity of addition [S1, unofficial]. Unlike [Hardware-Assisted Projected Tetrahedra](1729-hardware-assisted-projected-tetrahedra.md), which requires tetrahedral decomposition, this method projects hexahedral cells directly when possible, reducing mesh storage and rendering time [S1, unofficial].

## How it works
**Inputs:**
- A curvilinear hexahedral mesh (or tetrahedral mesh for degenerate cases).
- Extinction coefficients $K_\nu$ per vertex or cell.
- View-projection matrices.

**Outputs:**
- A 2D image representing the accumulated optical thickness (attenuation) per pixel.
- Final intensity derived from the accumulated attenuation.

**Data Structures:**
- **P-buffer:** A floating-point off-screen buffer (texture) storing the accumulated attenuation sum. Initialized to zero.
- **Vertex Attributes:** For each vertex, store the attenuation value $A_v = K_\nu \times \text{thickness}$, where thickness is computed in software [S1, unofficial].
- **Cell Geometry:** Hexahedral connectivity. If a hexahedron is degenerate or has non-convex face projections, it is subdivided into 5 tetrahedra [S1, unofficial].

**Steps:**
1.  **Preprocessing (Subdivision):**
    - For each hexahedral cell, test if it can be projected as a whole. A cell is valid for direct projection if:
        - It is non-degenerate (no coinciding vertices).
        - The projection of each of its 6 faces is a convex quadrilateral [S1, unofficial].
    - If valid, mark it for hexahedral projection.
    - If invalid, subdivide the hexahedron into 5 tetrahedra using a consistent method (e.g., [Max et al.](13) or similar) [S1, unofficial].
    - Compute the thickness for each vertex of the resulting primitives in software using cell geometry. Calculate vertex attenuation $A_v = K_\nu \times \text{thickness}$ [S1, unofficial].

2.  **Rendering Passes (Minimizing Copies):**
    - Because copying the P-buffer to a texture is slow, render multiple non-overlapping cells in a single pass before copying [S1, unofficial].
    - Form a logically circular list of pointers to all cells.
    - Iterate through the list, selecting cells that do not overlap with previously rendered cells in the current pass (or use a heuristic to maximize non-overlapping batches) [S1, unofficial].
    - For each selected cell:
        - **Projection:**
            - If hexahedral: Use the method of [Max](12) to define triangle fans or strips for the projection, assuming convex face projections [S1, unofficial].
            - If tetrahedral: Use the method of [Shirley and Tuchman](16) to divide the image plane projection into triangle fans [S1, unofficial].
        - **Rasterization:**
            - Render the primitives (triangles) to the P-buffer.
            - In the fragment program, interpolate the attenuation $A$ from the vertex data [S1, unofficial].
            - Read the current accumulated attenuation $I_{acc}$ from the image-aligned texture (P-buffer) [S1, unofficial].
            - Compute the new sum: $I_{new} = I_{acc} + A$ [S1, unofficial].
            - Write $I_{new}$ back to the P-buffer [S1, unofficial].

3.  **Buffer Management:**
    - After rendering a batch of cells, copy the P-buffer to a texture to keep it current for the next iteration. This is necessary because writes to output buffers are not guaranteed to be in polygon order, and texture caching does not account for textures changing on the fly [S1, unofficial].

4.  **Final Compositing:**
    - After all cells are processed, the P-buffer contains the total optical thickness $\tau$ for each pixel.
    - The final intensity is computed from the accumulated attenuation. For absorption-only radiography ($J_\nu = 0$), the recurrence is:
      $$I_V[i+1] = e^{-\tau[i]} I_V[i]$$
      where $I_V[i]$ is the intensity at cell $i$ and $\tau[i]$ is the optical thickness [S1, unofficial].
    - Since addition is commutative, the global product of exponentials is equivalent to a single exponential of the sum:
      $$I_{final} = I_{source} \cdot e^{-\sum \tau}$$
      where $\sum \tau$ is the value in the P-buffer [S1, unofficial].

**Formulas:**
- Vertex Attenuation: $A_v = K_\nu \times \text{thickness}$ [S1, unofficial].
- Accumulation: $I_{acc}^{new} = I_{acc}^{old} + A$ [S1, unofficial].
- Final Intensity: $I_{final} = I_{source} \cdot e^{-I_{acc}}$ [S1, unofficial].

**Parameters:**
- **Parallelism:** Four different X-ray wavelength bands can be accumulated in parallel using the red, green, blue, and alpha channels of the buffer [S1, unofficial].
- **Precision:** 32-bit floating point accuracy is required for attenuation computation [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Degenerate Cells:** Hexahedral cells with coinciding vertices must be subdivided into tetrahedra [S1, unofficial].
- **Non-Convex Projections:** If the projection of a hexahedral face is non-convex, the cell must be subdivided [S1, unofficial].
- **Gaps/Overlaps:** If subdivided tetrahedra share non-planar faces with adjacent non-subdivided cells, gaps or overlaps in viewing ray segments may occur, introducing small errors in total absorption [S1, unofficial].
- **Performance:** Copying the P-buffer to a texture is slow. Rendering as many non-overlapping cells as possible per pass minimizes copying [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of cells, but dependent on the number of rendering passes required to minimize buffer copies. Subdivision increases the number of primitives.
- **Memory Complexity:** Linear with respect to the number of cells for the mesh data. The P-buffer size is fixed by the image resolution.
- **Strengths:** No global sort required. Direct projection of hexahedra saves storage and rendering time compared to tetrahedral decomposition. Parallel accumulation of multiple wavelength bands.
- **Limitations:** Requires floating-point P-buffer support. Degenerate or non-convex cells require subdivision, which can introduce small errors. Buffer copying overhead if many overlapping cells exist.
- **When to use:** Absorption-only radiography (X-ray simulation) where emissivity is negligible. Suitable for curvilinear hexahedral meshes.
- **When not to use:** Emissive volumes (requires front-to-back compositing). Scenes with significant self-emission.

## Background
Originated in the context of industrial and medical X-ray simulation [S1, unofficial]. The method builds on the polyhedron projection approach [S1, unofficial]. It utilizes triangle fan/strip definitions for hexahedra from [Max](12) and tetrahedra from [Shirley and Tuchman](16) [S1, unofficial]. The need for floating-point accuracy in attenuation computation led to the use of fragment programs and P-buffers, as standard framebuffer compositing operations lacked sufficient precision at the time [S1, unofficial].

## Sources
[S1] Hardware-Accelerated Simulated Radiography, https://digital.library.unt.edu/ark:/67531/metadc876978/?q=front-to-back compositing, fetched 2026-10-29 (unofficial)
