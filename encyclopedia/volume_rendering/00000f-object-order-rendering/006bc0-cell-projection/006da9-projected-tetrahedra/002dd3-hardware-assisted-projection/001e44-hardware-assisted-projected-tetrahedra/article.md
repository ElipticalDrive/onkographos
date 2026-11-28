# Hardware-Assisted Projected Tetrahedra

Also called: CellFast algorithm, Hardware CellFast

## Overview
Hardware-Assisted Projected Tetrahedra (HAPT) is a flexible, highly efficient volume rendering method grounded on the original Projected Tetrahedra (PT) algorithm [S2, unofficial]. It solves the visibility ordering and rendering bottleneck of irregular tetrahedral meshes by streaming sorted tetrahedra as point primitives, decomposing them into triangles in a geometry shader, and evaluating the volume ray integral per fragment during rasterization [S1, unofficial]. Unlike its sibling [PTINT](1861-tetrahedral-projection-with-partial-pre-integration.md), which requires a two-pass approach with GPU read-back, HAPT performs the entire rendering pipeline in a single GPU pass, eliminating auxiliary data structures and reducing GPU memory consumption [S2, unofficial].

## How it works
### Inputs and Outputs
*   **Input:** An unstructured tetrahedral mesh where each vertex has a scalar value. The data may be static or time-varying [S2, unofficial].
*   **Output:** A 2D image representing direct volume rendering (DVR) or iso-surface rendering [S3, unofficial].

### Data Structures
Tetrahedra are streamed to the GPU as point primitives [S1, unofficial]. To minimize data throughput, each point primitive represents one tetrahedron and carries the following attributes:
1.  **Position:** The position of one vertex of the tetrahedron (e.g., $v_0$) [S2, unofficial].
2.  **Texture Coordinates:** The positions of the other three vertices ($v_1, v_2, v_3$) are passed as texture coordinates [S2, unofficial].
3.  **Scalar Value:** The scalar value of the primary vertex is passed as the $w$ coordinate of the vertex [S2, unofficial].

### Sorting
Before rendering, tetrahedra must be sorted in back-to-front order relative to the viewpoint [S1, unofficial]. HAPT supports multiple sorting strategies:
*   **CPU Methods:** STL-based introsort or MPVONC (Meshed Polyhedra Visibility Ordering for Non-Convex meshes) [S2, unofficial]. MPVONC is exact but requires auxiliary adjacency lists and precomputed face normals, increasing CPU memory by ~4x the volume size [S2, unofficial].
*   **GPU Methods:** CUDA-based quicksort or bitonic sort [S2, unofficial]. Bitonic sort requires power-of-two array sizes, which may slow performance if padding is needed [S2, unofficial].
*   **Approximate Sorting:** Centroid sorting is used for GPU methods (except MPVONC) [S2, unofficial]. It is significantly faster but approximate, potentially inverting adjacent tetrahedra [S2, unofficial]. The error is low (average ~1 RGB unit on [0,255] scale) and visually imperceptible in most cases [S2, unofficial].

### Rendering Pipeline
The rendering is performed in a single pass through the GPU pipeline [S1, unofficial].
1.  **Vertex Shader (VS):** Receives the point primitives [S2, unofficial]. If time-varying data is used, an early discard test removes empty tetrahedra (all vertices mapped to zero opacity) [S2, unofficial].
2.  **Geometry Shader (GS):**
    *   Decomposes the point primitive into one to four triangles based on the tetrahedron's projection case [S2, unofficial].
    *   Determines the projection case using PTINT's classification strategy: four cross products and a table lookup [S2, unofficial].
    *   Computes three attributes for each generated vertex: traversal length $l$, scalar front $s_f$, and scalar back $s_b$ [S2, unofficial].
    *   **Vertex Types:**
        *   **Thick Vertex:** The entry point where the ray traverses the maximum distance $l$ [S2, unofficial]. Its scalar value must be interpolated, and $l$ may need to be computed if it is not an edge length [S2, unofficial]. This vertex may not be one of the original four projected vertices [S2, unofficial].
        *   **Thin Vertices:** All other projected vertices [S2, unofficial]. For these, $s_f = s_b$ (original scalar value) and $l = 0$, as the ray traverses no distance at these extremities [S2, unofficial].
    *   Stores $s_f, s_b, l$ as RGB color attributes of the vertices to pass information to the fragment shader [S2, unofficial].
    *   For iso-surface rendering, the GS extracts the iso-surface as one or two triangles with face normals [S2, unofficial].
