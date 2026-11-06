# Garrity Ray Casting

Also called: Garrity, Garrity's Ray Casting

## Overview
Garrity Ray Casting is a method for rendering unstructured meshes that exploits intrinsic mesh connectivity to optimize ray traversal by tracking entrance and exit points of a ray from cell to cell [S1, S2, unofficial]. It distinguishes itself from sibling methods like Bunyk et al.'s approach by using a hierarchical spatial data structure to store boundary cells, thereby avoiding expensive search operations during re-entry when rays traverse non-convex geometries [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An unstructured mesh (typically a finite element mesh) defined by nodes and cells (e.g., tetrahedra, hexahedra); a set of rays defined by origin and direction; optical properties (opacity, color) associated with cells.
*   **Output:** A 2D image where each pixel contains the accumulated color and opacity along the ray path through the mesh.

**Data Structures:**
1.  **Mesh Connectivity:** The fundamental structure storing which nodes form each cell and which cells share nodes/edges/faces. This allows traversal from one cell to an adjacent cell via shared faces [S1, S2, unofficial].
2.  **Hierarchical Spatial Data Structure (Boundary Index):** A spatial index (e.g., a bounding volume hierarchy or octree) specifically built to store the **boundary cells** of the mesh. This structure maps spatial regions to the set of boundary cells intersecting that region [S1, S2, unofficial].

**Algorithm Steps:**
1.  **Preprocessing:**
    *   Identify all boundary cells of the unstructured mesh (cells that have at least one face not shared with another cell in the domain).
    *   Build the hierarchical spatial data structure using the bounding volumes of these boundary cells. This allows for efficient spatial queries to find which boundary cells are near a given point or ray segment [S1, S2, unofficial].

2.  **Ray Traversal (Per Ray):**
    *   **Initialization:** Determine the initial cell containing the ray origin. If the origin is outside the mesh, use the hierarchical spatial data structure to find the nearest boundary cell and compute the first intersection point (entrance) [S1, S2, unofficial].
    *   **Traversal Loop:**
        *   While the ray has not exited the mesh or reached the end of the volume:
            *   **Inside Cell:** Compute the intersection of the ray with the current cell's faces to find the exit point.
            *   **Accumulation:** Integrate the optical properties (color/opacity) along the segment from the previous point to the current exit point.
            *   **Exit Check:** Determine if the ray has exited the mesh domain.
                *   **If exiting to another cell:** Use the mesh connectivity to identify the adjacent cell sharing the exit face. Update the current cell to this neighbor. Continue traversal.
                *   **If exiting the mesh domain (to empty space):** The ray has left the volume. Use the hierarchical spatial data structure to search for the next boundary cell that the ray might re-enter. Compute the intersection with this new boundary cell to find the re-entry point. If no re-entry is found, terminate the ray.
    *   **Termination:** Stop when the ray exits the mesh and no further re-entry is detected, or when the accumulated opacity reaches a threshold (if using early ray termination).

**Formulas:**
*   **Ray-Cell Intersection:** For a cell defined by vertices $\{v_1, ..., v_k\}$, the intersection of ray $R(t) = O + tD$ with a face defined by vertices $\{v_i, v_j, v_k\}$ is found by solving for $t$ where the ray intersects the plane of the face, then checking if the intersection point lies within the triangle [S1, S2, unofficial].
*   **Accumulation:** $C_{out} = C_{in} + (1 - \alpha_{in}) \cdot c \cdot (1 - e^{-\sigma \Delta t})$, $\alpha_{out} = \alpha_{in} + (1 - \alpha_{in}) \cdot (1 - e^{-\sigma \Delta t})$, where $c$ is color, $\sigma$ is extinction coefficient, and $\Delta t$ is the path length through the cell segment [S1, S2, unofficial].

