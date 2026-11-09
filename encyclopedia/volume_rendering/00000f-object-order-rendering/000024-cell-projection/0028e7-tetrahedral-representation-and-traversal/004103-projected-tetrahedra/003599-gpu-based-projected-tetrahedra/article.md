# GPU-based Projected Tetrahedra

Also called: Two-Shader PT, Vertex Shader PT, Tetrahedral Projection using Vertex Shaders

## Overview
GPU-based Projected Tetrahedra is a direct volume rendering method for unstructured tetrahedral data that leverages programmable graphics hardware to achieve interactive frame rates [S1, unofficial]. It extends the standard Projected Tetrahedra algorithm by shifting the geometric classification and decomposition of volume cells into vertex shaders, allowing the GPU to perform volume scan conversion without pre-processing or additional data structures [S2, unofficial][S3, unofficial]. Unlike traditional ray casting, it avoids explicit per-pixel ray-tetrahedron intersection tests by rasterizing projected triangles and using hardware blending [S1, unofficial].

## How it works
The method projects each tetrahedron onto the screen, classifies its orientation relative to the view vector, and rasterizes the resulting geometry to compute color and opacity accumulation.

### Inputs and Outputs
*   **Input:** Unstructured tetrahedral mesh (vertices, connectivity, scalar values).
*   **Output:** 2D framebuffer with accumulated color and opacity.

### Data Structures
*   **Basis Graph:** A fixed topology sent to the GPU for every tetrahedron. It consists of the four original vertices of the tetrahedron plus one additional "phantom" vertex [S2, unofficial][S3, unofficial].
*   **Vertex Buffer:** Contains the four real vertices and the invalid phantom vertex for each tetrahedron, sent as a triangle fan [S2, unofficial][S3, unofficial].
*   **Texture Memory:** Used to store the model data and avoid redundant projections [S1, unofficial].

