# Projected Tetrahedra

Also called: Shirley and Tuchman, Shirley-Tuchman Method

## Overview
Projected Tetrahedra (PT) is an object-order (cell-by-cell) volume rendering technique that projects volumetric primitives, specifically tetrahedra, onto the 2D image plane to leverage standard graphics hardware for scan conversion and compositing [S2, S4, unofficial][S7]. It solves the problem of efficiently rendering unstructured tetrahedral grids (such as finite element meshes) by approximating the volume with partially transparent polygons, avoiding the stair-stepping artifacts of voxel-based methods and the high memory overhead of ray casting [S2, S9, unofficial]. Unlike [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md), which often employs explicit ray-tetrahedron intersection tests in a GPGPU pipeline, PT decomposes the projected 3D cell into 2D triangles in software or vertex shaders and relies on hardware rasterization and back-to-front compositing [S3, unofficial][S6, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh where each vertex $V_i$ has associated scalar values $s_i$ (e.g., density, temperature). The mesh may be derived from decomposing hexahedral cells [S9, S14, unofficial].
*   **Output:** A 2D image with color and opacity, composited using the Porter-Duff over operator [S1, S10, unofficial].

**Data Structures:**
*   **Tetrahedron:** Defined by four vertices $\{V_0, V_1, V_2, V_3\}$ in 3D space [S13, unofficial][S15].
*   **Basis Graph:** A fixed topology of 5 vertices used in GPU implementations: the 4 real tetrahedron vertices plus one "phantom" vertex $V_4'$ [S13, unofficial][S15]. The topology is drawn as a `GL_TRIANGLE_FAN` starting at $V_4'$, then $V_0', V_1', V_2', V_3'$, closing back to $V_0'$ [S13, unofficial].
*   **Classification Texture:** A 1D texture (8 bits per component) mapping test permutations to projection class and vertex order [S8].
*   **Vertex Data:** Per-vertex attributes typically include position (3 floats), scalar value (1 float), and potentially gradient/normal data (up to 15 floats total in some hardware-assisted variants) [S17].

**Numbered Steps:**

1.  **Decomposition:** Decompose the volume into tetrahedral cells. For regular hexahedral grids, each cube is subdivided into 5 or 6 tetrahedra. To avoid "cracks" at shared faces, adjacent cells must use alternating rotational states of the decomposition (a 3D checkerboard pattern) [S9, S14, S21, unofficial].
2.  **Visibility Sorting:** Sort the tetrahedra in back-to-front order relative to the viewpoint. This is typically done by sorting centroids, which is an approximation; exact sorting requires algorithms like MPVO or k-buffer sorting [S1, unofficial][S8, S12].
3.  **Projection and Classification:** For each tetrahedron, project the 3D vertices to 2D screen space. Classify the tetrahedron into one of four projection classes based on the visibility of its faces relative to the view direction [S2, S14, unofficial].
    *   **Class 1:** All 4 faces visible (+ + + +).
    *   **Class 2:** 3 faces visible, 1 hidden (+ + + -).
    *   **Class 3:** 2 faces visible, 2 hidden (+ + - - or + - + -). Degenerate cases of Class 1/2.
    *   **Class 4:** 1 face visible, 3 hidden (+ - - -). Degenerate case of Class 2.
    *   *Classification Logic:* Determine the sign of the dot product between the view vector and the face normals, or use cross-product tests on projected vertices [S14, unofficial][S15].
4.  **Decomposition into Triangles:** Decompose the projected outline of the tetrahedron into 1 to 4 triangles based on the classification [S1, S10, unofficial].
    *   Class 1: 3 triangles.
    *   Class 2: 4 triangles.
    *   Classes 3 & 4: Degenerate, handled by creating zero-area triangles or mapping to Class 1/2 logic [S13, unofficial].
5.  **Compute Thick Vertex:** Identify the "thick vertex," defined as the point on the ray segment traversing the maximum distance through the tetrahedron [S7].
    *   For Class 2, the thick vertex is the intersection of the front-facing and back-facing edges [S7][S20, unofficial].
    *   For other classes, it is one of the projected vertices [S7].
    *   In GPU implementations, this is computed via a "phantom vertex" in the basis graph [S13, unofficial][S15].
6.  **Compute Color and Opacity:** Calculate the scalar values at the entry ($s_f$) and exit ($s_b$) points of the ray through the tetrahedron [S7].
    *   **Thin Vertices:** Have $s_f = s_b = s_v$ (the projected scalar value). They are assigned zero opacity and original chromaticity [S7][S9, unofficial].
    *   **Thick Vertex:** Requires interpolation of $s_f$ and $s_b$ from thin vertices if not directly available [S7][S9, unofficial].
    *   **Opacity:** Compute opacity $\alpha$ using the thickness $l$ (ray length inside the tetrahedron) and average extinction coefficient $\tau_{avg}$:
        $$ \alpha = 1 - \exp(-\tau_{avg} l) $$
        where $\tau_{avg}$ is derived from the transfer function evaluated at $(s_f + s_b)/2$ [S7, S8].
    *   **Color:** Map scalar values to color $C(s)$ using a transfer function lookup [S7].
7.  **Rasterization and Compositing:** Rasterize the decomposed triangles using hardware Gouraud shading. Interpolate color and opacity across the triangles. Composite fragments back-to-front using the Porter-Duff over operator:
    $$ I_{new} = \alpha C + (1 - \alpha) I $$
    where $I$ is the previous framebuffer color, $C$ is the fragment color, and $\alpha$ is the fragment opacity [S7, S8].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for a single tetrahedron projection and rendering step
# Assumes tetrahedron vertices V[0..3] have 3D positions and scalar values s

def project_and_render_tetrahedron(V, view_matrix, projection_matrix, transfer_function):
    # 1. Project vertices to screen space
    screen_vertices = []
    for i in range(4):
        # Transform 3D vertex to 2D screen coordinates
        pos_3d = V[i].position
        pos_4d = transform(pos_3d, view_matrix)
        pos_4d = transform(pos_4d, projection_matrix)
        # Perspective divide
        if pos_4d.w != 0:
            x = pos_4d.x / pos_4d.w
            y = pos_4d.y / pos_4d.w
            screen_vertices.append({'x': x, 'y': y, 's': V[i].scalar, 'z': pos_4d.z / pos_4d.w})
        else:
            screen_vertices.append(None) # Handle infinity

    # 2. Classify tetrahedron
    # Determine visibility of faces using cross products of projected edges
    # Example logic for Class determination (simplified from S13, S15)
    v0, v1, v2, v3 = screen_vertices
    vec1 = v1 - v0
    vec2 = v2 - v0
    vec3 = v3 - v0
    
    # Cross product z-components
    cross1_z = cross_product_z(vec1, vec2)
    cross2_z = cross_product_z(vec1, vec3)
    cross3_z = cross_product_z(vec2, vec3)
    
    # Boolean tests for classification (S13)
    test1 = (cross1_z * cross2_z < 0)
    test2 = (cross1_z * cross3_z > 0)
    # test3 and test4 depend on specific intersection logic for thick vertex
    # ... (See S13 for full 14-case truth table)
    
    class_id = determine_class(test1, test2, ...) 
    
    # 3. Determine triangles and thick vertex
    triangles = []
    thick_vertex = None
    
    if class_id == 1:
        # Class 1: 3 triangles, thick vertex is one of the original vertices
        # Decompose into 3 triangles sharing a common vertex
        triangles = [(v0, v1, v2), (v0, v2, v3), (v0, v3, v1)]
        thick_vertex = v0 # Or whichever is thickest, often determined by depth
    elif class_id == 2:
        # Class 2: 4 triangles, thick vertex is intersection of front/back edges
        # Find intersection point I of the projected edges
        I = intersect_edges(v0, v1, v2, v3) # Simplified
        thick_vertex = I
        triangles = [(v0, v1, I), (v1, v2, I), (v2, v3, I), (v3, v0, I)]
    else:
        # Degenerate cases (3, 4)
        # Map to Class 1 or 2 logic or create zero-area triangles
        pass

    # 4. Compute Opacity and Color for vertices
    for tri in triangles:
        for vertex in tri:
            if vertex == thick_vertex:
                # Compute s_f and s_b for thick vertex
                # s_f and s_b are interpolated from thin vertices or computed via ray integration
                s_f, s_b = compute_entry_exit_scalars(V, thick_vertex)
                l = compute_thickness(V, thick_vertex) # Ray length inside tetrahedron
                tau_avg = transfer_function.get_extinction((s_f + s_b) / 2)
                alpha = 1 - exp(-tau_avg * l)
                color = transfer_function.get_color((s_f + s_b) / 2)
            else:
                # Thin vertex
                alpha = 0.0
                color = transfer_function.get_color(vertex.s)
            
            vertex.color = color
            vertex.alpha = alpha

    # 5. Rasterize and Composite
    for tri in triangles:
        rasterize_triangle(tri)
        composite_back_to_front()

def compute_entry_exit_scalars(V, thick_vertex):
    # Interpolate scalar values at entry and exit points
    # This requires knowing the ray direction and intersection points
    # Simplified: assume linear variation
    return V[0].scalar, V[1].scalar # Placeholder for exact interpolation logic

def compute_thickness(V, thick_vertex):
    # Calculate distance traversed by ray through tetrahedron
    # This is the distance between entry and exit points on the ray
    return 0.0 # Placeholder
```

**Parameters:**
*   **Transfer Function:** Maps scalar $s$ to color $C(s)$ and extinction $\tau(s)$. Can be a 2D texture map indexed by $s$ and thickness $l$ (pre-integration) [S5, S7].
*   **Sorting Algorithm:** Centroid sorting is common but approximate; MPVO or k-buffer for exactness [S8, S12].
*   **Bucket Sort Group Count:** Approximately 20 slabs perpendicular to the viewing vector for efficient sorting [S8].

**Edge Cases and Pitfalls:**
*   **Degeneracy:** Classes 3 and 4 occur when a vertex projects onto an edge or another vertex. These must be handled as zero-area triangles or mapped to Classes 1/2 [S7][S13, unofficial].
*   **Sorting Errors:** Centroid sorting is not 100% accurate, leading to visual artifacts if cells are elongated or overlapping significantly. Exact sorting is required for correctness but is expensive [S8, S12].
*   **Mach Banding:** Linear interpolation of opacity $\alpha$ across polygons is not mathematically equivalent to the exponential integration, causing banding artifacts [S16, book][S20, unofficial].
*   **Edge Artifacts:** Interpolation weights become zero at edges, potentially causing loss of classification for entering intersections and erroneous zero-thickness lookups [S17].
*   **GPU Constraints:** Vertex shaders lack flow control and dynamic vertex creation. All projection classes must be handled by the same code path, leading to redundant calculations (e.g., intersection tests for Class 1) [S13, unofficial][S15].

## Subtopics

### [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md)
A cell-projection technique that renders unstructured tetrahedral meshes by projecting individual tetrahedra onto the screen and compositing them in depth order. It solves the problem of visualizing complex, non-convex, or disconnected unstructured data with high geometric fidelity. Unlike PT, which generates triangles for hardware rasterization, this method often employs a two-pass GPGPU pipeline or differentiable rasterization to compute ray-tetrahedron intersections explicitly.

### [Two-Pass GPU Approach](1339-two-pass-gpu-approach.md)
An efficient ray-casting algorithm for tetrahedral meshes implemented on consumer graphics hardware using fragment programs, pre-integration, and multiple rendering passes. It solves the problem of rendering large unstructured tetrahedral volumes interactively by offloading geometric projection and classification logic to the GPU. The method relies on a strict two-stage pipeline: a first pass that computes per-tetrahedron projection data and classification indices in a fragment shader, and a second pass that rasterizes these projections as triangle fans.

### [Cell Projection of Tetrahedral Meshes (Data Transfer Optimization)](1499-cell-projection-of-tetrahedral-meshes-data-transfer-optimiza.md)
This method addresses the data transfer bottleneck between the CPU and the graphics board in unstructured volume visualization by employing a view-independent description of the mesh. It builds a fixed set of data in a preprocessing step that is reused every frame. Unlike PT, which produces a continuous stream of triangles, this approach pre-defines the geometry to optimize bandwidth usage through OpenGL mechanisms.

### [Alternating Tetrahedral Decomposition](3183-alternating-tetrahedral-decomposition.md)
A method for converting a structured hexahedral grid into a tetrahedral mesh by partitioning each cubic cell into five tetrahedra. It solves the topological consistency problem inherent in decomposing cubes, where using a single decomposition pattern for all cells creates T-junctions and discontinuities at shared faces. It sets itself apart by ensuring adjacent cells share common faces, maintaining manifold topology.

### [Hardware-Assisted Projection](5625-hardware-assisted-projection.md)
A family of volume rendering techniques that project 3D geometric primitives onto the 2D screen space to leverage GPU rasterization pipelines. It solves the performance bottleneck of explicit isosurface extraction. Unlike PT, which requires explicit triangle generation and sorting, this family generally relies on hardware compositing, global sorting, or commutative accumulation to handle depth ordering.

### [Sorting-free Projected Tetrahedra](7393-sorting-free-projected-tetrahedra.md)
A volume rendering technique for irregular tetrahedral datasets that approximates back-to-front accumulation using stochastic color composition, thereby eliminating the need for explicit visibility sorting. Unlike PT, which requires sorting triangles before rendering, SPT avoids sorting errors and extra buffer space by treating brightness as an expected value of luminosity over multiple stochastic repetitions.

### [GPU-based Projected Tetrahedra](7484-gpu-based-projected-tetrahedra.md)
A direct volume rendering method for unstructured tetrahedral data that leverages programmable graphics hardware to achieve interactive frame rates. It extends the standard PT algorithm by shifting the geometric classification and decomposition of volume cells into vertex shaders. Unlike traditional ray casting, it avoids explicit per-pixel ray-tetrahedron intersection tests by rasterizing projected triangles and using hardware blending.

### [Texture-Encoded Isosurfacing](7682-texture-encoded-isosurfacing.md)
A GPU-based technique for extracting isosurfaces from unstructured tetrahedral grids by using 2D texturing hardware to compute intersections. It sets itself apart from PT by using the decomposition step to create smaller tetrahedra instead of triangles, then projecting their front and back faces to leverage a checkerboard texture for intersection computation.

### Hybrid Screen-Space Projection Ray Casting
A method for irregular grids that projects volumetric primitives (tetrahedra) onto screen space to determine visibility order, then casts rays through the resulting 2D triangle list to fetch scalar values [S1, unofficial]. It decomposes each tetrahedron into faces, projects them to screen, and casts rays to generate a depth-sorted list of intersections. Rendering time is 11 seconds for a $256^2$ viewport and up to 100 seconds for $1024^2$ on an SGI Power Challenge, with a sorting step of ~2 seconds per viewpoint change [S1, unofficial]. The list of triangles per pixel must be updated whenever the viewpoint changes [S1, unofficial].

### CellFast
A software algorithm for interactive unstructured volume rendering that moves computations for order-independent transparency and volume scan conversion from CPU to graphics hardware, eliminating the need for a software sorting pass [S20, unofficial]. It preprocesses the dataset, visibility sorts cells, tests plane equations to determine class (1-4), projects and splits unique front/back faces, computes color/opacity for the thick vertex, and scans new triangles in hardware. In the original implementation, Steps II-V saturate the host CPU, leaving the graphics hardware underutilized. An average of 3.4 triangles are written per tetrahedron during splitting [S20, unofficial].

### Min-Volume Ellipsoid Approximation
A technique to approximate a tetrahedron's projection by finding the enclosing ellipsoid with the smallest volume, used to simplify constrained minimization over integrals with no closed form [S22]. It applies the transformation $T$ associated with the tetrahedron to a unit sphere to get the min-volume ellipsoid. The volume scaling factor is $\det(T)$, which preserves extrema. Sub-pixel tetrahedra can be represented as unshaped squares for speedups [S22].

### Projected Tetrahedra with MPVONC
The most popular technique for rendering unstructured tetrahedral meshes, combining Projected Tetrahedra with the MPVO with Non-Convexities sorting algorithm [S11, unofficial]. MPVONC sorting is not exact, leading to visual artifacts, but is fast and "accurate enough" for most applications. PT has accuracy and implementation issues leading to incorrect images, particularly regarding perspective projection and compositing accuracy on existing GPUs. Most existing codes do not generate strictly correct images and are often limited to parallel projections [S11, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear in the number of tetrahedra. Does not depend on the number of isosurfaces or employed transfer functions [S13, unofficial].
*   **Memory Complexity:** Substantially less than ray casting approaches because no topology information storage is required [S9, unofficial]. Memory usage is approx. 20 bytes per tetrahedron (assuming 4 tets per vertex) for texture-based implementations [S8]. Minimum texture resolution is $128^3$, consuming 4 MB of memory for a luminance alpha texture map [S17].
*   **Measured Performance:** Achieves interactive rates (up to 2.0 M Tets/s) on current graphics hardware (e.g., nVidia GeForce4) [S7, S8]. Rendering times were several seconds for small datasets in early software implementations [S1, unofficial].
*   **Strengths:** Efficient when one cell projects to many pixels; utilizes graphics hardware for Gouraud shading; no topology information storage required; supports interactive transfer function editing with pre-integration [S2, unofficial][S7, S8].
*   **Limitations:** Requires visibility sorting, which is a bottleneck and not always possible (cycles of overlapping tetrahedra) [S6, book][S7]. Sorting errors can cause visual artifacts. Linear interpolation of opacity causes Mach banding. GPU implementations are constrained by instruction limits (128 instructions) and lack of branching, leading to redundant calculations [S13, unofficial][S15][S16, book].
*   **When to Use:** Interactive visualization of unstructured tetrahedral meshes where performance is critical and approximate sorting is acceptable.
*   **When Not to Use:** When exact visibility ordering is required and sorting is too expensive; when sub-pixel accuracy is critical without stochastic methods; when data is highly structured and voxel-based methods are more efficient.

## Background
Projected Tetrahedra was proposed by Shirley and Tuchman in 1990 [S1, S2, unofficial][S7]. It was one of the first algorithms for projecting cells of unstructured grids, designed to handle irregular grids (e.g., CFD data) by tessellating volumetric geometry into tetrahedra [S5]. The method was developed to leverage the emerging capabilities of graphics hardware for Gouraud shading and scan conversion [S2, unofficial]. It has since been extended to GPU-based implementations using vertex and fragment shaders, and combined with various sorting algorithms like MPVO [S7, S8][S11, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Unstructured Volume Rendering Unstructured Volume Rendering. Grid Types. Structured Grids:. uniform. regular. rectilinear. curvilinear. Unstructured Grids:. regular. irregular. hybrid. curv, https://www.slideserve.com/tmauriello/unstructured-volume-rendering-powerpoint-ppt-presentation, fetched 2026-10-25 (unofficial)
[S3] An Overview of Rendering from Volume Data --- including Surface and Volume Rendering, https://core.ac.uk/download/63614.pdf, fetched 2026-10-30 (unofficial)
[S4] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
[S5] Chapter 7. Irregular Grids: The Projected Tetrahedra Render ..., https://techpubs.jurassic.nl/library/manuals/4000/007-4389-010/sgi_html/ch07.html, fetched 2026-10-27
[S6] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
[S7] PDF Volume and Isosurface Rendering with GPU-Accelerated Cell Projection*, https://graphics.tudelft.nl/~marroquim/publications/pdfs/marroquim-CGF2008.pdf, fetched 2026-10-27
[S8] PDF GPU-Based Cell Projection for Interactive Volume Rendering, https://graphics.tudelft.nl/~marroquim/publications/pdfs/marroquim-sibgrapi2006.pdf, fetched 2026-10-27
[S9] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
[S10] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S11] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
[S12] PDF Parallel Unstructured Volume Rendering in ParaView, https://www.kennethmoreland.com/documents/ParallelVolRen.pdf, fetched 2026-10-27
[S13] (PDF) Tetrahedral projection using vertex shaders - ResearchGate, https://www.researchgate.net/publication/221247894_Tetrahedral_projection_using_vertex_shaders, fetched 2026-10-28 (unofficial)
[S14] Projected Tetrahedra - DocsLib, https://docslib.org/doc/12619673/projected-tetrahedra, fetched 2026-10-27 (unofficial)
[S15] Tetrahedral Projection using Vertex Shaders, https://www.kennethmoreland.com/gator/volvis2002.pdf, fetched 2026-10-28
[S16] Cell projection of meshes with non-planar faces, https://digital.library.unt.edu/ark:/67531/metadc1408947/?q=projected tetrahedra, fetched 2026-10-28 (book)
[S17] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S18] (PDF) Image-space visibility ordering for cell projection volume ..., https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S19] PDF Hardware-Assisted Visibility Sorting for Unstructured Volume Rendering, https://www.sci.utah.edu/~stevec/papers/havs.pdf, fetched 2026-10-29
[S20] (PDF) An Architecture For Interactive Tetrahedral Volume ..., https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)
[S21] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S22] PDF Hardware-Assisted Point-Based Volume Rendering of Tetrahedral Meshes, https://www.sci.utah.edu/~eranders/papers/pbvr.pdf, fetched 2026-10-28