**Parameters:**
*   **Spatial Structure Type:** Not specified in sources. Common choices include BVH or Octree [S1, S2, unofficial].
*   **Re-entry Search Strategy:** The hierarchical structure is queried to find candidate boundary cells. The specific query algorithm (e.g., nearest neighbor, ray casting against BVH) is not detailed [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Convexity:** Rays may enter and exit the mesh multiple times. The algorithm must correctly handle re-entries by querying the boundary structure [S1, S2, unofficial].
*   **Precision:** Intersection calculations must be robust to avoid gaps or overlaps between cells, especially at shared faces [S1, S2, unofficial].
*   **Boundary Detection:** Correctly identifying boundary cells is crucial for building the spatial index [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def garrity_ray_casting(mesh, rays, optical_properties):
    # Preprocessing
    boundary_cells = identify_boundary_cells(mesh)
    boundary_index = build_hierarchical_spatial_structure(boundary_cells)
    
    image = create_empty_image(rays)
    
    for ray in rays:
        # Find initial cell
        if ray.origin inside mesh:
            current_cell = find_cell_containing_point(mesh, ray.origin)
            t_start = 0.0
        else:
            # Use boundary index to find first entry
            candidate_cells = boundary_index.query_near(ray.origin)
            current_cell, t_start = find_first_intersection(mesh, ray, candidate_cells)
            if current_cell is None:
                continue # Ray misses mesh entirely
        
        t_current = t_start
        accumulated_color = (0, 0, 0)
        accumulated_opacity = 0.0
        
        while True:
            # Find exit point from current cell
            exit_point, t_exit, exit_face = find_ray_cell_intersection(ray, current_cell, t_current)
            
            if exit_point is None:
                # Ray stays within cell indefinitely (should not happen in closed mesh)
                break
            
            # Calculate path length
            delta_t = t_exit - t_current
            
            # Accumulate color and opacity
            cell_props = optical_properties[current_cell]
            transmittance = exp(-cell_props.extinction * delta_t)
            color_contribution = cell_props.color * (1 - transmittance)
            
            accumulated_color = accumulated_color + (1 - accumulated_opacity) * color_contribution
            accumulated_opacity = accumulated_opacity + (1 - accumulated_opacity) * (1 - transmittance)
            
            if accumulated_opacity >= 1.0:
                break # Early ray termination
            
            # Check if exiting mesh
            if exit_face is boundary_face:
                # Ray exits the mesh domain
                # Use boundary index to find next re-entry
                next_boundary_cells = boundary_index.query_ray_segment(ray, t_exit, infinity)
                
                next_entry_cell, t_next_entry = find_first_intersection(mesh, ray, next_boundary_cells, t_exit)
                
                if next_entry_cell is None:
                    break # No more re-entries
                
                current_cell = next_entry_cell
                t_current = t_next_entry
            else:
                # Ray exits to adjacent cell
                current_cell = mesh.get_adjacent_cell(current_cell, exit_face)
                t_current = t_exit
        
        # Write to image
        write_pixel(image, ray.pixel_index, accumulated_color, accumulated_opacity)
    
    return image
```

## Subtopics

### Bunyk et al. Ray Casting
A method that handles non-convexities by precomputing boundary face fragments per pixel and storing them in front-to-back order, avoiding the need for a hierarchical spatial structure for re-entry [S1, S2, unofficial]. Choose this when memory for the spatial index is constrained or when pixel-based coherence can be exploited more effectively. See [Bunyk Ray Casting](link).

## Operating characteristics
*   **Time Complexity:** Traversal within convex regions is efficient due to connectivity. Re-entry detection depends on the cost of querying the hierarchical spatial structure, which is typically $O(\log N)$ for $N$ boundary cells.
*   **Memory Complexity:** Requires storage for the mesh connectivity and the hierarchical spatial index of boundary cells. The size of the index depends on the number of boundary cells and the structure's overhead.
*   **Strengths:** Efficiently handles non-convex meshes by leveraging connectivity for internal traversal and spatial indexing for re-entry. Avoids expensive global searches.
*   **Limitations:** Requires preprocessing to build the boundary index. Performance may degrade if the mesh has a very high surface-to-volume ratio (many boundary cells).
*   **When to Use:** Suitable for rendering unstructured meshes with complex, non-convex geometries where connectivity is available and re-entries are frequent.
*   **When Not to Use:** If the mesh is convex, simpler traversal methods may be faster. If memory is extremely limited, the spatial index overhead may be prohibitive.

## Background
Garrity [28] pioneered the use of ray casting for rendering unstructured meshes. The technique was developed to address the challenges of traversing irregular grids where standard grid-based traversal methods are not applicable. It is cited in surveys of GPU-based volume rendering of unstructured grids [S1, S2, unofficial].

## Sources
[S1] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
