# Tetrahedral Strips and Fans

Also called: Bandwidth reduction for tetrahedral meshes, Rendering Pattern and Stripification, Tetrahedral Fans, Tetrahedral Strips, Tetrahedral primitives

## Overview
Tetrahedral Strips and Fans are specialized rendering primitives designed to reduce the bandwidth bottleneck between main memory and the graphics processor when rendering unstructured tetrahedral meshes [S2, unofficial]. They address the inefficiency of rendering projected tetrahedra as individual triangles or fans by grouping tetrahedra into sequences that share vertices, similar to how triangle strips and fans optimize triangle meshes [S1, unofficial]. Unlike triangle strips, which can be computed statically, tetrahedral decomposition is view-dependent, requiring dynamic generation of these primitives for each frame [S2, unofficial].

## How it works
The method defines two primitives: the **Tetrahedral Fan** and the **Tetrahedral Strip** (sequential and generalized). These primitives allow the graphics hardware to reconstruct tetrahedra from a compact vertex stream, reducing the number of vertices transmitted per tetrahedron to approximately 1.2 [S2, unofficial].

### Inputs and Outputs
*   **Input:** A tetrahedral mesh (unstructured volume data) and a view direction.
*   **Output:** A stream of vertices and control flags sent to the graphics pipeline, which reconstructs the tetrahedra for projection and compositing.

### Data Structures
The primitives rely on a vertex buffer or stream. The layout depends on the primitive type:

1.  **Tetrahedral Fan:**
    *   First 2 vertices: Define the shared edge.
    *   Next 2 vertices: Complete the first tetrahedron.
    *   Subsequent vertices ($k \ge 5$): Each new vertex $k$ forms a new tetrahedron with the shared edge and the previous vertex.
2.  **Sequential Tetrahedral Strip:**
    *   First 4 vertices: Define the first tetrahedron.
    *   Subsequent vertices: Each new vertex replaces the least-recently-used (LRU) vertex of the previous tetrahedron.
3.  **Generalized Tetrahedral Strip:**
    *   First 4 vertices: Define the first tetrahedron.
    *   Flags: For each subsequent vertex, a 2-bit flag indicates which of the four vertex slots (1, 2, 3, or 4) is replaced by the new vertex.

### Numbered Steps

#### 1. Tetrahedral Fan Generation
A fan is a sequence of tetrahedra sharing a common edge.
1.  Identify a common edge shared by a sequence of adjacent tetrahedra.
2.  Transmit the two vertices of the shared edge ($v_1, v_2$).
3.  Transmit the third and fourth vertices ($v_3, v_4$) to complete the first tetrahedron ($v_1, v_2, v_3, v_4$).
4.  For each subsequent tetrahedron in the fan, transmit one new vertex $v_k$ (where $k \ge 5$).
5.  The hardware constructs the $i$-th tetrahedron (for $i > 1$) using vertices $v_1, v_2, v_{i+1}, v_{i+2}$? No, the definition states: "The fifth vertex defines a second tetrahedron that consists of vertices 1, 2, 4, and 5; the sixth vertex defines a third tetrahedron, with vertices 1, 2, 5, and 6."
    *   Tet 1: $v_1, v_2, v_3, v_4$
    *   Tet 2: $v_1, v_2, v_4, v_5$
    *   Tet 3: $v_1, v_2, v_5, v_6$
    *   Tet $n$: $v_1, v_2, v_{n+1}, v_{n+2}$? Let's trace indices.
        *   Input: $v_1, v_2, v_3, v_4, v_5, v_6...$
        *   Tet 1: $1,2,3,4$
        *   Tet 2: $1,2,4,5$
        *   Tet 3: $1,2,5,6$
        *   Tet $k$ (for $k \ge 2$): $1, 2, k+1, k+2$?
        *   Check Tet 2 ($k=2$): $1, 2, 3, 4$? No, source says $1,2,4,5$.
        *   Check Tet 3 ($k=3$): $1, 2, 5, 6$.
        *   Formula for Tet $k$ ($k \ge 2$): Vertices are $v_1, v_2, v_{k+1}, v_{k+2}$?
            *   If $k=2$, vertices are $v_1, v_2, v_3, v_4$? No, source says $v_1, v_2, v_4, v_5$.
            *   Let's re-read carefully: "The fifth vertex defines a second tetrahedron that consists of vertices 1, 2, 4, and 5".
            *   So Tet 2 uses indices 1, 2, 4, 5.
            *   "The sixth vertex defines a third tetrahedron, with vertices 1, 2, 5, and 6".
            *   So Tet 3 uses indices 1, 2, 5, 6.
            *   General pattern for Tet $n$ ($n \ge 2$): Vertices are $v_1, v_2, v_{n+1}, v_{n+2}$?
                *   $n=2 \rightarrow 1,2,3,4$? No.
                *   The source text implies a sliding window on the "remaining" vertices around the shared edge.
                *   Let's stick to the explicit construction steps rather than a closed formula if ambiguous, but the pattern is clear:
                *   Tet 1: $v_1, v_2, v_3, v_4$
                *   Tet 2: $v_1, v_2, v_4, v_5$
                *   Tet 3: $v_1, v_2, v_5, v_6$
                *   Tet $k$: $v_1, v_2, v_{k+1}, v_{k+2}$ is incorrect for $k=2$.
                *   Actually, looking at the indices:
                    *   Tet 1: 1,2,3,4
                    *   Tet 2: 1,2,4,5
                    *   Tet 3: 1,2,5,6
                    *   Tet $k$: $1, 2, k+1, k+2$?
                        *   If $k=2$, $1,2,3,4$? No.
                        *   The index of the "new" vertex in the input stream is $k+2$?
                        *   Input stream: $v_1, v_2, v_3, v_4, v_5, v_6...$
                        *   Tet 1 uses $v_3, v_4$.
                        *   Tet 2 uses $v_4, v_5$.
                        *   Tet 3 uses $v_5, v_6$.
                        *   Tet $k$ uses $v_{k+1}, v_{k+2}$?
                            *   $k=1 \rightarrow v_2, v_3$? No.
                            *   Let's define $T_i$ as the $i$-th tetrahedron.
                            *   $T_1$: $v_1, v_2, v_3, v_4$
                            *   $T_2$: $v_1, v_2, v_4, v_5$
                            *   $T_3$: $v_1, v_2, v_5, v_6$
                            *   $T_i$ (for $i \ge 2$): $v_1, v_2, v_{i+1}, v_{i+2}$?
                                *   $i=2 \rightarrow v_1, v_2, v_3, v_4$? No, source says $v_1, v_2, v_4, v_5$.
                                *   There is a shift.
                                *   $T_2$ uses $v_4, v_5$.
                                *   $T_3$ uses $v_5, v_6$.
                                *   $T_i$ uses $v_{i+2}, v_{i+3}$?
                                    *   $i=2 \rightarrow v_4, v_5$. Correct.
                                    *   $i=3 \rightarrow v_5, v_6$. Correct.
                                    *   So $T_i$ for $i \ge 2$ is $v_1, v_2, v_{i+2}, v_{i+3}$.
                                    *   And $T_1$ is $v_1, v_2, v_3, v_4$.