### Numbered Steps
1.  **Data Submission:** Send the four vertices of the tetrahedron and an invalid phantom vertex to the GPU as a triangle fan [S2, unofficial][S3, unofficial].
2.  **Vertex Shader Classification:** A vertex program performs geometric tests on the projected vertices to determine the classification of the cell (i.e., the relative position of the view vector with respect to the tetrahedron faces) [S2, unofficial][S3, unofficial].
3.  **Phantom Vertex Calculation:** Based on the classification, the vertices are mapped to the basis graph, and the position of the phantom vertex is calculated [S2, unofficial][S3, unofficial].
4.  **Thickness Interpolation:** The thickness of the cell at the phantom vertex is determined by linearly interpolating the scalar values of the four real vertices [S2, unofficial][S3, unofficial].
5.  **Attribute Assignment:** Color and opacity are assigned to the phantom vertex using a lookup table based on the calculated thickness [S2, unofficial][S3, unofficial].
6.  **Rasterization:** The modified faces (defined by the basis graph) are rasterized [S2, unofficial][S3, unofficial].
7.  **Compositing:** The rasterized fragments are composited into the framebuffer using hardware blending [S2, unofficial][S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Program
def vertex_shader(tetrahedron_vertices, view_vector):
    """
    Classifies the tetrahedron and calculates the phantom vertex.
    tetrahedron_vertices: List of 4 vec3 positions.
    view_vector: vec3 direction of view.
    Returns: Modified vertex list with phantom vertex.
    """
    # 1. Perform geometric tests to determine classification
    # This involves checking the sign of the dot product between 
    # the view vector and the face normals of the tetrahedron.
    classification = classify_tetrahedron(tetrahedron_vertices, view_vector)
    
    # 2. Map vertices to the basis graph
    # The basis graph is a fixed topology of 5 vertices (4 real + 1 phantom)
    basis_vertices = map_to_basis_graph(tetrahedron_vertices, classification)
    
    # 3. Calculate phantom vertex position
    # The phantom vertex is placed to represent the "thickest" part 
    # of the tetrahedron along the view ray, based on classification.
    phantom_vertex = calculate_phantom_vertex(basis_vertices, classification)
    
    # 4. Determine thickness at phantom vertex
    # Linearly interpolate the scalar values of the 4 real vertices
    # to find the scalar value at the phantom vertex position.
    thickness = linear_interpolate_thickness(basis_vertices, phantom_vertex)
    
    # 5. Assign color and opacity from lookup table
    color = lookup_color(thickness)
    opacity = lookup_opacity(thickness)
    
    # 6. Return the 5 vertices (4 real + 1 phantom) with attributes
    # The phantom vertex carries the computed color/opacity.
    # The real vertices may carry interpolated attributes or be ignored 
    # depending on the specific rasterization strategy.
    return [basis_vertices[0], basis_vertices[1], basis_vertices[2], 
            basis_vertices[3], phantom_vertex], [color, opacity]

# Fragment Shader Program
def fragment_shader():
    """
    Computes the final color for the pixel.
    Uses hardware blending to accumulate color and opacity.
    """
    # The fragment shader receives the interpolated attributes 
    # from the rasterized triangle fan.
    # It performs standard alpha compositing:
    # C_out = C_in + C_frag * (1 - A_in)
    # A_out = A_in + A_frag * (1 - A_in)
    pass

# Main Render Loop
def render(tetrahedra, view_matrix, projection_matrix):
    for tet in tetrahedra:
        # Transform vertices to clip space
        clip_vertices = transform(tet.vertices, view_matrix, projection_matrix)
        
        # Send to GPU as triangle fan
        draw_triangle_fan(clip_vertices)
        
        # GPU performs classification, phantom vertex calculation, 
        # and rasterization/compositing
        pass
```

### Parameters
*   **Pre-processing:** None required [S1, unofficial].
*   **Additional Data Structures:** None required [S1, unofficial].
*   **Topology:** Fixed basis graph (5 vertices) [S2, unofficial][S3, unofficial].

### Edge Cases and Pitfalls
*   **Topology Constraints:** Current GPUs cannot change the topology of graphics primitives dynamically; therefore, the most general topology (the basis graph) must always be sent [S2, unofficial][S3, unofficial].
*   **Data Persistence:** No data can be saved across vertex primitives; each primitive must be sent with all relevant data structures [S2, unofficial][S3, unofficial].
*   **Precision:** The method relies on accurate geometric tests in the vertex shader; precision issues may affect classification correctness in degenerate cases [G].

## Subtopics

### Cell Projection with 3D/2D Textures
This variant stores color and opacity in a 3D texture map computed during pre-processing, enabling concurrent use of transfer functions [S4, unofficial]. All associated triangles are assigned to each vertex of each projected tetrahedron, and they are blended into the frame-buffer using back-to-front compositing [S4, unofficial]. To reduce memory consumption, the approach can be adapted to use 2D texture maps: one 2D texture map stores the linear approximation of dependencies of the volume rendering integral with respect to the length between sample points, and another stores the remaining parts of the integral that depend only on sample values [S4, unofficial].

### Hardware Caching for Projected Tetrahedra
This variant optimizes performance by maintaining a buffer holding the four vertices of the current tetrahedron and the additional fifth vertex created by the algorithm [S5, unofficial]. Vertices are swapped in the buffer as necessary to follow the direction of the strip [S5, unofficial]. It caches edge lengths and face areas computed for the current tetrahedron, as these are required by the Projected Tetrahedra algorithm and are reused by the next tetrahedron in a strip [S5, unofficial]. A pitfall is that computing edge lengths and face areas requires a square root, which impacts running time and complexity [S5, unofficial].

### Bunyk Tetrahedral Projection
A hybrid approach that projects the complete mesh onto the screen to simplify the problem from 3D to 2D [S6, unofficial]. Precomputed adjacency information is used to determine the correct accumulation order [S6, unofficial].

### GPU-based Tetrahedral Projection Code
A method used to simulate the initial set of radiation leaving a hohlraum [S7, unofficial]. It subdivides collapsed cells (single point) and cells with coinciding vertices or nonconvex quadrilateral faces into tetrahedra [S7, unofficial]. Remaining hexahedra are projected using the method of Max [S7, unofficial].

## Operating characteristics
*   **Performance:** Capable of rendering over 2.0 million tetrahedra per second (Tet/s) on current graphics hardware [S1, unofficial].
*   **Interactivity:** Allows interactive viewing of large unstructured datasets on a desktop personal computer [S1, unofficial].
*   **Memory Footprint:** Occupies a substantially smaller memory footprint compared to recent ray-casting approaches [S1, unofficial].
*   **Strengths:** No pre-processing or additional data structures required; leverages hardware rasterization and blending [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires sending the full basis graph for every tetrahedron, which may increase vertex data volume; reliance on vertex shader classification may be sensitive to precision [S2, unofficial][S3, unofficial].

## Background
The technique was introduced by Wylie et al. in 2002 as an extension of the Projected Tetrahedra algorithm by Shirley and Tuchman [S2, unofficial][S3, unofficial]. It was designed to map geometric computations into GPUs using programmable vertex shaders [S2, unofficial][S3, unofficial]. The approach was further refined by Marroquim et al. in 2006, who introduced a two-fragment-shader approach to avoid redundant projections and improve performance [S1, unofficial].

## Sources
[S1] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
[S2] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S3] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
[S4] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S5] (PDF) An Architecture For Interactive Tetrahedral Volume ..., https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)
[S6] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S7] Hardware-Accelerated Simulated Radiography, https://digital.library.unt.edu/ark:/67531/metadc876978/?q=front-to-back compositing, fetched 2026-10-29 (unofficial)
