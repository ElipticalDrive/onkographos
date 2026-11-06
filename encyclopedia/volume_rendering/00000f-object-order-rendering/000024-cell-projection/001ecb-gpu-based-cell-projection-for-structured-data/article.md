# GPU-based Cell Projection for Structured Data

Also called: Cell projection algorithm for large volumetric data sets, GPU-based PT for structured grids

## Overview
This method is an interactive volume rendering technique for large, regular (structured) volumetric data sets on GPUs. It solves the performance bottleneck of general Projected Tetrahedra (PT) by eliminating redundant projection and visibility sorting computations through the exploitation of data regularity and orthographic projection [S1, unofficial]. Unlike general PT, which requires expensive cell sorting, this method achieves constant-time traversal order and reuses projection parameters for all voxels, dedicating over 60% of processing time to the rendering pipeline itself [S1, unofficial]. It consumes less memory than Hardware-Based Ray Casting (HARC) while achieving high rendering speeds [S1, unofficial].

## How it works
**Inputs:** A structured 3D volume grid (regular hexahedral cells) and a camera view (orthographic).
**Outputs:** A 2D color image with alpha compositing.

**Data Structures:**
1.  **Basis Hexahedron Parameters:** Precomputed for a single reference hexahedron. For each of its 5 constituent tetrahedra, store:
    *   `basis_projection_class`: Integer (0-3) derived from cross-product tests.
    *   `basis_projected_vertices`: Screen coordinates of the 4 projected vertices.
    *   `basis_thick_vertex`: Screen coordinates of the "thick" vertex (the one with non-zero depth traversal).
    *   `basis_intersection_params`: Parameters to compute front ($s_f$) and back ($s_b$) scalar values at the thick vertex.
    *   `basis_rendering_order`: Integer defining the back-to-front draw order of the 5 tetrahedra.
2.  **Vertex Array:** A structure allocated on the CPU containing volume data. Each entry corresponds to a grid vertex and includes:
    *   3D lattice indices $(i, j, k)$.
    *   Original scalar value $s$.
    *   Packed normal vector: $x$-component stores `tetid` (0-4, which tetrahedron), $y$-component stores `vertid` (0-3, which vertex of the tetrahedron) [S1, unofficial].
3.  **Transfer Function:** A 1D texture with 256 positions mapping scalar values to RGBA [S1, unofficial].

**Steps:**

1.  **Preprocessing (CPU):** Subdivide each hexahedron in the volume into five tetrahedra. This set is termed a "volunit" [S1, unofficial].
2.  **Basis Projection (CPU):** Compute projection parameters for a single "basis hexahedron" once.
    *   Determine the projection class for each of the 5 basis tetrahedra using four cross-product tests [S1, unofficial].
    *   Compute the thick vertex coordinates, basis projected vertices coordinates, basis intersection parameters (for $s_f, s_b$), and basis rendering order for each tetrahedron [S1, unofficial].
    *   Store these values as global uniforms for the GPU [S1, unofficial].
3.  **Traversal Order (CPU):** Determine the traversal order for the whole volume in constant time. Because the grid is regular and orthographic projection is used, the relative depth order of voxels is fixed and can be computed directly without sorting [S1, unofficial].
4.  **Vertex Array Allocation (CPU):** Allocate volume information into the Vertex Array structure. Pack `tetid` and `vertid` into the vertex normal vector to shift computational load to the GPU [S1, unofficial].
5.  **Rendering (GPU):** Render each tetrahedron as a triangle fan.
    *   The basis hexahedron is iteratively displaced to the position of each volunit.
    *   The stored basis projected vertices are used to compose the tetrahedra triangles.
    *   Each triangle fan is rendered with the number of triangles relative to its basis projection class, following the basis rendering order.
    *   The first vertex of the fan (central node) is the basis thick vertex [S1, unofficial].
