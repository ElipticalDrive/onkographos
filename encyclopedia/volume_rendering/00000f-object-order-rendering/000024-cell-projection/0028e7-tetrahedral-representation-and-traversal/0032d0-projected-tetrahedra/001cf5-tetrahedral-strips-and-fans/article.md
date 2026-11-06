# Tetrahedral Strips and Fans

Also called: Bandwidth reduction for tetrahedral meshes, Rendering Pattern and Stripification, Tetrahedral Fans, Tetrahedral Strips, Tetrahedral primitives

## Overview
Tetrahedral Strips and Fans are specialized rendering primitives designed to reduce the bandwidth bottleneck between main memory and the graphics processor when rendering unstructured tetrahedral meshes [S2, unofficial]. They address the inefficiency of rendering projected tetrahedra as individual triangles or fans by grouping tetrahedra into sequences that share vertices, similar to how triangle strips and fans optimize triangle meshes [S1, unofficial]. Unlike triangle strips, which can be computed statically, tetrahedral decomposition is view-dependent, requiring dynamic generation of these primitives for each frame [S2, unofficial].

## How it works
The method defines two primitives: the **Tetrahedral Fan** and the **Tetrahedral Strip** (sequential and generalized) [S2, unofficial]. These primitives allow the graphics hardware to reconstruct tetrahedra from a compact vertex stream, reducing the number of vertices transmitted per tetrahedron to approximately 1.2 [S2, unofficial].

### Inputs and Outputs
*   **Input:** A tetrahedral mesh (unstructured volume data) and a view direction [S2, unofficial].
*   **Output:** A stream of vertices and control flags sent to the graphics pipeline, which reconstructs the tetrahedra for projection and compositing [S2, unofficial].

### Data Structures
The primitives operate on a vertex stream. The layout depends on the primitive type [S2, unofficial]:
1.  **Tetrahedral Fan:**
    *   First 2 vertices: Define the shared edge [S2, unofficial].
    *   Next 2 vertices: Complete the first tetrahedron [S2, unofficial].
    *   Subsequent vertices ($k \ge 5$): Each new vertex $k$ forms a new tetrahedron with the shared edge and the previous vertex [S2, unofficial].
2.  **Sequential Tetrahedral Strip:**
    *   First 4 vertices: Define the first tetrahedron [S2, unofficial].
    *   Subsequent vertices: Each new vertex replaces the least-recently-used (LRU) vertex of the previous tetrahedron [S2, unofficial].
3.  **Generalized Tetrahedral Strip:**
    *   First 4 vertices: Define the first tetrahedron [S2, unofficial].
    *   Flags: For each subsequent vertex, a 2-bit flag indicates which of the four vertex slots (1, 2, 3, or 4) is replaced by the new vertex [S2, unofficial].

### Numbered Steps

#### 1. Tetrahedral Fan Generation
A fan is a sequence of tetrahedra sharing a common edge [S2, unofficial].
1.  Identify a common edge shared by a sequence of adjacent tetrahedra [S2, unofficial].
2.  Transmit the two vertices of the shared edge ($v_1, v_2$) [S2, unofficial].
3.  Transmit the third and fourth vertices ($v_3, v_4$) to complete the first tetrahedron ($v_1, v_2, v_3, v_4$) [S2, unofficial].
4.  For each subsequent tetrahedron in the fan, transmit one new vertex $v_k$ (where $k \ge 5$) [S2, unofficial].
5.  The hardware constructs the $m$-th tetrahedron using vertices $v_1, v_2, v_{m+2}, v_{m+3}$. For example, the second tetrahedron uses $v_1, v_2, v_4, v_5$, and the third uses $v_1, v_2, v_5, v_6$ [S2, unofficial].

#### 2. Sequential Tetrahedral Strip Generation
A sequential strip connects tetrahedra by shared faces, replacing the LRU vertex [S2, unofficial].
1.  Transmit the four vertices of the first tetrahedron: $v_1, v_2, v_3, v_4$ [S2, unofficial].
2.  For each subsequent tetrahedron, transmit one new vertex $v_k$ ($k \ge 5$) [S2, unofficial].
3.  The new tetrahedron is formed by replacing the LRU vertex of the previous tetrahedron with $v_k$, effectively shifting the vertex window by one position [S2, unofficial]. For example, transmitting $v_5$ forms tetrahedron $v_2, v_3, v_4, v_5$ [S2, unofficial].

