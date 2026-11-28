# Span-Buffer Ray-Casting

Also called: SBRC

## Overview
Span-Buffer Ray-Casting (SBRC) is a hybrid volume rendering algorithm for unstructured grids that adopts an image-then-object traversal order to overcome the inability of prior methods to support early ray termination and to eliminate redundant cell processing [S1]. Unlike Koyamada’s algorithm, which lacks image-space coherency, and Challinger’s or Lazy Sweep Ray Casting (LSRC), which use object-then-image traversal that prevents early ray termination, SBRC activates cells only when hit by a ray, thereby maintaining full utilization of both image and object space coherencies [S1].

## How it works
**Inputs:** A tetrahedral (or general irregular) grid, a camera position, and a screen resolution.
**Outputs:** A rendered image.

**Data Structures:**
1.  **Active Cell List:** Stores entries for cells currently being processed. Each entry includes a pointer to the cell and a `y-bucket` index.
2.  **Y-Bucket List:** An array of size equal to the screen height ($H$). Each bucket contains a list of pointers to entries in the Active Cell List. Used for efficient deletion of cells whose bottom y-coordinate is less than the current scanline.
3.  **Active Edge List:** A hash table mapping edge identifiers to edge data. Stores edges currently intersecting the current scanline.
4.  **Span Buffer:** A data structure associated with each active cell for the current scanline.
    *   **Layout:** An array indexed by pixel column ($x$).
    *   **Fields per pixel:**
        *   $z$: The depth (z-coordinate) of the ray exit point from the cell.
        *   $s$: The accumulated sample value (e.g., opacity/intensity) at the exit point.
        *   $f_{id}$: The identifier of the exit face used to determine the next cell.

**Algorithm Steps:**

1.  **Ray Traversal Initialization:**
    *   Iterate through pixels in image order (scanline by scanline, top to bottom).
    *   For each pixel, shoot a ray into the volume.

2.  **Cell Hit and Activation:**
    *   If the ray hits a cell that is not in the Active Cell List:
        *   **Activate Cell:** Allocate an entry in the Active Cell List.
        *   **Identify Back-Face Edges:** Determine the edges of the cell that belong to at least one back-face (back-face edges). For a tetrahedron, there are 6 edges; depending on the viewpoint, between 2 and 6 edges may be back-face edges [S1].
        *   **Sort Edges:** Sort the back-face edges such that their intersection points with a virtual line parallel to the scanline (constant $y$) appear in increasing order of $x$ coordinates. This sorting is performed once per cell activation [S1].
        *   **Y-Bucket Insertion:** Calculate the bottom $y$-coordinate of the cell. Insert a pointer to the Active Cell List entry into the Y-Bucket List at the index corresponding to this $y$-coordinate [S1].
        *   **Create Span Buffer:** Initialize the Span Buffer for the current scanline.

3.  **Scan Conversion and Span Buffer Update:**
    *   **Edge State Management:** For each back-face edge in the Active Edge List (or newly identified):
        *   Compare the edge's $y$-extent with the current scanline $y$.
        *   Update states:
            *   If edge $y$-extent is above current scanline: State = `done`.
            *   If edge intersects current scanline: State = `active`.
            *   If edge $y$-extent is below current scanline: State = `inactive`.
        *   If an edge transitions from `inactive` to `active`, insert it into the Active Edge List via hashing [S1].
    *   **Compute Intersections:** Using the sorted list of active back-face edges, compute the intersection points of the current scanline with these edges. Let the sorted intersection points be $(x_i, z_i, s_i)$ for $i=0 \dots k$.
    *   **Fill Span Buffer:** For the pixels covered by the cell along the current scanline:
        *   Interpolate $z$ and $s$ values between successive intersection points.
        *   Store the interpolated $(z, s)$ values and the exit-face identifier in the Span Buffer for the respective pixel column [S1].

