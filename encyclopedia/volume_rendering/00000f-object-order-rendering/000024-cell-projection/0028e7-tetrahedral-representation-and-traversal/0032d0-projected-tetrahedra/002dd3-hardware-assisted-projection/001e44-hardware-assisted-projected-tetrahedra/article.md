# Hardware-Assisted Projected Tetrahedra

Also called: CellFast algorithm, Hardware CellFast

## Overview
Hardware-Assisted Projected Tetrahedra (HAPT) is a volume rendering method that projects tetrahedra onto the screen and rasterizes them as triangles, evaluating the volume ray integral per fragment in the fragment shader [S2, unofficial]. It solves the performance bottleneck of explicit isosurface extraction and the data transfer overhead of previous hybrid methods by performing the entire rendering pipeline on the GPU in a single pass [S1, unofficial][S2, unofficial]. Unlike its sibling [Projected Tetrahedra](877-projected-tetrahedra.md), which requires CPU-side sorting and triangle generation, HAPT streams sorted tetrahedra as point primitives and uses the GPU geometry shader to decompose them into triangles [S2, unofficial]. It differs from [Tetrahedral Projection with Partial Pre-integration](1861-tetrahedral-projection-with-partial-pre-integration.md) (PTINT) by eliminating the need to read data back from the GPU, allowing rendering of volumes with several million tetrahedra that would exceed GPU memory limits [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of tetrahedra defined by four vertices $(v_0, v_1, v_2, v_3)$ and a scalar value $s_i$ for each vertex [S2, unofficial]. The tetrahedra must be sorted in back-to-front visibility order [S2, unofficial].
*   **Output:** A final image composed of fragments with color and opacity.

**Data Structures:**
*   **Tetrahedron Stream:** Each tetrahedron is sent to the GPU as a single point primitive [S2, unofficial]. The vertex position is the position of one vertex (e.g., $v_0$) [S2, unofficial]. The other three vertices ($v_1, v_2, v_3$) are passed as texture coordinates [S2, unofficial]. The scalar value of the primary vertex is passed as the $w$ coordinate of the vertex attribute [S2, unofficial].
*   **Sorting Buffer:** An array of indices representing the sorted order of tetrahedra [S2, unofficial]. This can be stored in a Vertex Buffer Object (VBO) on the GPU or streamed from CPU memory if it exceeds VRAM [S2, unofficial].

**Algorithm Steps:**

1.  **Sorting:** Compute the visibility order of the tetrahedra [S2, unofficial]. This can be done on the CPU or GPU using methods such as: CPU: STL-based introsort, MPVONC (exact); GPU (CUDA): Quicksort, Bitonic Sort (requires power-of-two array size) [S2, unofficial][S3, unofficial]. Approximate sorting using tetrahedron centroids is commonly used for speed [S2, unofficial].
2.  **Streaming:** Send the sorted tetrahedra to the graphics pipeline [S2, unofficial]. If using GPU sorting, write indices directly to a VBO [S2, unofficial]. If the data exceeds VBO size, read back to CPU and stream [S2, unofficial].
3.  **Vertex Shader (VS):** Receive the point primitives [S2, unofficial]. Pass the vertex position, texture coordinates (other three vertices), and scalar value ($w$) to the geometry shader [S2, unofficial].
4.  **Geometry Shader (GS):**
    *   **Decomposition:** Decompose the point primitive into one to four triangles representing the projected tetrahedron [S2, unofficial].
    *   **Classification:** Determine the projection case using PTINT's classification strategy. This requires computing four cross products and performing a table lookup to identify if the vertex is 'thick' or 'thin' [S2, unofficial][S3, unofficial].
    *   **Vertex Type Computation:**
        *   **Thick Vertex:** The entry point where the ray traverses the maximum distance $l$ [S2, unofficial]. If the thick vertex is not one of the original projected vertices, its scalar value must be interpolated, and $l$ must be computed (if not an edge length) [S2, unofficial].
        *   **Thin Vertices:** All other projected vertices [S2, unofficial]. For these, $s_f = s_b$ (original scalar value) and $l = 0$ [S2, unofficial].
    *   **Attribute Assignment:** Store $s_f$ (scalar front), $s_b$ (scalar back), and $l$ (traversal length) as RGB color attributes for each vertex of the generated triangles [S2, unofficial].
    *   **Iso-surface Extraction (Optional):** If rendering iso-surfaces, extract the surface in the GS and send it as one or two triangles with face normals [S2, unofficial].
5.  **Rasterization:** The GPU rasterizes the triangles [S2, unofficial]. Scalar values ($s_f, s_b$) and traversal length ($l$) are interpolated per fragment [S2, unofficial].
6.  **Fragment Shader (FS):**
    *   **Partial Pre-integration:** Compute the volume ray integral using the interpolated $s_f, s_b,$ and $l$ [S2, unofficial]. Access a pre-computed table using indices derived from $l$ and the front/back colors (extracted from $s_f/s_b$ via the transfer function) [S2, unofficial].
    *   **Compositing:** Compose the final color and opacity for the fragment [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hapt_pipeline(sorted_indices, tetrahedra):
    # Pre-sorted tetrahedra are streamed as point primitives
    for idx in sorted_indices:
        tet = tetrahedra[idx]
        v = [tet.v0, tet.v1, tet.v2, tet.v3]
        s = [tet.s0, tet.s1, tet.s2, tet.s3]
        
        # Geometry Shader: Classification
        # Sources do not detail the exact cross-product logic; uses 4 cross products + table lookup [S2, unofficial]
        case_id = classify_tetrahedron(v, s)
        
        triangles = []
        for tri_idx in range(num_triangles(case_id)):
            if is_thick_vertex(tri_idx, case_id, v, s):
                # Thick vertex: entry point with max traversal distance l
                pos = compute_thick_vertex_position(case_id, v)
                sf = interpolate_scalar(case_id, v, s)
                sb = sf
                l = compute_traversal_length(case_id, v)
            else:
                # Thin vertex: original projected vertex
                orig_idx = get_original_vertex_index(tri_idx, case_id)
                pos = v[orig_idx]
                sf = s[orig_idx]
                sb = s[orig_idx]
                l = 0.0
            
            # Store sf, sb, l as RGB color attributes
            color = (sf, sb, l)
            triangles.append((pos, color))
        
        # Rasterization & Fragment Shader
        for pos, color in triangles:
            sf, sb, l = color
            # Partial Pre-integration
            idx_f = transfer_function_map(sf)
            idx_b = transfer_function_map(sb)
            alpha = lookup_pre_integration_table(l, idx_f, idx_b)
            rgb = compute_fragment_color(l, idx_f, idx_b)
            # Front-to-back compositing
            out_color = rgb * alpha + current_color * (1.0 - alpha)
            out_alpha = alpha + current_alpha * (1.0 - alpha)
```

**Parameters:**
*   **Sorting Method:** Centroid sorting (approximate, fast) or MPVONC (exact, slow, high memory) [S2, unofficial].
*   **Transfer Function:** Used in the fragment shader to map scalar values to color/opacity for the pre-integration table lookup [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Cases:** MPVONC may fail to perform correct visibility ordering in some degenerated cases, though it works for most meshes [S2, unofficial]. Class 4 in projection is a degenerate case with two faces perpendicular to the viewpoint, requiring no new vertices [S4, unofficial].
*   **Bitonic Sort:** Requires power-of-two array sizes, which may slow down sorting if the buffer is enlarged [S2, unofficial].
*   **Empty Tetrahedra:** For time-varying data, an early discard test in the vertex shader can remove tetrahedra with zero opacity, but requires large empty regions to be profitable due to texture fetch overhead [S2, unofficial].
*   **Precision:** Square root computations for Euclidean distances are implemented as lookup tables in hardware [S4, unofficial].

## Subtopics

### [Tetrahedral Projection with Partial Pre-integration](1861-tetrahedral-projection-with-partial-pre-integration.md)
A hybrid method that projects tetrahedra and computes fragment colors using pre-integrated optical depth tables. It solves the performance bottleneck of explicit isosurface extraction by computing isosurfaces per fragment within the rendering pipeline, avoiding the case-handling complexity of marching tetrahedra. Unlike full pre-integration, PTINT uses a transfer-function-independent $\psi$ table, allowing interactive transfer function editing without expensive table recomputation.

## Operating characteristics
*   **Time Complexity:** Sorting is the main bottleneck [S3, unofficial]. Centroid sorting is $O(N \log N)$ but approximate [S1, unofficial]. Exact sorting (MPVONC) is more expensive [S2, unofficial]. Rendering is $O(N)$ per frame after sorting [S2, unofficial].
*   **Memory Complexity:** Low GPU memory consumption as tetrahedra are streamed [S1, unofficial]. No auxiliary volume data structures are needed during rendering [S2, unofficial]. CPU memory for MPVONC increases by ~4x the volume size [S2, unofficial].
*   **Performance:** Achieves over 6 M Tet/s for unstructured datasets using CUDA-based visibility ordering [S2, unofficial].
    *   **spX2 (828K tet):** 7.4 FPS (6.11 M Tet/s) with HAPT (Quicksort) [S3, unofficial].
    *   **Turbulent Jet (1M tet/frame):** 17.5 FPS [S3, unofficial].
    *   **Blunt Fin (187K tet):** 19.2 FPS [S3, unofficial].
*   **Strengths:** Single-pass GPU rendering, low GPU memory usage, supports direct and indirect (iso-surface) rendering, handles time-varying data efficiently [S1, unofficial][S2, unofficial].
*   **Limitations:** Sorting cost is significant. Centroid sorting introduces error (avg ~0.4%, max ~5% depending on dataset) [S2, unofficial]. Requires geometry shader support [S3, unofficial].
*   **When to use:** Large unstructured volumes where GPU memory is a constraint, or when interactive transfer function editing is needed with partial pre-integration [S2, unofficial].
*   **When not to use:** When exact visibility ordering is critical and sorting time is prohibitive, or on hardware without geometry shader support [S3, unofficial].

## Background
HAPT was introduced by A. Maximo, R. Marroquim, and R. Farias in 2010, published in the Journal of Computer Graphics Techniques [S3, unofficial]. It builds on the original Projected Tetrahedra (PT) algorithm and PTINT, adapting them for full hardware acceleration [S1, unofficial]. It is used for direct volume rendering of torso, fighter, and turbjet datasets, and supports iso-surface extraction using the marching tetrahedra method [S3, unofficial].

## Sources
[S1] (PDF) Hardware-Assisted Projected Tetrahedra - ResearchGate, https://www.researchgate.net/publication/220507188_Hardware-Assisted_Projected_Tetrahedra, fetched 2026-10-29 (unofficial)
[S2] PDF Hardware-Assisted Projected Tetrahedra, https://w3.impa.br/~andmax/papers/cgf2010.pdf, fetched 2026-10-28 (unofficial)
[S3] (PDF) Hardware-Assisted Projected Tetrahedra - Academia.edu, https://www.academia.edu/16536671/Hardware_Assisted_Projected_Tetrahedra, fetched 2026-10-28 (unofficial)
[S4] (PDF) An Architecture For Interactive Tetrahedral Volume ..., https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)
