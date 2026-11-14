# Interactive RBF Reconstruction via Volume Slicing

Also called: On-the-fly RBF Reconstruction, Slice-based RBF Rendering

## Overview
This method visualizes volumetric scalar data encoded as a weighted sum of Radial Basis Functions (RBFs) by eliminating the need for 3D texture storage. Instead, it uses the GPU fragment processor to perform on-the-fly reconstruction of the RBF-encoded data during the rasterization of 2D slices [S1]. It sets itself apart from standard texture-based volume slicing by computing field values procedurally per fragment, allowing the encoding to remain hidden from the rendering pipeline and supporting arbitrary visualization algorithms like cutting planes or isosurfaces [S1].

## How it works
**Inputs:** RBF parameters (centers $\mu_i$, weights $w_i$, radii $\sigma_i$) stored in 2D floating-point textures; an adaptive octree spatial decomposition mapping cells to lists of relevant RBF centers; viewing parameters (slice orientation, spacing).
**Outputs:** A composited 2D image representing the volume.

**Data Structures:**
1.  **RBF Parameter Textures:** Two 2D floating-point textures store the RBF parameters. Texture 0 and Texture 1 hold the parameters required for evaluation [S1].
2.  **Adaptive Octree:** A non-uniform spatial subdivision based on basis function center locations. Each cell stores a list of contributing basis functions, determined by checking if the basis function's radius of influence $r_i$ intersects the cell [S1].
3.  **Active Cell List:** A list of cells intersected by the current slice, used to minimize traversal costs [S1].
4.  **Intersected Cell List:** A copy of the active cell list, used to track cells requiring a final rendering pass [S1].
5.  **P-Buffers:** Two hardware-accelerated floating-point p-buffers used for intermediate results [S1].

**Rendering Algorithm:**
The method adopts a texture-based volume slicing approach [S1].
1.  **Slice Generation:** Compute slice polygons by intersecting a plane with the bounding box of the desired volume domain. Place a set of slices oriented orthogonal to the viewing direction equidistantly within the volume domain [S1].
2.  **Traversal Order:** The outer loop iterates over all slices. For each slice, the algorithm processes the intersected cells [S1].
3.  **Cell-Based Optimization:** If the hierarchy contains only cells renderable in a single pass, the algorithm switches to cell-based traversal: processing all slices for a given cell first before switching to the next cell. In this case, visibility sorting is required [S1].
    *   **Sorting:** A recursive sorting algorithm sorts the eight children at each level based on the distance of their centers from the viewer’s position. The traversal descends in a depth-first manner based on this level-wise ordering [S1].
4.  **Multipass Rendering (Two Phases):** Because a cell may contain more RBFs than the fragment processor can handle in one step, multipass rendering is used [S1].
    *   **Phase I (Partial Evaluation):** The fragment program partially evaluates the RBF sum and writes the intermediate result into one of the p-buffers. The p-buffer is bound to a texture map to serve as input for the next pass [S1].
    *   **Phase II (Final Pass):** The final pass writes directly to the graphics context (framebuffer) [S1].
5.  **Active List Management:**
    *   Initialize `activeCellList` with all cells intersected by the current slice.
    *   Initialize `intersectedCellList` as a copy of `activeCellList` [S1].
    *   While cells remain in `activeCellList`:
        *   Setup rendering pass.
        *   For each cell $c$ in `activeCellList`:
            *   If $c$ contains only one chunk of unrendered RBFs, remove $c$ from `activeCellList`.
            *   Otherwise, render the intersection polygon of the slice clipped to the cell boundaries [S1].
    *   Setup final rendering pass.
    *   For each cell $c$ in `intersectedCellList`, render the intersection polygon [S1].
6.  **Volume Rendering Semantics:** The reconstructed volume data is mapped to the alpha channel of the fragment color. The OpenGL alpha test is used to simulate the first-hit semantics of a volume ray caster [S1].
7.  **Compositing:** Slices are rendered with texture-mapped (procedurally generated) volume data and composited back to front [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_slices(slices, octree):
    for slice in slices:
        # Initialize lists for this slice
        active_cell_list = create_active_cell_list(slice, octree)
        intersected_cell_list = copy(active_cell_list)
        
        # Phase I: Partial evaluation
        while active_cell_list is not empty:
            setup_rendering_pass()
            for cell in active_cell_list:
                # Check if cell has only one chunk of unrendered RBFs left
                if cell.has_only_one_chunk_left():
                    remove_from_active_cells(cell)
                else:
                    # Clip slice at cell boundaries and render
                    render_intersection_polygon(cell, slice)
        
        # Phase II: Final pass to framebuffer
        setup_final_rendering_pass()
        for cell in intersected_cell_list:
            render_intersection_polygon(cell, slice)

def render_intersection_polygon(cell, slice):
    # Clip the slice polygon to the cell's bounding box
    clipped_polygon = clip_polygon_to_cell(slice.polygon, cell.bounding_box)
    if clipped_polygon is empty:
        return
    
    # Bind cell-specific RBF parameters to textures
    bind_rbf_textures(cell.rbf_parameters)
    
    # Render the clipped polygon
    # The fragment program evaluates the RBF sum on-the-fly
    # and writes to the p-buffer (Phase I) or framebuffer (Phase II)
    render_polygon(clipped_polygon)
```

**Edge Cases and Pitfalls:**
*   **Simultaneous Read/Write:** The GeForce FX does not support simultaneous read and write operations on the same buffer, necessitating the use of two p-buffers [S1].
*   **Single-Pass Cells:** If all cells in the hierarchy can be rendered in a single pass, the algorithm must switch to cell-based traversal and perform visibility sorting to ensure correct semi-transparent volume rendering [S1].

## Operating characteristics
**Time Complexity:** The complexity is driven by the number of slices, the number of intersected cells per slice, and the number of RBFs per cell. The active cell list optimization minimizes traversal costs by removing cells once they require only one additional pass [S1].
**Memory Complexity:** RBF parameters are stored in 2D floating-point textures in local GPU memory, avoiding CPU-to-GPU transfer bottlenecks. The total amount of RBF data is small enough to reside in local memory [S1].
**Strengths:** Eliminates the need for 3D texture storage; extends to various visualization algorithms (cutting planes, isosurfaces); reduces the number of basis functions evaluated per fragment via spatial decomposition [S1].
**Limitations:** Requires multipass rendering if a cell contains more RBFs than the fragment processor can handle in one step; hardware limitations (e.g., GeForce FX) may require two p-buffers [S1].

## Background
This method was introduced by Jang, Weiler, Hopf, Huang, Ebert, Gaither, and Ertl in 2004 [S1]. It builds on the RBF Volume Encoding and Adaptive Octree Spatial Decomposition method [14683-rbf-volume-encoding-and-adaptive-octree-spatial-decompositio.md] and the RBF Encoding of Volume Data method [14685-rbf-encoding-of-volume-data.md].

## Sources
[S1] Interactively Visualizing Procedurally Encoded Scalar Fields, https://doi.org/10.2312/vissym/vissym04/035-044, fetched 2026-10-26
