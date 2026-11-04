# ZSWEEP

Also called: ZSweep, ZSweep Algorithm

## Overview
ZSWEEP is an exact, object-order cell projection algorithm for volume rendering on unstructured grids that optimizes for sorting efficiency and memory usage [S1][S3, S2, unofficial]. It solves the problem of correctly ordering cell segments for front-to-back compositing without the high memory overhead of ray-casting or the complexity of true global visibility sorts [S2, unofficial]. It sets itself apart from its parent [A-buffer visibility sort for non-convex grids](1057-a-buffer-visibility-sort-for-non-convex-grids.md) by using a sweep-plane approach that exploits the implicit global ordering induced by vertex z-coordinates, rather than maintaining full A-buffer lists for all pixels simultaneously [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An unstructured mesh containing tetrahedral and/or hexahedral cells [S2, unofficial].
*   **Output:** A rendered image with correct front-to-back compositing of cell contributions [S2, unofficial].

**Data Structures:**
*   **Sweep Plane State:** A set of active cells and faces currently intersected by the sweep plane.
*   **Pixel Lists:** For each pixel, a short list of ordered face intersections. This list is maintained to be correct and complete at the instant each stage of computation is completed [S2, unofficial].
*   **Auxiliary Structures:** Stores partial information taken from a small number of "slices" of the dataset, rather than the entire dataset in memory [S2, unofficial].

**Steps:**
1.  **Sort Vertices:** Order all vertices of the mesh by their z-coordinate (depth) relative to the viewing plane [S2, unofficial].
2.  **Initialize Sweep:** Initialize the sweep plane at the minimum z-coordinate of the mesh [S2, unofficial].
3.  **Sweep and Project:**
    *   Advance the sweep plane to the next vertex in the sorted z-order [S2, unofficial].
    *   Identify all cells incident to the current vertex [S2, unofficial].
    *   Project the faces of these cells onto the image plane [S2, unofficial].
    *   **Avoid Double Projection:** Apply logic to ensure internal faces are not projected twice (once for each adjacent cell) [S2, unofficial].
    *   **Update Pixel Lists:** Update the ordered face intersection lists for the affected pixels. The z-ordering of vertices induces an approximate global ordering on the cells, which helps maintain the correct front-to-back order in the pixel lists [S2, unofficial].
4.  **Compute Contribution:**
    *   For each pixel, use the current short list of ordered face intersections to compute the color and opacity contribution [S2, unofficial].
    *   Ensure the list is correct and complete before computing the contribution for that stage [S2, unofficial].
    *   Composite the contribution into the frame buffer using standard front-to-back alpha compositing equations [S2, unofficial].
5.  **Sparsification (Optional):** Apply a data sparsification technique to reduce the number of cells processed, if applicable [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zsweep(mesh, view_direction):
    # 1. Sort vertices by z-coordinate
    vertices = mesh.get_vertices()
    sorted_vertices = sort_by_z(vertices, view_direction)
    
    # 2. Initialize data structures
    # pixel_lists: dict mapping pixel_id -> list of (face, z_depth)
    pixel_lists = {}
    # active_cells: set of cells currently intersected by sweep plane
    active_cells = set()
    
    # 3. Sweep through vertices
    for vertex in sorted_vertices:
        # Get cells incident to this vertex
        incident_cells = mesh.get_incident_cells(vertex)
        
        # Update active cells
        for cell in incident_cells:
            if cell not in active_cells:
                active_cells.add(cell)
                # Project faces of the cell
                faces = cell.get_faces()
                for face in faces:
                    # Check if face is internal to avoid double projection
                    if not is_internal_face(face, active_cells):
                        projected_face = project_face(face, view_direction)
                        # Update pixel lists for the pixels covered by this face
                        update_pixel_lists(pixel_lists, projected_face, vertex.z)
        
        # 4. Compute contributions for affected pixels
        affected_pixels = get_affected_pixels(active_cells)
        for pixel_id in affected_pixels:
            if pixel_id in pixel_lists:
                # Ensure list is sorted and complete
                sort_pixel_list(pixel_lists[pixel_id])
                # Compute and composite contribution
                composite_pixel(pixel_id, pixel_lists[pixel_id])
                
    return frame_buffer

def is_internal_face(face, active_cells):
    # A face is internal if it is shared by two cells in the active set
    # This is a simplified check; actual implementation requires adjacency info
    neighbors = face.get_neighbors()
    in_active = [n for n in neighbors if n in active_cells]
    return len(in_active) > 1

def update_pixel_lists(pixel_lists, projected_face, z_depth):
    for pixel_id in projected_face.get_covered_pixels():
        if pixel_id not in pixel_lists:
            pixel_lists[pixel_id] = []
        # Insert face into list maintaining z-order
        insert_in_order(pixel_lists[pixel_id], (projected_face, z_depth))
```

**Parameters:**
*   **Cell Types:** Handles tetrahedral and/or hexahedral cells [S2, unofficial].
*   **Sparsification:** A simple technique may be applied to reduce dataset size [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Double Projection:** Internal faces must be carefully handled to avoid double counting [S2, unofficial].
*   **Ordering Correctness:** The implicit ordering induced by vertex z-coordinates is approximate; care must be taken to assure correctness in the projection order [S2, unofficial].
*   **Hardware Mapping:** There is no simple way to map this algorithm into currently available graphics hardware [S1][S3, unofficial].

## Subtopics

### [Out-of-Core ZSWEEP](2134-out-of-core-zsweep.md)
Out-of-Core ZSWEEP is a direct volume rendering technique for large unstructured grids that extends the in-core ZSWEEP algorithm to handle datasets exceeding available RAM by processing data in tiles and loading meta-cells from disk only when needed. It differs from its sibling [MIR](1281-mir.md) by being substantially faster and leveraging the ZSWEEP optimization of front-to-back sorting, at the cost of more complex preprocessing and higher main-memory requirements.

## Operating characteristics
*   **Time Complexity:** Up to 5 times faster than the best previously-known exact algorithms that use comparable amounts of memory [S2, unofficial].
*   **Memory Complexity:** Memory efficient, using much less memory than ray-casting [S2, unofficial]. Auxiliary structures store only partial information from a small number of dataset slices [S2, unofficial].
*   **Strengths:** Simple, fast, and memory-efficient for exact rendering of unstructured datasets [S2, unofficial]. Adaptable to general non-tetrahedral cell formats [S2, unofficial].
*   **Limitations:** No simple way to map into currently available graphics hardware [S1][S3, unofficial]. Requires careful handling of internal faces and ordering correctness [S2, unofficial].
*   **When to Use:** When exact rendering of unstructured grids is required, and memory is a constraint, and hardware acceleration is not a primary requirement [S2, unofficial].
*   **When Not to Use:** When hardware acceleration is needed or when approximate rendering is acceptable [S1][S3, unofficial].

## Background
ZSWEEP was introduced by Ricardo Farias, Joseph S. B. Mitchell, and Claudio Silva in 2000 [S2, unofficial]. It is an A-Buffer technique that optimizes for sorting and memory usage [S1][S3, unofficial]. The algorithm is hardware-independent and handles datasets containing tetrahedral and/or hexahedral cells [S2, unofficial].

## Sources
[S1] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://doi.org/10.1109/tvcg.2004.45, fetched 2026-10-25
[S2] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
[S3] PDF 0078-0903-2 1..13, https://sci.utah.edu/~csilva/papers/tvcg2004cr.pdf, fetched 2026-10-28 (unofficial)
[S4] Hierarchical and parallelizable direct volume rendering for irregular and multiple grids, https://doi.org/10.1109/visual.1996.567606, fetched 2026-10-24
