# GPU-Based Projected Tetrahedra

Also called: GPU Volume Scan Conversion of Tetrahedra, Tetrahedral Projection using Vertex Shaders, Texture Memory PT, Two-Shader PT, Vertex Shader PT

## Overview
GPU-Based Projected Tetrahedra is a method for interactive volume rendering of unstructured tetrahedral data that leverages programmable graphics hardware to perform volume scan conversion directly on the GPU [S1, unofficial]. It solves the performance bottleneck of traditional CPU-based projected tetrahedra by avoiding redundant projections and pre-processing, achieving interactive rates with a substantially smaller memory footprint than ray-casting approaches [S1, unofficial]. It differs from its siblings by utilizing a specific two-shader architecture (or vertex shader-only variants) to handle projection and rasterization, whereas [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md) focuses on sorting algorithms and [Two-Pass GPU Approach](1339-two-pass-gpu-approach.md) separates projection and rendering into distinct passes [S1, unofficial].

## How it works
The method implements the Projected Tetrahedra (PT) algorithm [Parent](877-projected-tetrahedra.md) using the graphics pipeline. It requires no pre-processing of the data and no additional data structures beyond those inherent to the GPU pipeline [S1, unofficial].

### Inputs and Outputs
*   **Input:** Unstructured tetrahedral mesh (vertices and connectivity), volume data (scalar values at vertices), camera parameters (view and projection matrices).
*   **Output:** Rendered image (color and opacity) on the frame buffer.

### Data Structures
*   **Model Storage:** The tetrahedral mesh and associated volume data are stored in texture memory (e.g., 3D textures or 2D texture arrays) to allow efficient access by shaders [S1, unofficial].
*   **Vertex Buffer:** Contains the four vertices of each tetrahedron. In variants using strips, a buffer holds the four vertices of the current tetrahedron and the additional fifth vertex created by the PT algorithm [S3, unofficial].

### Steps
1.  **Data Loading:** Load the tetrahedral mesh and volume data into GPU texture memory [S1, unofficial].
2.  **Projection:** Use programmable vertex shaders to project the 3D tetrahedra onto the 2D screen space [S1, unofficial]. This step computes the screen-space coordinates of the tetrahedron vertices.
3.  **Rasterization:** The projected tetrahedra are converted into triangles (scan conversion) by the GPU hardware [S1, unofficial].
4.  **Rendering:** Fragment shaders compute the final color and opacity for each pixel covered by the projected triangles, performing volume integration [S1, unofficial].

### Formulas
The specific mathematical formulas for the PT algorithm (e.g., ray-tetrahedron intersection, opacity integration) are defined in [Projected Tetrahedra](877-projected-tetrahedra.md). The GPU-based implementation relies on these geometric primitives but offloads the computation to the vertex and fragment shaders.

### Parameters
*   **Texture Memory Size:** Must be sufficient to hold the entire model.
*   **Shader Programs:** Two fragment shaders are employed in the Marroquim et al. approach: one for computing tetrahedra projections and another for rendering the elements [S1, unofficial].