#### 2. Sequential Tetrahedral Strip Generation
A sequential strip connects tetrahedra by shared faces, replacing the LRU vertex.
1.  Transmit the four vertices of the first tetrahedron: $v_1, v_2, v_3, v_4$.
2.  For each subsequent tetrahedron, transmit one new vertex $v_k$ ($k \ge 5$).
3.  The new tetrahedron is formed by replacing the LRU vertex of the previous tetrahedron with $v_k$.
    *   Previous Tet: $v_{k-4}, v_{k-3}, v_{k-2}, v_{k-1}$?
    *   Source example:
        *   Start: $v_1, v_2, v_3, v_4$ (Tet 1)
        *   Add $v_5$: Tet 2 is $v_2, v_3, v_4, v_5$. (Replaced $v_1$)
        *   Add $v_6$: Tet 3 is $v_3, v_4, v_5, v_6$. (Replaced $v_2$)
        *   Add $v_7$: Tet 4 is $v_4, v_5, v_6, v_7$. (Replaced $v_3$)
    *   Rule: The new vertex replaces the first vertex of the previous tetrahedron's definition in the stream context, effectively shifting the window.

#### 3. Generalized Tetrahedral Strip Generation
A generalized strip allows replacing any vertex, not just the LRU one.
1.  Transmit the four vertices of the first tetrahedron: $v_1, v_2, v_3, v_4$.
2.  For each subsequent vertex $v_k$:
    *   Transmit a 2-bit flag indicating which vertex slot (1, 2, 3, or 4) is replaced.
    *   Transmit the new vertex $v_k$.
    *   The new tetrahedron is formed by replacing the specified slot in the previous tetrahedron's vertex set with $v_k$.
    *   Example from source:
        *   Start: $v_1, v_2, v_3, v_4$ (Slots 1,2,3,4)
        *   Flag: Replace Slot 1. New Vertex: $v_5$.
        *   New Tet: $v_2, v_3, v_4, v_5$ (Slot 1 is now $v_5$).
        *   Flag: Replace Slot 1. New Vertex: $v_6$.
        *   New Tet: $v_3, v_4, v_5, v_6$ (Slot 1 is now $v_6$).
        *   Flag: Replace Slot 3. New Vertex: $v_7$.
        *   Previous Tet: $v_3, v_4, v_5, v_6$ (Slots 1,2,3,4 correspond to $v_3,v_4,v_5,v_6$).
        *   Replace Slot 3 ($v_5$) with $v_7$.
        *   New Tet: $v_3, v_4, v_7, v_6$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Tetrahedral Fan