6.  **Vertex Shader (GPU):**
    *   Compute vertex coordinates on the fly based on 3 integer indices of the vertex in the lattice and the basis hexahedron information stored as global uniform variables [S1, unofficial].
    *   Decode `tetid` and `vertid` from the vertex normal [S1, unofficial].
    *   Compute scalar values ($s_f, s_b$) and thickness ($l$) for each vertex:
        *   For every vertex other than the thick vertex: $s_f = s_b = s$ (original scalar), $l = 0$ [S1, unofficial].
        *   For the thick vertex: Calculate $s_f$ and $s_b$ using the basis intersection parameters. Set $l$ to the basis thickness computed for that basis tetrahedron [S1, unofficial].
7.  **Fragment Shader (GPU):**
    *   Receive trilinearly interpolated vertex colors ($s_f, s_b, l$) for each triangle fragment [S1, unofficial].
    *   Use interpolated scalar values to lookup chromaticity and opacity values in the 1D transfer function texture (256 positions) [S1, unofficial].
    *   Compute opacity: $\alpha = 1 - e^{-\tau \cdot l}$, where $\tau$ is the extinction coefficient from the transfer function and $l$ is the interpolated thickness [S1, unofficial].
    *   Compose fragments in back-to-front order: $C_{fb} = \alpha \cdot C_{new} + (1 - \alpha) C_{fb}$, where $C_{fb}$ is the frame buffer color, and $C_{new}$ is the new fragment color [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global Uniforms (Basis Hexahedron Data)
# basis_params[tet_id] contains:
#   class: int (0-3)
#   projected_verts: vec4[4]
#   thick_vertex: vec2
#   thick_thickness: float
#   intersection_params: vec2 (for sf, sb calculation)
#   rendering_order: int

# Vertex Shader Input
# vertex_index: vec3 (i, j, k lattice indices)
# normal: vec3 (packed: x=tetid, y=vertid)

# Vertex Shader Output
# gl_Position: vec4
# frag_sf: float
# frag_sb: float
# frag_l: float

def vertex_shader(vertex_index, normal):
    tetid = int(normal.x)
    vertid = int(normal.y)
    
    # Get basis parameters for this tetrahedron
    bp = basis_params[tetid]
    
    # Compute screen position based on lattice index and basis displacement
    # This effectively displaces the basis hexahedron to the current volunit
    screen_pos = compute_screen_position(vertex_index, bp)
    
    # Determine if this is the thick vertex
    is_thick = (vertid == get_thick_vertex_id(bp.class))
    
    if is_thick:
        # Calculate front and back scalars using basis intersection parameters
        # and the scalar value at the thick vertex location
        sf, sb = compute_scalars_from_params(bp.intersection_params, vertex_index)
        l = bp.thick_thickness
    else:
        # Non-thick vertices have zero thickness and uniform scalar
        sf = get_scalar_at(vertex_index)
        sb = sf
        l = 0.0
        
    gl_Position = vec4(screen_pos, 0.0, 1.0)
    frag_sf = sf
    frag_sb = sb
    frag_l = l

def fragment_shader(frag_sf, frag_sb, frag_l):
    # Interpolate scalars if necessary (though sf/sb are per-vertex, 
    # trilinear interpolation happens across the triangle)
    # The source states trilinearly interpolated vertex colors are received.
    # In standard PT, sf and sb are interpolated to get the scalar at the fragment.
    # However, the opacity depends on thickness l.
    
    # Calculate average scalar for color lookup
    avg_scalar = (frag_sf + frag_sb) / 2.0
    
    # Lookup transfer function (1D texture, 256 positions)
    tf_rgba = texture1D(transfer_function_texture, avg_scalar)
    C_new = tf_rgba.rgb
    tau = tf_rgba.a # Extinction coefficient
    
    # Compute opacity
    alpha = 1.0 - exp(-tau * frag_l)
    
    # Back-to-front compositing
    C_fb = texture2D(frame_buffer, gl_FragCoord.xy)
    C_final = alpha * C_new + (1.0 - alpha) * C_fb
    
    return C_final, alpha
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on orthographic projection to ensure all volunits are projected identically. Perspective projection would break the assumption that basis parameters can be reused, requiring per-voxel projection [S1, unofficial].
*   **Boundary Conditions:** The constant-time traversal order assumes a regular grid. Irregular grids are not supported by this specific optimization [S1, unofficial].
*   **Thick Vertex Identification:** Correctly identifying the "thick vertex" based on the projection class is critical for computing non-zero thickness and correct scalar interpolation. Errors here lead to incorrect opacity and color [S1, unofficial].

## Subtopics

### [Parallel Cell Projection](875-parallel-cell-projection.md)
Distributes the cell projection workload across multiple processors/GPUs. Choose when single-GPU memory or compute limits are reached.

### [Cell Projection with Texture Mapping for Opacity](993-cell-projection-with-texture-mapping-for-opacity.md)
Uses texture mapping to handle opacity, potentially simplifying the fragment shader logic. Choose when texture hardware acceleration for opacity is preferred over manual blending.

### [Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md)
Implements a topological sort for visibility ordering. Choose when the data is not a regular grid and constant-time traversal is not possible.

### [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-accurate-sorting-for-projected-tetrahedra.md)
Provides accurate sorting for projected tetrahedra in parallel. Choose when high-quality transparency requires precise depth sorting over the speed of constant-time traversal.

### [Cell Projection with Scan-line Conversion](1968-cell-projection-with-scan-line-conversion.md)
Uses scan-line conversion for rasterization. Choose when fine-grained control over rasterization is needed or for specific hardware architectures.

### [Cell-Projection Rendering](4279-cell-projection-rendering.md)
General cell projection rendering techniques. Choose for a broader overview of the family.

### [Tetrahedral representation and traversal](4396-tetrahedral-representation-and-traversal.md)
Details on tetrahedral mesh generation and traversal. Choose for understanding the underlying geometric representation.

### [Sorting-free Pre-integrated Projected Tetrahedra](7491-sorting-free-pre-integrated-projected-tetrahedra.md)
Pre-integrates opacity to avoid sorting. Choose when pre-integration is feasible and desired for performance.

### [Incoherent Projection](7541-incoherent-projection.md)
Handles incoherent data access patterns. Choose when data locality is poor.

## Operating characteristics
*   **Time Complexity:** Projection and sorting are $O(1)$ per voxel due to regularity and orthographic projection. Rendering is dominated by the GPU pipeline.
*   **Memory Complexity:** Lower than HARC. The method stores only basis parameters and vertex data, avoiding the high memory consumption of ray casting approaches [S1, unofficial].
*   **Performance:** Achieves rendering speeds of over 5.0 M Tet/s on current graphics hardware [S1, unofficial]. More than 60% of processing time is dedicated to the rendering pipeline, indicating efficient CPU-GPU handoff [S1, unofficial].
*   **Strengths:** High performance for large structured datasets; low memory usage; simple visibility ordering.
*   **Limitations:** Restricted to regular (structured) grids; requires orthographic projection for optimal performance; preprocessing step required to split hexahedra into tetrahedra [S1, unofficial].
*   **When to use:** Interactive visualization of large medical or scientific datasets that are structured grids.
*   **When not to use:** Irregular/unstructured grids; perspective projection requirements where basis reuse is invalid; scenes requiring highly accurate transparency sorting that cannot be approximated by the grid order.

## Background
The method is an implementation of the Projected Tetrahedra (PT) algorithm proposed by Shirley and Tuchman (1990), adapted for GPUs and structured data [S1, unofficial]. It addresses the computational cost of visibility sorting in general PT algorithms, which was previously addressed by works such as Williams (1992), Stein et al. (1994), and Comba et al. (1999) [S1, unofficial]. The first GPU PT implementation, GATOR (Wylie et al., 2002), was noted as fast but redundant [S1, unofficial]. This method improves upon GATOR by eliminating redundancy through basis hexahedron reuse [S1, unofficial]. It is used in medical and scientific visualization for interactive analysis of large 3D data sets, such as those ranging from $256^3$ to $512^3$ voxels [S1, unofficial].

## Sources
[S1] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2007/20825/, fetched 2026-10-29 (unofficial)
