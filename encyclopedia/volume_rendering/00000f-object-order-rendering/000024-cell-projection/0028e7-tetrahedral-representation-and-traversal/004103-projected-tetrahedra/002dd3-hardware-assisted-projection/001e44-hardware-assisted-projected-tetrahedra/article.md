# Hardware-Assisted Projected Tetrahedra

Also called: CellFast algorithm, Hardware CellFast

## Overview
Hardware-Assisted Projected Tetrahedra (HAPT) is a flexible, highly efficient volume rendering method that adapts the original Projected Tetrahedra (PT) algorithm to exploit programmable graphics hardware [S1, S2, S3, unofficial]. It solves the performance bottleneck of software-based cell projection by streaming sorted tetrahedra as point primitives to the GPU, decomposing them into triangles in a geometry shader, and evaluating the volume ray integral per fragment in a fragment shader [S1, S2, unofficial]. Unlike its sibling [Hardware-Assisted Polyhedron Projection](1453-hardware-assisted-polyhedron-projection.md), which handles general convex polyhedra, HAPT is specialized for tetrahedral meshes, allowing it to use a single-pass GPU pipeline that avoids the auxiliary basis graph structure required by methods like GATOR [S2, S3, unofficial]. It builds on [Hardware-Assisted Projection](5625-hardware-assisted-projection.md) by leveraging GPU rasterization for cell projection [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An unstructured tetrahedral mesh (positions, scalar values per vertex) and a viewpoint.
*   **Output:** A 2D image with direct volume rendering (DVR) or iso-surface rendering (ISO) colors.

**Data Structures:**
*   **Tetrahedron Stream:** Each tetrahedron is represented as a single point primitive sent to the GPU.
    *   Vertex Position: The position of one vertex of the tetrahedron.
    *   Texture Coordinates: The positions of the other three vertices of the tetrahedron.
    *   Scalar Value: The scalar value of the primary vertex is passed as the `w` coordinate of the vertex attribute [S2, unofficial].
*   **Sorting Indices:** A buffer of tetrahedron indices sorted by visibility order [S2, unofficial].

**Numbered Steps:**

1.  **Visibility Sorting:**
    Sort the tetrahedra in back-to-front order relative to the viewpoint. This is a prerequisite for correct alpha blending [S1, S4, unofficial]. Methods include:
    *   CPU-based: STL-based introsort, MPVONC (exact, requires adjacency list and face normals) [S2, S3, unofficial].
    *   GPU-based: CUDA Quicksort, CUDA Bitonic Sort (requires power-of-two array size) [S2, S3, unofficial].
    *   Approximate: Centroid sorting (fast, low error) [S2, S3, unofficial].

2.  **Streaming to GPU:**
    Stream the sorted tetrahedra to the graphics pipeline. To avoid GPU memory limits, tetrahedra are stored in a Vertex Buffer Object (VBO) and streamed in chunks if the volume exceeds VRAM [S1, S2, unofficial]. For time-varying data, frames are streamed as separate static volumes [S2, unofficial].

3.  **Vertex Shader (VS):**
    Receive the point primitive. Optionally, perform an early discard test for empty tetrahedra (all vertices map to zero opacity) [S2, unofficial]. Pass the vertex position and the three texture coordinates (other vertices) to the Geometry Shader [S2, unofficial].

4.  **Geometry Shader (GS):**
    *   **Decomposition:** Decompose the point primitive into one to four triangles representing the projected tetrahedron [S1, S2, unofficial].
    *   **Classification:** Determine the correct projection case using PTINT's classification strategy. This involves four cross products and a table lookup to identify vertex types [S2, S3, unofficial].
    *   **Vertex Attribute Computation:** For each generated triangle vertex, compute:
        *   **Traversal Length ($l$):** The length of the viewing ray segment within the tetrahedron.
        *   **Scalar Front ($s_f$) and Scalar Back ($s_b$):** The scalar values at the entry and exit points of the ray.
    *   **Vertex Types:**
        *   **Thick Vertex:** The entry point where the ray traverses the maximum distance $l$. Its scalar value may need interpolation, and $l$ may need calculation if it is not an edge length. It may not be one of the original four projected vertices [S2, unofficial].
        *   **Thin Vertices:** All other projected vertices. For these, $s_f = s_b$ (original scalar value) and $l = 0$ [S2, unofficial].
    *   **Output:** Send the triangle primitives with $s_f$, $s_b$, and $l$ encoded as RGB color attributes (or normals for ISO) to the Fragment Shader [S2, unofficial].

5.  **Rasterization:**
    The hardware rasterizes the triangles. Scalar values ($s_f, s_b$) and traversal length ($l$) are interpolated per fragment [S2, unofficial].

6.  **Fragment Shader (FS):**
    Evaluate the volume ray integral using the interpolated values.
    *   **Direct Volume Rendering (DVR):** Use partial pre-integration. Access a pre-computed table using indices computed from $l$ and the front/back colors (extracted from $s_f/s_b$ via the transfer function) [S2, unofficial].
    *   **Iso-Surface Rendering (ISO):** Extract the iso-surface in the GS (as one or two triangles) and render with flat shading (normals restricted to faces) [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vertex_shader(tetrahedron_point, other_three_vertices, scalar_w):
    # Pass data to Geometry Shader
    # tetrahedron_point: position of one vertex
    # other_three_vertices: positions of the other three vertices (as texture coords)
    # scalar_w: scalar value of the primary vertex
    return {
        'pos': tetrahedron_point,
        'tex_coords': other_three_vertices,
        'scalar': scalar_w
    }

# Geometry Shader
def geometry_shader(input_point):
    # 1. Decompose into triangles
    # Determine projection case using PTINT classification (4 cross products + table lookup)
    # This determines if vertices are 'thick' or 'thin'
    
    triangles = []
    
    # 2. For each vertex of the generated triangles:
    for vertex in projected_vertices:
        if vertex is 'thick':
            # Compute traversal length l (may require interpolation of scalar and distance calc)
            l = compute_traversal_length(vertex, input_point)
            # Interpolate scalar value if necessary
            s_f = interpolate_scalar(vertex, input_point)
            s_b = s_f # Or computed based on exit point
        else: # 'thin' vertex
            l = 0.0
            s_f = input_point['scalar']
            s_b = input_point['scalar']
            
        # Encode s_f, s_b, l as RGB color
        color = encode_rgb(s_f, s_b, l)
        
        triangles.append({
            'vertex': vertex,
            'color': color
        })
        
    return triangles

# Fragment Shader
def fragment_shader(interpolated_color):
    # Decode s_f, s_b, l from RGB
    s_f, s_b, l = decode_rgb(interpolated_color)
    
    # Partial Pre-Integration
    # Map s_f and s_b to colors using transfer function
    color_f = transfer_function(s_f)
    color_b = transfer_function(s_b)
    
    # Compute indices for pre-integration table
    idx_l = map_to_table_index(l)
    idx_colors = map_colors_to_table_index(color_f, color_b)
    
    # Lookup pre-computed ray integral
    result_color = pre_integration_table[idx_l][idx_colors]
    
    return result_color
```

**Parameters:**
*   **Sorting Method:** Quicksort (CUDA), Bitonic Sort (CUDA), Introsort (CPU), MPVONC (CPU), or Centroid Sort [S2, S3, unofficial].
*   **Pre-integration Table:** Pre-computed table for partial pre-integration, independent of the transfer function [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Sorting Error:** Centroid sorting is approximate and can invert the order of adjacent tetrahedra, introducing visual error (avg ~1 RGB unit, max 1.2-5.1% depending on dataset) [S2, S3, unofficial].
*   **MPVONC Degeneracy:** MPVONC may fail to perform correct visibility ordering in some degenerated cases [S2, S3, unofficial].
*   **Bitonic Sort Size:** CUDA Bitonic Sort requires power-of-two array sizes, necessitating buffer enlargement and potentially slower performance [S2, unofficial].
*   **Memory Usage:** MPVONC requires auxiliary data structures (adjacency list, face normals), increasing CPU memory by ~4x the volume size [S2, unofficial].
*   **Early Discard Overhead:** Early discard in VS requires large empty regions to be profitable due to texture fetch overhead [S2, unofficial].

## Subtopics

### [Tetrahedral Projection with Partial Pre-integration](1861-tetrahedral-projection-with-partial-pre-integration.md)
PTINT is a hybrid method that projects tetrahedra and computes fragment colors using pre-integrated optical depth tables. It solves the performance bottleneck of explicit isosurface extraction by computing isosurfaces per fragment. Unlike full pre-integration, PTINT uses a transfer-function-independent $\psi$ table. It builds on HAPT by leveraging GPU rasterization for cell projection.

## Operating characteristics
*   **Time Complexity:** Dominated by sorting and rasterization. Sorting can be $O(N \log N)$ or $O(N)$ depending on the method (e.g., Quicksort vs. Radix/Bitonic). Rasterization is linear in the number of projected triangles.
*   **Memory Complexity:** Low GPU memory usage as tetrahedra are streamed. CPU memory usage depends on the sorting method (MPVONC uses ~4x volume size) [S2, unofficial].
*   **Performance:** Achieves over 6 M Tet/s for unstructured datasets using CUDA-based visibility ordering [S1, S2, unofficial].
    *   Blunt fin (187 K Tet): 19.2 fps [S1, S3, unofficial].
    *   Oxygen post (513 K Tet): 8.1 fps [S1, S3, unofficial].
    *   Turbulent jet (1 M Tet/frame): 17.5 fps [S1, S3, unofficial].
    *   HAPT (Quicksort) on spx2 (828K tet): 7.4 FPS, 6.11 M Tet/s [S3, unofficial].
*   **Strengths:** Single-pass GPU rendering, low GPU memory consumption, supports direct and indirect rendering, handles time-varying data efficiently [S1, S2, unofficial].
*   **Limitations:** Requires sorting (visibility ordering is a disadvantage for cell-projection algorithms), centroid sorting introduces error, MPVONC is memory-intensive [S2, S3, unofficial].
*   **When to use:** When rendering large unstructured tetrahedral meshes on modern GPUs with limited VRAM, and interactive performance is required.
*   **When not to use:** When exact visibility ordering is critical and sorting overhead is prohibitive, or when the hardware does not support geometry shaders.

## Background
HAPT was introduced by A. Maximo, R. Marroquim, and R. Farias in 2010, published in the Journal of Computer Graphics Techniques [S3, unofficial]. It is grounded on the original Projected Tetrahedra (PT) algorithm [ST90] and developed to exploit programmable graphics hardware [S3, unofficial]. It addresses the visibility ordering problem using a fast quicksort algorithm adapted to CUDA [S3, unofficial]. It is used for direct volume rendering of torso, fighter, and turbjet datasets, and supports iso-surface extraction [S3, unofficial].

## Sources
[S1] (PDF) Hardware-Assisted Projected Tetrahedra - ResearchGate, https://www.researchgate.net/publication/220507188_Hardware-Assisted_Projected_Tetrahedra, fetched 2026-10-29 (unofficial)
[S2] PDF Hardware-Assisted Projected Tetrahedra, https://w3.impa.br/~andmax/papers/cgf2010.pdf, fetched 2026-10-28 (unofficial)
[S3] (PDF) Hardware-Assisted Projected Tetrahedra - Academia.edu, https://www.academia.edu/16536671/Hardware_Assisted_Projected_Tetrahedra, fetched 2026-10-28 (unofficial)
[S4] (PDF) An Architecture For Interactive Tetrahedral Volume ..., https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)