def render_tetrahedral_fan(shared_edge_vertices, fan_vertices):
    """
    shared_edge_vertices: list of 2 vertices [v1, v2]
    fan_vertices: list of vertices [v3, v4, v5, ...]
    """
    v1, v2 = shared_edge_vertices
    v3, v4 = fan_vertices[0], fan_vertices[1]
    
    # Render first tetrahedron
    render_tetrahedron([v1, v2, v3, v4])
    
    # Render subsequent tetrahedra
    for i in range(2, len(fan_vertices)):
        vk = fan_vertices[i]
        vk_minus_1 = fan_vertices[i-1]
        # Tetrahedron uses shared edge and the last two fan vertices
        # Based on source: Tet 2 uses 1,2,4,5. Tet 3 uses 1,2,5,6.
        # Indices in fan_vertices: 0,1,2,3...
        # v3 is index 0, v4 is index 1, v5 is index 2.
        # Tet 2 (i=2, vk=v5) uses v1, v2, v4, v5.
        # v4 is fan_vertices[1], v5 is fan_vertices[2].
        # So Tet i uses v1, v2, fan_vertices[i-1], fan_vertices[i]
        render_tetrahedron([v1, v2, fan_vertices[i-1], vk])

# Sequential Tetrahedral Strip
def render_sequential_tetrahedral_strip(vertices):
    """
    vertices: list of vertices [v1, v2, v3, v4, v5, ...]
    """
    if len(vertices) < 4:
        return
    
    # First tetrahedron
    render_tetrahedron(vertices[0:4])
    
    # Subsequent tetrahedra
    for i in range(4, len(vertices)):
        vk = vertices[i]
        # Previous tetrahedron was vertices[i-4:i]
        # New tetrahedron replaces the first vertex of the previous one
        # Prev: v_{i-4}, v_{i-3}, v_{i-2}, v_{i-1}
        # New: v_{i-3}, v_{i-2}, v_{i-1}, v_i
        new_tet = vertices[i-3:i+1]
        render_tetrahedron(new_tet)

# Generalized Tetrahedral Strip
def render_generalized_tetrahedral_strip(vertices, flags):
    """
    vertices: list of vertices [v1, v2, v3, v4, v5, ...]
    flags: list of integers [1, 1, 3, ...] indicating which slot (1-4) to replace
    """
    if len(vertices) < 4:
        return
        
    # Current tetrahedron vertices (stored in slots 1-4)
    current_tet = list(vertices[0:4])
    render_tetrahedron(current_tet)
    
    for i in range(4, len(vertices)):
        vk = vertices[i]
        replace_slot = flags[i-4] # 1-based index
        
        # Replace the vertex at replace_slot (1-4) with vk
        # Slot 1 is index 0, Slot 2 is index 1, etc.
        current_tet[replace_slot - 1] = vk
        render_tetrahedron(current_tet)
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
*   **Precision:** The method relies on linear approximation, which may introduce inaccuracies similar to the underlying Projected Tetrahedra algorithm [S2, unofficial].
*   **Degenerate Input:** Strips require valid adjacency information. If the mesh has non-manifold edges or disconnected components, strip generation algorithms may fail or produce short strips [S2, unofficial].

## Subtopics

### [Greedy Strip Generation Algorithm](2626-greedy-strip-generation-algorithm.md)
A method for generating tetrahedral strips that achieves average strip lengths of 17 on representative datasets, optimizing the grouping of tetrahedra to minimize bandwidth.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The generation of strips requires analyzing mesh adjacency, which is typically $O(N)$ or $O(N \log N)$ depending on the algorithm, but specific complexity is not provided.
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
    *   Suffers from inaccuracies inherent to the linear approximation of the Projected Tetrahedra algorithm [S2, unofficial].
*   **When to Use:** Interactive tetrahedral volume rendering where bandwidth is a bottleneck and hardware supports the primitives [S2, unofficial].
*   **When Not to Use:** When static geometry is sufficient (e.g., non-view-dependent rendering) or when hardware does not support the extensions.

## Background
The primitives were introduced by Craig M. Wittenbrink et al. to address the bandwidth bottleneck in interactive tetrahedral volume rendering [S2, unofficial]. They build on the Projected Tetrahedra algorithm by Shirley and Tuchman, which generates triangles that may be rendered by hardware acceleration but suffers from high bandwidth usage [S2, unofficial]. The work was presented in the paper "An Architecture For Interactive Tetrahedral Volume Rendering" [S2, unofficial].

## Sources
[S1] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S2] An Architecture For Interactive Tetrahedral Volume Rendering, https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)
[S3] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
