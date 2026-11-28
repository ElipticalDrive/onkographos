# GPU-based Projected Tetrahedra

Also called: Two-Shader PT, Vertex Shader PT, Tetrahedral Projection using Vertex Shaders

## Overview
GPU-based Projected Tetrahedra is a method for interactive volume rendering of unstructured tetrahedral data that shifts geometric classification and decomposition from the CPU to the GPU using programmable shaders [S2, S3, S4, unofficial]. It solves the performance bottleneck of traditional Projected Tetrahedra by avoiding redundant projections and pre-processing, allowing interactive viewing of large datasets on desktop hardware [S1, S2, S3, S4, unofficial]. The family includes vertex-shader-based classification approaches introduced by Wylie et al. [S2, S3, S4, unofficial] and fragment-shader-based projection variants introduced by Marroquim et al. [S1, unofficial]. It differs from ray-casting approaches by occupying a substantially smaller memory footprint while achieving competitive frame rates [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Unstructured tetrahedral mesh data (vertex positions and scalar field values) [S1, S2, S3, S4, unofficial].
*   **Output:** A composited image in the framebuffer [S2, S3, S4, unofficial].

**Data Structures:**
*   **Basis Graph:** A fixed topology used for all tetrahedra, containing four vertices of the cell with an additional phantom vertex [S2, S3, S4, unofficial].
*   **Triangle Fan:** The primitive type sent to the GPU, comprising the four vertices and the phantom vertex [S2, S3, S4, unofficial].
*   **Texture Memory:** Used to store the model data to avoid redundant projections and improve access speed [S1, unofficial].
*   **Lookup Table:** Maps thickness (scalar value) to color and opacity [S2, S3, S4, unofficial].

**Algorithm Steps:**
1.  **Data Upload:** Store the tetrahedral model in GPU texture memory [S1, unofficial].
2.  **Primitive Submission:** Send the four vertices of a tetrahedron and an invalid phantom vertex to the GPU as a triangle fan [S2, S3, S4, unofficial].
3.  **Vertex Shader Classification:** A vertex program performs geometric tests on the input vertices to determine the classification of the cell relative to the view direction [S2, S3, S4, unofficial].
4.  **Basis Graph Mapping:** Based on the classification, map the four vertices to the basis graph topology [S2, S3, S4, unofficial].
5.  **Phantom Vertex Calculation:** Calculate the position of the phantom vertex based on the classification results [S2, S3, S4, unofficial].
6.  **Thickness Interpolation:** Determine the thickness of the cell at the phantom vertex by linearly interpolating the scalar values at the four vertices [S2, S3, S4, unofficial].
7.  **Attribute Assignment:** Assign color and opacity to the phantom vertex using a lookup table indexed by the calculated thickness [S2, S3, S4, unofficial].
8.  **Rasterization and Compositing:** Rasterize the modified faces (triangles formed by the basis graph) and composite them into the framebuffer using hardware blending in back-to-front order [S2, S3, S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def project_tetrahedron(vertices, scalars, view_direction, transfer_function):
    # vertices: list of 4 vec3, scalars: list of 4 float
    # 1. Classify tetrahedron relative to view direction
    classification = compute_classification(vertices, view_direction)
    
    # 2. Map vertices to fixed basis graph topology (5 slots: 4 corners + 1 phantom)
    basis_verts = [None] * 5
    for i in range(4):
        basis_verts[i] = vertices[permutation_table[classification][i]]
        
    # 3. Calculate phantom vertex position
    basis_verts[4] = compute_phantom_position(basis_verts[:4], classification)
    
    # 4. Interpolate thickness (scalar value) at phantom vertex
    thickness = interpolate_scalar(scalars, basis_verts[4], basis_verts[:4])
    
    # 5. Assign color and opacity from lookup table
    color, opacity = lookup_color_opacity(thickness, transfer_function)
    
    # 6. Rasterize basis graph faces and composite
    faces = get_basis_graph_faces(classification)
    for face in faces:
        tri_color, tri_opacity = interpolate_triangle(basis_verts, face, color, opacity)
        composite_back_to_front(tri_color, tri_opacity)
        
    return basis_verts, color, opacity

def compute_classification(vertices, view_direction):
    # Compute face normals for the tetrahedron
    normals = [cross(vertices[(i+1)%4]-vertices[i], vertices[(i+2)%4]-vertices[i]) for i in range(4)]
    # Dot view direction with normals to determine front/back facing
    signs = [1 if dot(n, view_direction) > 0 else -1 for n in normals]
    # Return integer classification code based on sign pattern
    return sum(s << i for i, s in enumerate(signs))

def compute_phantom_position(corners, classification):
    # Calculate phantom vertex position based on classification
    # Typically the centroid of the projected triangle fan
    return sum(corners) / 4.0

def interpolate_scalar(scalars, target_pos, corners):
    # Barycentric interpolation of scalar values at corners to target position
    # Returns thickness value
    pass

def lookup_color_opacity(thickness, transfer_function):
    # Sample precomputed transfer function or lookup table
    # Returns (color, opacity) tuple
    pass

def interpolate_triangle(basis_verts, face_indices, color, opacity):
    # Interpolate color and opacity across triangle vertices
    # Returns (tri_color, tri_opacity)
    pass

def composite_back_to_front(color, opacity):
    # Standard hardware blending for back-to-front compositing
    # out_color = color * opacity + prev_color * (1 - opacity)
    pass
```

**Parameters:**
*   **Lookup Table:** Maps thickness (scalar value) to color and opacity [S2, S3, S4, unofficial].
*   **Basis Graph Topology:** Fixed for all cells [S2, S3, S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Topology Constraints:** Current GPUs cannot change the topology of graphics primitives dynamically, so the most general topology (the basis graph with 5 vertices) must always be sent, even if the classification results in a simpler geometry [S2, S3, S4, unofficial].
*   **Data Independence:** No data can be saved across vertex primitives, so each primitive must be sent with all relevant data structures, as the GPU does not maintain state between independent draw calls for different tetrahedra [S2, S3, S4, unofficial].
*   **Precision:** Linear interpolation of thickness may introduce approximation errors in regions of high gradient [S2, S3, S4, unofficial].

## Subtopics

### [Cell Projection with 3D/2D Textures](1996-cell-projection-with-3d-2d-textures.md)
This method extends GPU-based Projected Tetrahedra by pre-computing volume rendering integrals into texture maps, enabling hardware-accelerated compositing of semi-transparent triangles [1996-cell-projection-with-3d-2d-textures.md]. It solves the limitation of standard projected tetrahedra by supporting arbitrary transfer functions via pre-integration, at the cost of increased memory usage for 3D textures or approximation errors for 2D textures [1996-cell-projection-with-3d-2d-textures.md]. It differs from [Sorted Rasterization of Projected Tetrahedra](3018-sorted-rasterization-of-projected-tetrahedra.md) by relying on texture-based integral approximation rather than explicit sorting of all triangles, and from [On-the-fly Shell Geometry Construction](13927-on-the-fly-shell-geometry-construction.md) by using pre-computed textures instead of dynamic geometry generation [1996-cell-projection-with-3d-2d-textures.md].

### [Sorted Rasterization of Projected Tetrahedra](3018-sorted-rasterization-of-projected-tetrahedra.md)
Sorted Rasterization of Projected Tetrahedra is a GPU-based direct volume rendering (DVR) method for unstructured tetrahedral data that achieves interactive rates by treating each tetrahedron as a distinct primitive and rasterizing them using hardware operations. It differs from traditional ray casting by leveraging hardware rasterization and blending to compute pixel color, avoiding explicit per-pixel ray-tetrahedron intersection tests.

### [On-the-fly Shell Geometry Construction](13927-on-the-fly-shell-geometry-construction.md)
On-the-fly shell geometry construction is a method for generating shell geometry (tetrahedrons or prisms) dynamically on the GPU during runtime, enabling shell mapping techniques to be applied to animated surfaces rather than requiring static, offline CPU preprocessing. It differs from traditional shell mapping by eliminating data-dependent preprocessing steps; instead, it uses a deterministic, data-independent rule based on vertex indices to ensure consistency between adjacent prisms, allowing for full parallelization in compute shaders.

### Hardware Caching for Projected Tetrahedra
This variant optimizes the GPU-based PT method by maintaining a buffer holding the four vertices of the current tetrahedron and the additional fifth vertex sometimes created by the algorithm [S5, unofficial]. It swaps the vertices in the buffer as necessary to follow the direction of the strip and caches edge lengths and face areas computed for the current tetrahedron, as these are required by the PT algorithm and are reused by the next tetrahedron in a strip [S5, unofficial]. A pitfall is that computing edge lengths and face areas requires a square root, which impacts running time and complexity [S5, unofficial].

### Bunyk Tetrahedral Projection
A hybrid approach for tetrahedral meshes that projects the complete mesh onto the screen to simplify the problem from 3D to 2D, using precomputed adjacency information for correct accumulation order [S6, unofficial]. The step involves projecting the complete mesh onto the screen [S6, unofficial].

### GPU-based Tetrahedral Projection Code
A method for projecting cells onto the detector, specifically used to simulate the initial set of radiation leaving the hohlraum [S7, unofficial]. It subdivides collapsed cells (single point) and cells with coinciding vertices or nonconvex quadrilateral faces into tetrahedra, and projects remaining hexahedra using the method of Max [S7, unofficial].

## Operating characteristics
*   **Performance:** Capable of rendering over 2.0 million tetrahedra per second (Tet/s) on current graphics hardware [S1, unofficial].
*   **Memory Footprint:** Occupies a substantially smaller memory footprint compared to ray-casting approaches [S1, unofficial].
*   **Pre-processing:** Requires no pre-processing of the data and no additional data structures [S1, unofficial].
*   **Scalability:** Allows interactive viewing of large unstructured datasets on a desktop personal computer [S1, unofficial].
*   **Strengths:** High performance, low memory usage, no pre-processing required, leverages hardware acceleration [S1, S2, S3, S4, unofficial].
*   **Limitations:** Requires sending the most general topology (basis graph) for all cells, which may involve redundant computation for simple cases; no data persistence across primitives limits optimization strategies that rely on inter-primitive state [S2, S3, S4, unofficial].

## Background
The method was introduced by Wylie et al. in 2002 as an extension of the Projected Tetrahedra (PT) algorithm [S2, S3, S4, unofficial]. It was designed to leverage the programmable vertex shaders available on commodity graphics cards to perform classification and decomposition of volume cells on the GPU [S2, S3, S4, unofficial]. This shifted the processing burden from the CPU to the GPU, enabling interactive visualization of unstructured data sets [S2, S3, S4, unofficial]. The approach is based on the PT algorithm by Shirley and Tuchman [S2, S3, S4, unofficial].

## Sources
[S1] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
[S2] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S3] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
[S4] A Survey of GPU-Based Volume Rendering of ..., https://www.sci.utah.edu/~csilva/papers/rita2005.pdf, fetched 2026-10-29 (unofficial)
[S5] (PDF) An Architecture For Interactive Tetrahedral Volume ..., https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)
[S6] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S7] Hardware-Accelerated Simulated Radiography, https://digital.library.unt.edu/ark:/67531/metadc876978/?q=front-to-back compositing, fetched 2026-10-29 (unofficial)
