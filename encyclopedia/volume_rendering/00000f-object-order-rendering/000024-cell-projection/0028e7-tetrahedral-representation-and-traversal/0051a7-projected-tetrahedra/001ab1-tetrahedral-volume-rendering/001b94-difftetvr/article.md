# DiffTetVR

Also called: Differentiable Tetrahedral Volume Rendering

## Overview
DiffTetVR is a differentiable direct volume renderer for tetrahedral meshes that enables optimization of vertex positions, colors, opacities, and local mesh subdivision without multigrid methods [S1, unofficial]. It solves the limitation of previous differentiable volume rendering works restricted to regular grids [S1, unofficial]. Unlike sibling methods that use ray marching or approximate cell sampling, DiffTetVR uses a rasterization-based approach with per-pixel linked lists to efficiently handle unstructured geometry [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh defined by vertex positions, vertex colors, and vertex opacities [S1, unofficial].
*   **Output:** A rendered image and the gradients of the loss function with respect to vertex colors and positions [S1, unofficial].

**Data Structures:**
The implementation relies on several GPU buffers to manage the transition from tetrahedral cells to rasterizable triangles [S2, unofficial]:
*   `TetIndexBuffer`: Indices defining the tetrahedra.
*   `TetVertexPositionBuffer`: 3D positions of mesh vertices.
*   `TetVertexColorBuffer`: RGB colors of mesh vertices.
*   `TriangleVertexPositionBuffer`: Positions of triangle vertices generated from tetrahedra.
*   `TriangleVertexColorBuffer`: Colors of triangle vertices.
*   `TriangleCounterBuffer`: Count of triangles generated per pixel or tile.
*   `TriangleKeyValueBuffer`: Stores triangle IDs and their associated data for sorting.
*   `SortedTriangleKeyValueBuffer`: The triangle list sorted back-to-front.
*   `DrawIndirectCommandBuffer`: Commands for indirect drawing calls.
*   `DispatchIndirectCommandBuffer`: Commands for indirect compute shader dispatches.
*   `UniformDataBuffer`: Camera and rendering parameters.

**Algorithm Steps:**

1.  **Triangle Generation (`GenerateTrianglesPass`):**
    Each tetrahedron is projected onto the image plane [S2, unofficial]. The visible faces of the tetrahedra are extracted and converted into triangles using a compute shader [S2, unofficial]. The workgroup dispatch is indirect, based on the number of tetrahedral cells and a fixed `BLOCK_SIZE` of 256 [S2, unofficial].

2.  **Depth Computation (`ComputeTrianglesDepthPass`):**
    The depth of each generated triangle is computed [S2, unofficial]. This pass uses indirect dispatch via `DispatchIndirectCommandBuffer` to handle variable workloads [S2, unofficial].

3.  **Sorting:**
    Triangles are sorted back-to-front to ensure correct alpha blending [S2, unofficial]. The implementation supports two sorting methods: `FUCHSIA_RADIX_SORT` (GPU-based) or `CPU_STD_SORT` (CPU-based) [S2, unofficial]. The output is stored in `SortedTriangleKeyValueBuffer` [S2, unofficial].

4.  **Rasterization and Rendering (`ProjectedRasterPass`):**
    Triangles are rasterized using indirect draw calls from `DrawIndirectCommandBuffer` [S2, unofficial]. The rendering uses back-to-front premultiplied alpha blending [S2, unofficial].

    *Accumulation Formula:*
    For each pixel, the color $C$ and opacity $\alpha$ are accumulated along the view ray [S1, unofficial]. Unlike simple alpha blending of locally sampled colors, DiffTetVR derives a more accurate accumulation formula for tetrahedral elements [S1, unofficial]. Let $c_0, \alpha_0$ be the color and opacity at the entry face of a tetrahedron, and $c_1, \alpha_1$ be the values at the exit face, obtained via barycentric interpolation [S1, unofficial]. The accumulation follows a derived equation that provides higher accuracy than regular alpha blending [S1, unofficial]. The exact formula by Williams and Max involving the difference of two imaginary error functions has far too high numerical instability to be usable in real-world workflows on GPUs when using only 32-bit floating point precision [S1, unofficial]. A simplified, stable formula must be used instead [S1, unofficial].

5.  **Differentiable Backpropagation:**
    Derivatives for the rendering process are deduced to enable backpropagation regarding parameters such as vertex colors and positions [S1, unofficial]. The gradients are computed using automatic differentiation over the pipeline [S1, unofficial].

6.  **Regularization:**
    A differentiable regularization term is introduced to avoid degeneration of tetrahedral elements during the optimization process [S1, unofficial].

7.  **Local Subdivision (Coarse-to-Fine):**
    The tetrahedral mesh can be locally refined to enable a coarse-to-fine optimization approach [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def diff_tetvr_forward(tet_indices, tet_vertices, tet_colors, tet_opacities, camera_params):
    """
    Forward pass of DiffTetVR.
    Returns rendered image and intermediate buffers for gradient computation.
    """
    # 1. Generate Triangles from Tetrahedra (GenerateTrianglesPass)
    triangles = []
    for tet_idx in tet_indices:
        verts = tet_vertices[tet_idx]
        triangles.extend(project_and_extract_faces(verts, camera_params))
    
    # 2. Compute Triangle Depths (ComputeTrianglesDepthPass)
    triangle_depths = compute_depths(triangles, camera_params)
    
    # 3. Sort Triangles Back-to-Front
    sorted_indices = sort_back_to_front(triangle_depths)
    sorted_triangles = [triangles[i] for i in sorted_indices]
    
    # 4. Rasterize and Blend (ProjectedRasterPass)
    image = rasterize_back_to_front(sorted_triangles, camera_params)
    
    return image

def project_and_extract_faces(verts, camera_params):
    """Project vertices, cull back-facing triangles, and return visible faces."""
    projected = [project_vertex(v, camera_params) for v in verts]
    faces = [[0, 1, 2], [0, 1, 3], [0, 2, 3], [1, 2, 3]]
    visible = []
    for f in faces:
        if is_front_facing(projected[f], camera_params):
            visible.append([projected[f[0]], projected[f[1]], projected[f[2]]])
    return visible

def compute_depths(triangles, camera_params):
    """Compute the depth (z-value) for each triangle."""
    return [compute_triangle_z(t, camera_params) for t in triangles]

def sort_back_to_front(depths):
    """Sort indices by depth in descending order (back-to-front)."""
    return sorted(range(len(depths)), key=lambda i: depths[i], reverse=True)

def rasterize_back_to_front(triangles, camera_params):
    """Rasterize triangles back-to-front with premultiplied alpha blending."""
    color_buffer = zeros_like(camera_params.resolution)
    alpha_buffer = zeros_like(camera_params.resolution)
    for tri in triangles:
        for pixel in rasterize_triangle(tri):
            c, a = interpolate_triangle(tri, pixel)
            color_buffer[pixel] = color_buffer[pixel] * (1 - a) + c * a
            alpha_buffer[pixel] = alpha_buffer[pixel] * (1 - a) + a
    return color_buffer

def compute_gradients(loss, tet_vertices, tet_colors, tet_opacities):
    """
    Compute gradients for the backward pass.
    Sources do not provide the exact analytical gradient implementation details.
    Standard automatic differentiation frameworks compute these over the forward graph.
    """
    # Gradients are deduced for vertex colors and positions [S1, unofficial]
    # Implementation relies on framework autograd [S1, unofficial]
    pass

def compute_loss(rendered_image, target_image):
    """Compute reconstruction loss (e.g., L2 distance)."""
    return mean_squared_error(rendered_image, target_image)

def local_subdivision(tet_indices, tet_vertices, tet_colors, tet_opacities):
    """
    Refine mesh locally for coarse-to-fine optimization.
    Sources do not provide the exact subdivision algorithm details.
    """
    # Local refinement step [S1, unofficial]
    pass
```

**Parameters:**
*   `BLOCK_SIZE`: 256 (for compute shader dispatches) [S2, unofficial].
*   `learning_rate`: Not given in the sources.
*   `num_iterations`: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Numerical Instability:** The exact analytical formula for tetrahedral color accumulation by Williams and Max involves the difference of two imaginary error functions [S1, unofficial]. This formula is numerically unstable in 32-bit floating point precision on GPUs and cannot be used in real-world workflows [S1, unofficial]. A simplified, stable formula must be used instead [S1, unofficial].
*   **Degenerate Tetrahedra:** During optimization, tetrahedra can degenerate (e.g., zero volume) [S1, unofficial]. A differentiable regularization term is required to prevent this [S1, unofficial].
*   **Sorting Stability:** The sorting step uses `FUCHSIA_RADIX_SORT` or `CPU_STD_SORT` to produce the `SortedTriangleKeyValueBuffer` [S2, unofficial].

## Subtopics

### [Viewpoint-dependent subdivision of non-planar faces](835-viewpoint-dependent-subdivision-of-non-planar-faces.md)
Adapts mesh resolution based on camera angle to handle curved surfaces. Choose when geometric accuracy on non-planar boundaries is critical.

### [Approximate Cell Sampling](974-approximate-cell-sampling.md)
Estimates cell contributions via stochastic or deterministic sampling. Choose when performance constraints prevent exact geometric rasterization.

### [Vertex-centered Data Sampling](1037-vertex-centered-data-sampling.md)
Interpolates field values directly from vertex attributes. Choose for smooth scalar/vector fields on unstructured grids.

### [High Accuracy Volume Rendering for Unstructured Data](1076-high-accuracy-volume-rendering-for-unstructured-data.md)
Focuses on precise integration over arbitrary cells. Choose when visual fidelity outweighs computational cost.

### [BT Volume as Reconstruction Filter](1419-bt-volume-as-reconstruction-filter.md)
Applies a band-limited texture volume as a filtering kernel. Choose when aliasing control is required for high-frequency data.

## Operating characteristics
*   **Time Complexity:** Dominated by triangle generation, sorting, and rasterization [S2, unofficial]. Sorting is $O(N \log N)$ where $N$ is the number of triangles [S2, unofficial].
*   **Memory Complexity:** Linear in the number of triangles and pixels [S2, unofficial]. Buffers for triangle data and sorted lists are required [S2, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:**
    *   Enables optimization of vertex positions and local mesh subdivision without multigrid methods [S1, unofficial].
    *   Efficient rasterization-based rendering for unstructured meshes [S1, unofficial].
    *   Provides differentiable gradients for vertex colors and positions [S1, unofficial].
*   **Limitations:**
    *   Numerical instability of exact integration formulas in 32-bit float [S1, unofficial].
    *   Requires careful regularization to prevent mesh degeneration [S1, unofficial].
*   **When to use:** When optimizing unstructured tetrahedral meshes from images is required, especially for coarse-to-fine refinement [S1, unofficial].
*   **When not to use:** When exact integration in 32-bit float is critical or sorting overhead is prohibitive [S1, unofficial].

## Background
DiffTetVR was introduced by Chris Mile et al. in 2026 [S1, unofficial]. It addresses the limitations of previous differentiable volume rendering works that were restricted to regular grids and required multigrid methods for optimization [S1, unofficial]. The method builds on per-pixel linked lists for efficient rasterization of unstructured meshes [S1, unofficial]. The source code is publicly available on GitHub [S2, unofficial].

## Sources
[S1] DiffTetVR: Differentiable Tetrahedral Volume Rendering, https://doi.org/10.48550/arxiv.2601.00114, fetched 2026-10-26 (unofficial)
[S2] GitHub - chrismile/DiffTetVR: Differentiable volume renderer for ..., https://github.com/chrismile/DiffTetVR, fetched 2026-10-28 (unofficial)