4.  **Ray Sampling and Propagation:**
    *   **First Hit (New Span):** If the current ray hits the cell for the first time (creating the span buffer):
        *   Use the $(z, s)$ values from the first pixel of the Span Buffer (corresponding to the ray's entry point or the first intersection) for sampling.
        *   Determine the exit face from the Span Buffer.
        *   Use the exit-point $(z, s)$ values as the entry-point values for the neighbor cell.
        *   Follow the ray to the neighbor cell using connectivity relations (as in Koyamada’s algorithm) [S1].
    *   **Subsequent Hit (Same Scanline):** If a ray from the *same* scanline hits the cell:
        *   Retrieve the $(z, s)$ values and exit-face identifier directly from the Span Buffer at the current pixel's column.
        *   Use these values for sampling and to determine the next cell. No new span buffer creation is needed [S1].
    *   **Subsequent Hit (Next Scanline):** If a ray from the *next* scanline hits the cell for the first time:
        *   Create a new Span Buffer for the cell for this new scanline.
        *   Proceed with scan conversion for the new scanline [S1].

5.  **Deletion and Cleanup:**
    *   **Span Buffer Deletion:** After processing a scanline, initialize the span buffer list via a pointer operation to overwrite previous span buffers with new ones for the next scanline. This avoids explicit memory deallocation [S1].
    *   **Edge Deletion:** Traverse the Active Edge List after each scanline. Delete edges that are in the `done` state and will never be accessed again. Rasterize remaining active edges by updating their coordinates for the next scanline [S1].
    *   **Cell Deletion:** After processing a scanline, use the Y-Bucket List. For the current scanline $y$, access the Y-Bucket at index $y$. Delete the cell entries in the Active Cell List pointed to by this bucket. This handles the uncertainty caused by early ray termination, as we cannot guarantee that rays from the rightmost pixels will hit the cell [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def span_buffer_ray_casting(grid, camera, screen_width, screen_height):
    # Data Structures
    active_cell_list = {}  # Map: cell_id -> {y_bottom, span_buffer, edges}
    y_bucket_list = [ [] for _ in range(screen_height) ]
    active_edge_list = {}  # Hash Map: edge_id -> edge_data
    
    # Pre-compute back-face edges and sort them for each cell upon first hit
    # This is done inside the cell activation step
    
    for y in range(screen_height):
        for x in range(screen_width):
            ray = camera.get_ray(x, y)
            
            # Find first hit cell
            cell_id = find_first_hit_cell(grid, ray)
            
            if cell_id is None:
                continue
                
            # Check if cell is active
            if cell_id not in active_cell_list:
                activate_cell(cell_id, grid, active_cell_list, y_bucket_list, y)
            
            cell_data = active_cell_list[cell_id]
            
            # Check if span buffer is valid for current scanline
            # (Implicitly handled by creating new span buffer if needed)
            if cell_data['current_scanline'] != y:
                create_span_buffer_for_scanline(cell_id, y, active_cell_list, active_edge_list)
                cell_data['current_scanline'] = y
            
            span_buffer = cell_data['span_buffer']
            
            # Get exit info from span buffer for this pixel
            exit_z, exit_s, exit_face_id = span_buffer[x]
            
            # Sample the volume at the exit point (or use entry point if first hit)
            # Note: For the very first hit, entry point is ray origin. 
            # Subsequent hits use previous exit point.
            sample_value = sample_volume(grid, cell_id, exit_z, exit_s)
            
            # Determine next cell using connectivity and exit_face_id
            next_cell_id = get_neighbor_cell(cell_id, exit_face_id, grid)
            
            if next_cell_id is not None:
                # Update ray entry point for next cell
                ray.entry_z = exit_z
                ray.entry_s = exit_s
                # Follow ray to next cell (recursive or iterative step)
                # This part re-enters the loop logic for the next cell
                # In a real implementation, this might be a stack-based traversal
                pass 
            
            # Handle Early Ray Termination / Accumulation
            # (Standard volume rendering accumulation logic)
            pass

        # End of Scanline Processing
        delete_done_edges(active_edge_list)
        delete_cells_for_scanline(y, active_cell_list, y_bucket_list)

def activate_cell(cell_id, grid, active_cell_list, y_bucket_list, y):
    cell = grid.get_cell(cell_id)
    
    # 1. Allocate entry
    active_cell_list[cell_id] = {
        'y_bottom': cell.get_bottom_y(),
        'span_buffer': None,
        'current_scanline': -1,
        'back_face_edges': []
    }
    
    # 2. Identify back-face edges
    back_face_edges = identify_back_face_edges(cell, grid.camera_position)
    
    # 3. Sort back-face edges by x-intersection with horizontal line
    sorted_edges = sort_edges_by_x_intercept(back_face_edges)
    
    active_cell_list[cell_id]['back_face_edges'] = sorted_edges
    
    # 4. Insert into Y-Bucket
    y_bucket_list[cell.get_bottom_y()].append(cell_id)

def create_span_buffer_for_scanline(cell_id, y, active_cell_list, active_edge_list):
    cell_data = active_cell_list[cell_id]
    sorted_edges = cell_data['back_face_edges']
    
    # Update edge states and active edge list
    current_active_edges = []
    for edge in sorted_edges:
        if edge.is_intersecting_scanline(y):
            if edge.state == 'inactive':
                edge.state = 'active'
                active_edge_list[edge.id] = edge
            current_active_edges.append(edge)
        elif edge.state == 'active':
            if edge.is_below_scanline(y):
                edge.state = 'inactive'
                # Remove from active_edge_list if needed, or mark for deletion
                pass
    
    # Compute intersections and fill span buffer
    # Intersections are sorted by x
    intersections = compute_scanline_intersections(current_active_edges, y)
    
    # Create span buffer array for the cell's x-range
    span_buffer = create_span_buffer(intersections)
    cell_data['span_buffer'] = span_buffer

def delete_done_edges(active_edge_list):
    edges_to_delete = []
    for edge_id, edge in active_edge_list.items():
        if edge.state == 'done':
            edges_to_delete.append(edge_id)
    for edge_id in edges_to_delete:
        del active_edge_list[edge_id]

def delete_cells_for_scanline(y, active_cell_list, y_bucket_list):
    # Delete cells whose bottom y is <= current scanline
    # Note: The source says "pointers in the respective y-bucket list are used to delete"
    # Assuming y-bucket at index y contains cells that end at or before y?
    # Source: "pointer to active cell list entry is inserted into y-bucket list according to bottom y-coordinate"
    # "After processing a scanline, the pointers in the respective y-bucket list are used to delete"
    
    # If bottom y is <= current scanline, the cell is no longer needed for future scanlines
    # (Assuming scanlines go top to bottom, and y increases downwards or upwards consistently)
    # Let's assume y increases downwards. If bottom y <= current y, the cell is below the current scanline.
    
    cells_to_delete = []
    for cell_id in y_bucket_list[y]:
        cells_to_delete.append(cell_id)
        
    for cell_id in cells_to_delete:
        del active_cell_list[cell_id]
    
    # Clear the bucket
    y_bucket_list[y] = []
```

## Operating characteristics
*   **Time Complexity:** The algorithm avoids the extensive sorting operations of Challinger’s algorithm by using connectivity information [S1]. However, it requires sorting back-face edges upon cell activation. The scan conversion step involves hashing and linear traversal of active edges. Overall, it is efficient for general irregular grids.
*   **Memory Complexity:** Requires three additional data structures: Active Cell List, Active Edge List, and Span Buffers. The memory usage is proportional to the number of active cells and edges per scanline.
*   **Strengths:** Supports early ray termination, which is not possible in object-then-image methods like Challinger’s or LSRC [S1]. Utilizes both image-space and object-space coherencies [S1].
*   **Limitations:** The sorting of back-face edges adds overhead per cell activation. The span buffer management and edge state updates add complexity compared to simpler ray casting methods.
*   **When to Use:** Suitable for rendering general irregular grids (e.g., tetrahedral meshes) where early ray termination is beneficial and image-space coherency is high.
*   **When Not to Use:** If the grid is highly convex and simple, or if early ray termination is not a primary concern, simpler methods might be faster due to lower overhead.

## Background
Span-Buffer Ray-Casting (SBRC) was proposed by H. Berk et al. in 2003 [S1]. It is a hybrid method that combines aspects of Koyamada’s algorithm, Challinger’s algorithm, and Lazy Sweep Ray Casting (LSRC) [S1]. It was designed to address the deficiencies of prior hybrid methods, specifically their inability to support early ray termination and their redundant cell processing [S1].

## Sources
[S1] Direct volume rendering of unstructured grids, https://doi.org/10.1016/s0097-8493(03)00034-7, fetched 2026-10-24
