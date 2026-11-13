# Röttger 3D Texture Mapping

Also called: Hardware-Assisted Projected Tetrahedra with 3D Texture Mapping, Pre-integrated Volume Rendering for Tetrahedra, Röttger et al. 3D Texture Mapping

## Overview
Röttger 3D Texture Mapping is an extension of the projected tetrahedra algorithm that enables hardware-accelerated, accurate rendering of volumetric data using 3D texture mapping [S1, unofficial]. It solves the problem of applying arbitrary, non-linear transfer functions during volume integration by pre-computing the optical depth integration into a texture, allowing the graphics pipeline to perform the compositing via standard 2D texture lookups [S2, unofficial]. Unlike methods restricted to linear transfer functions, this approach supports arbitrary opacity and color variations along the ray [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh representing the volume; a transfer function mapping density/extinction to color and opacity; camera parameters.
*   **Output:** A 2D image with correct alpha blending.

**Data Structures:**
*   **Pre-integration Texture:** A 3D texture map $T(u, v, w)$ where:
    *   $u$: Value at cell entry point (e.g., extinction coefficient $\tau_{in}$ or density).
    *   $v$: Value at cell exit point (e.g., extinction coefficient $\tau_{out}$ or density).
    *   $w$: Normalized distance through the cell (optical depth parameter, $0 \le w \le 1$).
    *   *Note:* The 3D texture is approximated using 2D texture mapping hardware in practice [S2, unofficial].

**Steps:**
1.  **Pre-computation:** For the given transfer function, compute the accumulated color and opacity for all combinations of entry/exit values and distances. Store these in the 3D texture map. The value at $(u, v, w)$ represents the integrated result of a ray segment of length proportional to $w$ starting with property $u$ and ending with property $v$ [S2, unofficial].
2.  **Projection:** Project each tetrahedron onto the screen space. Subdivide the projected polygon into regions where the optical thickness varies linearly [S1, unofficial].
3.  **Vertex Attribute Setup:** For each vertex of the projected tetrahedron (or subdivision region), compute:
    *   The entry and exit property values (e.g., $\tau_{in}, \tau_{out}$) [S2, unofficial].
    *   The geometric thickness $l$ of the ray segment through the cell at that vertex [S1, unofficial].
4.  **Hardware Interpolation:** Pass the entry/exit values and thickness to the rasterizer. The hardware interpolates these values across the screen-space triangle [S2, unofficial].
5.  **Texture Lookup:** Use the interpolated entry/exit values as the first two texture coordinates and the interpolated thickness (normalized) as the third coordinate to sample the pre-integration texture [S2, unofficial].
6.  **Compositing:** The sampled color and opacity are composited using standard alpha blending (Porter-Duff over) in back-to-front order [S1, unofficial][S2, unofficial].

**Formulas:**
The pre-integration texture stores the result of:
$$ C_{out} = \int_0^1 C(\tau(s)) \exp\left(-\int_0^s \tau(t) dt\right) ds $$
$$ \alpha_{out} = 1 - \exp\left(-\int_0^1 \tau(s) ds\right) $$
where $\tau(s)$ is the extinction coefficient varying linearly from $\tau_{in}$ to $\tau_{out}$ along the normalized path $s \in [0,1]$ [S2, unofficial].

**Parameters:**
*   Texture resolution: Not given in the sources.
*   Transfer function: Arbitrary [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Sorting:** Correct compositing requires cells to be sorted in back-to-front order relative to the viewpoint [S1, unofficial][S2, unofficial].
*   **Approximation:** The 3D texture is approximated using 2D texture mapping, which may introduce interpolation artifacts if not carefully managed [S2, unofficial].
*   **Linear Transfer Functions:** Earlier methods (e.g., [153]) could only handle linear transfer functions without artifacts; Röttger's method specifically addresses this limitation [S2, unofficial].

## Subtopics

### Hardware-Assisted Projected Tetrahedra
The base method using 2D textures for constant extinction. Röttger's method extends this to arbitrary transfer functions using 3D textures [S1, unofficial].

### Pre-integrated Volume Rendering
The general concept of pre-computing integration into textures. Röttger applied this to tetrahedral cells [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by sorting and rasterization. Pre-computation is offline.
*   **Memory Complexity:** Requires storage for the 3D pre-integration texture. Size depends on the resolution of the texture map [S2, unofficial].
*   **Strengths:** Supports arbitrary transfer functions; hardware-accelerated; avoids artifacts associated with linear interpolation of transparency [S2, unofficial].
*   **Limitations:** Requires back-to-front sorting; 3D texture memory usage can be high; approximation via 2D texture mapping may have limitations [S2, unofficial].
*   **When to use:** When arbitrary transfer functions are needed and hardware acceleration is desired for tetrahedral meshes [S2, unofficial].
*   **When not to use:** When sorting is prohibitively expensive or memory is constrained [S2, unofficial].

## Background
The method was introduced by Röttger et al. [RKE00] as an extension of the projected tetrahedra algorithm [ST90] [S1, unofficial]. It builds on the concept of pre-integrated volume rendering to allow for arbitrary transfer functions, addressing the limitations of earlier hardware-assisted methods that were restricted to linear transfer functions [S2, unofficial].

## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