#### 3. Generalized Tetrahedral Strip Generation
A generalized strip allows replacing any vertex, not just the LRU one [S2, unofficial].
1.  Transmit the four vertices of the first tetrahedron: $v_1, v_2, v_3, v_4$ [S2, unofficial].
2.  For each subsequent vertex $v_k$:
    *   Transmit a 2-bit flag indicating which vertex slot (1, 2, 3, or 4) is replaced [S2, unofficial].
    *   Transmit the new vertex $v_k$ [S2, unofficial].
    *   The new tetrahedron is formed by replacing the specified slot in the previous tetrahedron's vertex set with $v_k$ [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_tetrahedral_fan(shared_edge, fan_vertices):
    """
    shared_edge: list of 2 vertices [v1, v2]
    fan_vertices: list of vertices [v3, v4, v5, ...]
    Yields vertex lists for each tetrahedron.
    """
    v1, v2 = shared_edge
    # First tetrahedron uses v1, v2 and the first two fan vertices
    yield [v1, v2, fan_vertices[0], fan_vertices[1]]
    
    # Subsequent tetrahedra use v1, v2 and the last two fan vertices
    for i in range(2, len(fan_vertices)):
        yield [v1, v2, fan_vertices[i-1], fan_vertices[i]]

def generate_sequential_tetrahedral_strip(vertices):
    """
    vertices: list of vertices [v1, v2, v3, v4, v5, ...]
    Yields vertex lists for each tetrahedron.
    """
    if len(vertices) < 4:
        return
    # First tetrahedron
    yield vertices[0:4]
    # Subsequent tetrahedra shift the window by one
    for i in range(4, len(vertices)):
        yield vertices[i-3 : i+1]

def generate_generalized_tetrahedral_strip(vertices, flags):
    """
    vertices: list of vertices [v1, v2, v3, v4, v5, ...]
    flags: list of integers [1, 1, 3, ...] indicating which slot (1-4) to replace
    Yields vertex lists for each tetrahedron.
    """
    if len(vertices) < 4:
        return
    current_tet = list(vertices[0:4])
    yield current_tet
    for i in range(4, len(vertices)):
        vk = vertices[i]
        replace_slot = flags[i-4] # 1-based index
        current_tet[replace_slot - 1] = vk
        yield current_tet
```

### Parameters
*   **Transmission Rate:** The primitives achieve transmission rates approaching 1.2 vertices per tetrahedron [S2, unofficial].
*   **Strip Lengths:**
    *   Mean strip lengths of 17 on representative datasets [S1, unofficial].
    *   Method 2 achieves mean strip lengths of 29 tetrahedra per strip [S2, unofficial].
*   **Bandwidth Cost:** Rendering tetrahedra requires 50-60% more bandwidth than rendering individual triangles even with fans [S2, unofficial].
*   **Generalized Strip Overhead:** 2 bits per vertex for the replacement flag [S2, unofficial].

### Edge Cases and Pitfalls
*   **View Dependence:** Static strips cannot be used because tetrahedral decomposition is view-dependent. Strips must be regenerated for each frame or viewpoint change [S2, unofficial].
*   **Degenerate Input:** Strips require valid adjacency information. If the mesh has non-manifold edges or disconnected components, strip generation algorithms may fail or produce short strips [S2, unofficial].

## Subtopics

### [Greedy Strip Generation Algorithm](2626-greedy-strip-generation-algorithm.md)
A method for generating tetrahedral strips that achieves average strip lengths of 17 on representative datasets, optimizing the grouping of tetrahedra to minimize bandwidth.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   The combined effect of the architecture and new primitives is a 72 to 85 times increase in performance over triangle graphics hardware approaches [S1, unofficial].
    *   On the 'phoenix' dataset, Method 2 achieved a mean strip length of 29.4 tetrahedra per strip with 1.10 vertices per tetrahedron [S2, unofficial].
    *   On the 'langley' dataset, Method 2 achieved a mean strip length of 49.0 tetrahedra per strip with 1.06 vertices per tetrahedron [S2, unofficial].
*   **Strengths:**
    *   Significantly reduces bandwidth between main memory and graphics processor [S2, unofficial].
    *   Makes software sorting pass unnecessary by moving computations for order-independent transparency and volume scan conversion to graphics hardware [S1, unofficial].
    *   Tetrahedral stripping is inherently more efficient than triangle stripping, with gains greater than expected [S2, unofficial].
*   **Limitations:**
    *   Requires hardware support for the new primitives (GL_TET_FAN_EXT, GL_GENERAL_TET_STRIP_EXT) or software emulation [S2, unofficial].
*   **When to Use:** Interactive tetrahedral volume rendering where bandwidth is a bottleneck and hardware supports the primitives [S2, unofficial].
*   **When Not to Use:** When static geometry is sufficient (e.g., non-view-dependent rendering) or when hardware does not support the extensions.

## Background
The primitives were introduced by Craig M. Wittenbrink et al. to address the bandwidth bottleneck in interactive tetrahedral volume rendering [S2, unofficial]. They build on the Projected Tetrahedra algorithm by Shirley and Tuchman, which generates triangles that may be rendered by hardware acceleration but suffers from high bandwidth usage [S2, unofficial]. The work was presented in the paper "An Architecture For Interactive Tetrahedral Volume Rendering" [S2, unofficial].

## Sources
[S1] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S2] An Architecture For Interactive Tetrahedral Volume Rendering, https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)
[S3] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