### Edge Cases and Pitfalls
*   **Redundant Projections:** Implementations using vertex shaders must avoid redundant projections of tetrahedral elements to maintain performance [S1, unofficial].
*   **Square Root Computation:** Computing edge lengths and face areas requires square roots, which impacts running time and complexity [S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for GPU-Based Projected Tetrahedra
# Based on [S1, unofficial] and [S3, unofficial]

def gpu_based_projected_tetrahedra(tetrahedra, volume_data, view_matrix, proj_matrix):
    """
    Renders unstructured tetrahedral data using GPU shaders.
    
    Args:
        tetrahedra: List of tetrahedra, each defined by 4 vertex indices.
        volume_data: Scalar values at each vertex.
        view_matrix: 4x4 view matrix.
        proj_matrix: 4x4 projection matrix.
    """
    
    # Step 1: Load data into texture memory
    # Store model in texture memory to avoid redundant projections [S1, unofficial]
    texture_memory = load_to_gpu_texture(tetrahedra, volume_data)
    
    # Step 2: Project tetrahedra using vertex shaders
    # Use programmable vertex shaders to project tetrahedral elements [S1, unofficial]
    # Avoid redundant projections [S1, unofficial]
    projected_triangles = []
    for tet in tetrahedra:
        # Get 3D vertices
        v0, v1, v2, v3 = get_vertices(tet, volume_data)
        
        # Project vertices to screen space
        # This is handled by the vertex shader in hardware
        screen_v0 = project_vertex(v0, view_matrix, proj_matrix)
        screen_v1 = project_vertex(v1, view_matrix, proj_matrix)
        screen_v2 = project_vertex(v2, view_matrix, proj_matrix)
        screen_v3 = project_vertex(v3, view_matrix, proj_matrix)
        
        # Generate triangles for scan conversion
        # The PT algorithm determines which triangles to generate based on view direction
        # This logic is typically embedded in the vertex shader or pre-computed
        triangles = generate_pt_triangles(screen_v0, screen_v1, screen_v2, screen_v3)
        projected_triangles.extend(triangles)
        
    # Step 3: Rasterize and render using fragment shaders
    # Employ two fragment shaders: one for computing projections and another for rendering [S1, unofficial]
    # Perform volume scan conversion within the GPU [S1, unofficial]
    render_triangles(projected_triangles, texture_memory)

def project_vertex(vertex, view_matrix, proj_matrix):
    """Project a 3D vertex to 2D screen space."""
    # Homogeneous coordinates
    clip_coords = proj_matrix @ view_matrix @ [vertex.x, vertex.y, vertex.z, 1.0]
    # Perspective division
    if clip_coords.w != 0:
        ndc_x = clip_coords.x / clip_coords.w
        ndc_y = clip_coords.y / clip_coords.w
        return [ndc_x, ndc_y]
    else:
        return None

def generate_pt_triangles(v0, v1, v2, v3):
    """
    Generate triangles for a projected tetrahedron.
    The specific logic for determining which triangles to generate
    is defined in the PT algorithm [Parent](877-projected-tetrahedra.md).
    """
    # Placeholder for PT triangle generation logic
    # This depends on the view direction and the classification of the tetrahedron
    return []

def render_triangles(triangles, texture_memory):
    """
    Render triangles using fragment shaders.
    """
    # Fragment shader computes color and opacity based on volume data
    # and blends into the frame buffer
    pass
```

## Subtopics

### Cell Projection with 3D/2D Textures
This variant stores color and opacity in a 3D texture map computed during pre-processing, enabling concurrent use of transfer functions encoded in the texture map [S2, unofficial]. It assigns all associated triangles to each vertex of each projected tetrahedron and blends them into the frame buffer using back-to-front compositing [S2, unofficial]. To reduce memory consumption, it can be adapted for 2D texture maps by approximating dependencies of the volume rendering integral with respect to the length between sample points in linear terms and storing them in the first 2D texture map, while storing remaining parts of the integral (dependent only on sample values) in another 2D texture map [S2, unofficial].

### Hardware Caching for Projected Tetrahedra
This variant maintains a buffer holding the four vertices of the current tetrahedron and the additional fifth vertex sometimes created by the PT algorithm [S3, unofficial]. It swaps the vertices in the buffer as necessary to follow the direction of the strip [S3, unofficial]. It caches edge lengths and face areas computed for the current tetrahedron, as these are required by the PT algorithm and are reused by the next tetrahedron in a strip [S3, unofficial]. A pitfall is that computing edge lengths and face areas requires a square root, which impacts running time and complexity [S3, unofficial].

## Operating characteristics
*   **Performance:** Capable of rendering over 2.0 M Tet/s on current graphics hardware [S1, unofficial].
*   **Memory Footprint:** Occupies a substantially smaller memory footprint compared to ray-casting approaches [S1, unofficial].
*   **Interactivity:** Allows interactive viewing of large unstructured datasets on a desktop personal computer [S1, unofficial].
*   **Strengths:** No pre-processing or additional data structures required [S1, unofficial]. Competitive with recent ray-casting approaches [S1, unofficial].
*   **Limitations:** Performance can be impacted by the computation of square roots for edge lengths and face areas [S3, unofficial].

## Background
The method is based on the Projected Tetrahedra (PT) algorithm by Shirley and Tuchman [S1, unofficial]. The GPU-based implementation was presented by Wylie et al. in 2002, who introduced tetrahedral projection using vertex shaders [S1, unofficial]. Marroquim et al. further refined the approach in 2006 with a practical implementation using two fragment shaders [S1, unofficial]. The technique builds on the foundational work of Kajiya and Von Herzen for ray tracing volume densities [S1, unofficial].

## Sources
[S1] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
[S2] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S3] (PDF) An Architecture For Interactive Tetrahedral Volume ..., https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)
