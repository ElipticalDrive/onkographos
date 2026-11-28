# Absorption-Only Radiography via Unsorted Hexahedron Projection

Also called: Unsorted Hexahedron Projection, Absorption-Only Radiography

## Overview
This method simulates X-ray radiographs for curvilinear hexahedral meshes by accumulating optical thickness additively, thereby eliminating the need for a global back-to-front visibility sort [S1, unofficial]. It differs from its parent [Hardware-Assisted Projection](5625-hardware-assisted-projection.md) by exploiting the commutative property of addition in the absorption-only regime ($J=0$) to allow unsorted rendering, and from [Hardware-Assisted Projected Tetrahedra](1729-hardware-assisted-projected-tetrahedra.md) by supporting direct projection of hexahedral cells to reduce mesh storage and rendering overhead.

## How it works
**Inputs:** A curvilinear hexahedral mesh where each cell has an extinction coefficient $K_v$ and geometric definition. An image plane resolution.
**Outputs:** A radiograph image representing accumulated optical thickness (or intensity) per pixel.

**Data Structures:**
*   **Mesh Cell:** Defined by 8 vertices. Each vertex $v$ has an associated extinction coefficient $K_v$ and a precomputed thickness $t_v$.
*   **P-buffer:** A floating-point off-screen buffer (texture) storing the accumulated attenuation $\tau_{acc}$ for each pixel. Initialized to zero.
*   **Projection Geometry:** For each hexahedron, a set of triangle fans or strips is generated based on the projection of its faces.

**Steps:**
1.  **Preprocessing (Software):**
    *   For each hexahedral cell, compute the thickness $t_v$ for each vertex $v$ using the cell's geometry [S1, unofficial].
    *   Compute the vertex attenuation $a_v = K_v \cdot t_v$ [S1, unofficial].
    *   Determine if the hexahedron can be projected directly. A hexahedron is valid for direct projection if:
        *   It is non-degenerate (no coinciding vertices).
        *   The projection of each of its 6 faces is a convex quadrilateral [S1, unofficial].
    *   If valid, generate triangle fans or strips for the hexahedron using the method of Max [12].
    *   If invalid (degenerate or non-convex face projections), subdivide the hexahedron into 5 tetrahedra [S1, unofficial]. Note: If these tetrahedra share non-planar faces with adjacent non-subdivided cells, gaps or overlaps may occur, introducing small errors [S1, unofficial].

2.  **Rendering Passes (Hardware):**
    *   Initialize the P-buffer texture with zero attenuation.
    *   Group cells into batches of non-overlapping cells to minimize P-buffer to texture copies [S1, unofficial].
    *   For each batch:
        1.  Render the triangles (fans/strips) of the cells in the batch.
        2.  **Fragment Program:** For each fragment covered by a triangle:
            *   Interpolate the vertex attenuations $a_v$ from the triangle's vertices to get the local attenuation $\tau_{cell}$ for the fragment [S1, unofficial].
            *   Read the current accumulated attenuation $\tau_{acc}$ from the P-buffer texture at the fragment's screen position.
            *   Compute the new accumulated attenuation: $\tau_{new} = \tau_{acc} + \tau_{cell}$ [S1, unofficial].
            *   Write $\tau_{new}$ to the P-buffer [S1, unofficial].
        3.  After rendering the batch, copy the P-buffer contents to the input texture for the next batch [S1, unofficial]. This copy is necessary because writes to output buffers are not guaranteed to be in polygon order, and texture caching does not account for on-the-fly texture changes [S1, unofficial].

3.  **Final Compositing (Software or Hardware):**
    *   After all cells are processed, the P-buffer contains the total optical thickness $\tau_{total}$ for each pixel.
    *   Compute the final intensity $I$ for each pixel using the absorption-only recurrence:
        $$ I = I_0 \exp(-\tau_{total}) $$
        where $I_0$ is the incident X-ray intensity [S1, unofficial].
    *   Alternatively, if multiple wavelength bands are used, accumulate them in parallel in the RGBA channels of the P-buffer and compute the final intensity for each band separately [S1, unofficial].

**Formulas:**
*   **Vertex Attenuation:** $a_v = K_v \cdot t_v$ [S1, unofficial].
*   **Accumulation:** $\tau_{new} = \tau_{acc} + \tau_{cell}$ [S1, unofficial].
*   **Final Intensity:** $I = I_0 \exp(-\tau_{total})$ [S1, unofficial].
*   **Recurrence (Conceptual):** $I_V[i+1] = e^{-\tau[i]} I_V[i]$ [S1, unofficial].

**Parameters:**
*   **Floating Point Precision:** 32-bit floating point is required for the P-buffer to maintain accuracy in attenuation computation [S1, unofficial].
*   **Parallelism:** Up to 4 wavelength bands can be accumulated in parallel using the RGBA channels [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Hexahedra:** Must be subdivided into tetrahedra [S1, unofficial].
*   **Non-Convex Face Projections:** Must be subdivided into tetrahedra [S1, unofficial].
*   **Non-Planar Faces:** Subdividing a hexahedron into tetrahedra may create non-planar faces with adjacent non-subdivided cells, leading to gaps or overlaps in viewing ray segments and small errors in total absorption [S1, unofficial].
*   **Texture Copy Overhead:** Copying the P-buffer to a texture after every cell is slow; batching non-overlapping cells is necessary for performance [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of cells, assuming constant-time projection and fragment processing. The number of texture copies depends on the batching strategy.
*   **Memory Complexity:** $O(R)$ where $R$ is the number of pixels in the image, for the P-buffer.
*   **Strengths:** No global sort required; supports curvilinear hexahedral meshes directly; parallel accumulation of multiple wavelength bands.
*   **Limitations:** Limited to absorption-only regime ($J=0$); potential errors from subdivision of degenerate/non-convex cells; texture copy overhead.
*   **When to Use:** Industrial and medical X-ray simulation where emissivity is negligible and hexahedral meshes are used.
*   **When Not to Use:** When emissivity is significant; when tetrahedral meshes are used and hexahedral projection is not applicable; when high precision is required and subdivision errors are unacceptable.

## Background
This method was developed for industrial and medical settings where X-ray self-emission is negligible [S1, unofficial]. It builds on the polyhedron projection method [S1, unofficial] and uses the method of Max [12] for hexahedral projection and Shirley and Tuchman [16] for tetrahedral projection. The technique addresses the need for floating-point accuracy in attenuation computation, which is not supported by standard framebuffer compositing operations [S1, unofficial].

## Sources
[S1] Hardware-Accelerated Simulated Radiography, https://digital.library.unt.edu/ark:/67531/metadc876978/?q=front-to-back compositing, fetched 2026-10-29 (unofficial)