3.  **Rasterization:** The hardware rasterizer processes the triangles [G].
4.  **Fragment Shader (FS):**
    *   Interpolates $s_f, s_b, l$ per fragment [S2, unofficial].
    *   Computes the volume ray integral using partial pre-integration [S1, unofficial].
    *   Accesses a pre-computed table using indices derived from $l$ and the front/back colors (extracted from $s_f/s_b$ via the transfer function) [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hapt_render(tetrahedra, viewpoint, transfer_function, pre_int_table):
    # 1. Sorting: sort tetrahedra back-to-front using CPU/GPU method [S2, unofficial]
    sorted_indices = sort_back_to_front(tetrahedra, viewpoint)
    
    # 2. Stream to GPU as point primitives [S2, unofficial]
    point_primitives = []
    for idx in sorted_indices:
        tet = tetrahedra[idx]
        point_primitives.append({
            'pos': tet.v0,
            'tex_coords': (tet.v1, tet.v2, tet.v3),
            'w': tet.scalar_v0
        })
    
    # 3. Geometry Shader Execution (per tetrahedron)
    triangles = []
    for prim in point_primitives:
        # Determine projection case using 4 cross products and a table lookup [S2, unofficial]
        case = ... # Not given in the sources.
        
        # Decompose into triangles based on case [S2, unofficial]
        tri_vertices = ... # Not given in the sources.
        
        for tri in tri_vertices:
            verts = []
            for v in tri:
                # Determine vertex type and compute attributes [S2, unofficial]
                if is_thick_vertex(v, case):
                    l = ... # Not given in the sources.
                    sf = ... # Not given in the sources.
                    sb = ... # Not given in the sources.
                else:
                    l = 0.0
                    sf = prim.w
                    sb = prim.w
                
                # Store as RGB color for interpolation [S2, unofficial]
                verts.append({'pos': v, 'color': (sf, sb, l)})
            triangles.append(verts)
    
    # 4. Rasterization and Fragment Shader Execution
    image = initialize_image()
    
    for tri in triangles:
        fragments = rasterize_triangle(tri)
        for frag in fragments:
            sf, sb, l = frag.color
            c_front = transfer_function(sf)
            c_back = transfer_function(sb)
            idx1 = ... # Not given in the sources.
            idx2 = ... # Not given in the sources.
            result = pre_int_table[idx1][idx2]
            image.composite(frag.pos, result)
            
    return image
```

### Edge Cases and Pitfalls
*   **Sorting Error:** Centroid sorting is approximate and may invert adjacent tetrahedra. While visually imperceptible in most cases, it introduces error (avg ~1 RGB unit) [S2, unofficial].
*   **MPVONC Degeneracy:** MPVONC does not perform correct visibility ordering in some degenerated cases, though it works for most meshes [S2, unofficial].
*   **Bitonic Sort Padding:** Bitonic sort requires power-of-two array sizes, which may slow performance if padding is needed [S2, unofficial].
*   **Empty Regions:** Early discard in the vertex shader is only profitable if the volume has large empty regions, due to texture fetch overhead [S2, unofficial].
*   **Iso-surface Normals:** Normals are restricted to faces (flat shading) unless adjacency info is carried, which impacts memory/performance [S2, unofficial].

## Subtopics

### [Tetrahedral Projection with Partial Pre-integration](1861-tetrahedral-projection-with-partial-pre-integration.md)
A hybrid method that uses pre-integrated optical depth tables. HAPT builds on this by leveraging GPU rasterization for cell projection, removing the need for GPU read-back and auxiliary basis graphs.

## Operating characteristics
*   **Time Complexity:** Sorting dominates the pre-processing time [S3, unofficial]. GPU sorting (quicksort) is fast, achieving over 6 M Tet/s [S1, unofficial]. Rendering is a single pass, efficient due to hardware rasterization [S2, unofficial].
*   **Memory Complexity:** Low GPU memory consumption [S2, unofficial]. Tetrahedra are streamed as point primitives, avoiding the need to store the entire volume in GPU memory [S1, unofficial]. This allows rendering of volumes with several million tetrahedra [S2, unofficial]. CPU memory for MPVONC is ~4x the volume size [S2, unofficial].
*   **Performance:** 
    *   Blunt fin (187 K Tet): 19.2 fps [S1, unofficial].
    *   Oxygen post (513 K Tet): 8.1 fps [S1, unofficial].
    *   Turbulent jet (1 M Tet/frame): 17.5 fps [S1, unofficial].
    *   spx2 (828 K Tet): 7.4 fps (DVR), 8.2 fps (Iso-surface) [S3, unofficial].
*   **Strengths:** Single-pass rendering, low GPU memory usage, support for time-varying data, hybrid DVR/iso-surface rendering [S1, unofficial].
*   **Limitations:** Requires geometry shader [S3, unofficial]. Centroid sorting is approximate [S2, unofficial]. MPVONC is memory-intensive and has degenerate cases [S2, unofficial]. Flat shading for iso-surfaces unless adjacency info is carried [S2, unofficial].
*   **When to use:** Large unstructured tetrahedral datasets where GPU memory is a constraint, or time-varying data requires streaming [S2, unofficial].
*   **When not to use:** When exact visibility ordering is critical and CPU memory is limited (MPVONC) [S2, unofficial]. When smooth iso-surface normals are required without adjacency overhead [S2, unofficial].

## Background
HAPT was introduced by A. Maximo, R. Marroquim, and R. Farias in 2010, published in the Journal of Computer Graphics Techniques [S3, unofficial]. It is an adaptation of the PTINT method, designed to exploit programmable graphics hardware by performing the entire algorithm in a single GPU pass [S2, unofficial]. It addresses the visibility ordering problem using sorting algorithms such as CUDA quicksort or CPU MPVONC, and leverages the geometry shader for triangle decomposition [S2, unofficial][S3, unofficial].

## Sources
[S1] (PDF) Hardware-Assisted Projected Tetrahedra - ResearchGate, https://www.researchgate.net/publication/220507188_Hardware-Assisted_Projected_Tetrahedra, fetched 2026-10-29 (unofficial)
[S2] PDF Hardware-Assisted Projected Tetrahedra, https://w3.impa.br/~andmax/papers/cgf2010.pdf, fetched 2026-10-28 (unofficial)
[S3] (PDF) Hardware-Assisted Projected Tetrahedra - Academia.edu, https://www.academia.edu/16536671/Hardware_Assisted_Projected_Tetrahedra, fetched 2026-10-28 (unofficial)
[S4] (PDF) An Architecture For Interactive Tetrahedral Volume ..., https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)
