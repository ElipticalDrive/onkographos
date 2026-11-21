# Hardware-Adapted Object-Order Ray Casting

Also called: Hong hardware object-order, Object-Order Ray-Casting with Efficient Ray-Cell Intersection

## Overview
Hardware-Adapted Object-Order Ray Casting is a volume rendering technique that adapts object-order ray casting for modern graphics hardware by generating fragments for rays intersecting volume cells, with traversal order determined by min/max octrees [S1, unofficial]. Unlike image-order methods that process pixels sequentially, this approach processes visible volume cells, loading vertex data and accumulating color/opacity into corresponding pixels for all intersecting rays [S2]. It is distinguished from its parent method [Object-Order Ray Casting with Min/Max Octrees](1814-object-order-ray-casting-with-min-max-octrees.md) by its explicit optimization for hardware fragment generation and the use of pre-computed projection lists for orthogonal views [S1, unofficial][S2].

## How it works
**Inputs:** Volume dataset (scalar field), min/max octree structure, camera viewpoint (orthogonal projection assumed for efficient projection lists), iso-surface or transfer function parameters [S2].
**Outputs:** Rendered image (color and opacity per pixel) [S2].

**Data Structures:**
1.  **Min/Max Octree:** Hierarchical structure storing min/max values for sub-volumes to cull empty or fully occluded regions [S1, unofficial].
2.  **Projection Lists:** Pre-computed lists of relative pixel coordinates representing the template hexagon projection of a cell, associated with subdivisions of a square of four neighboring pixels [S2].
3.  **Layer Lists:** Pre-computed layers for sub-volumes to reduce visibility ordering complexity [S1, unofficial].

**Algorithm Steps:**

1.  **Precompute Projection Lists (Viewpoint Dependent):**
    *   Subdivide a square of four neighboring pixels on the image plane [S2].
    *   For each subdivision, store a list of relative coordinates corresponding to the template hexagon projection of a cell [S2].
    *   Store the global coordinate of the square where a cell center projects [S2].
    *   *Note:* These lists are recomputed every time the viewpoint changes [S2].

2.  **Traverse Octree and Determine Visibility:**
    *   Use the min/max octree to implicitly determine the correct front-to-back order between sub-volumes [S1, unofficial].
    *   Divide sub-volumes into pre-computed layers to further reduce visibility ordering complexity [S1, unofficial].
    *   Identify visible cells (not transparent and not hidden) [S2].

3.  **For Each Visible Cell:**
    *   Load the values and gradients of the 8 vertices of the cell [S2].
    *   Determine the global coordinate of the square where the cell center projects [S2].
    *   Identify the subdivision containing the projected center [S2].
    *   Retrieve the relative coordinate list for that subdivision [S2].
    *   Calculate rasterized pixel indices by adding the relative coordinates to the global coordinate of the square [S2]:
        $$ P_{\text{rasterized}} = P_{\text{global}} + C_{\text{relative}} $$
        Where $P_{\text{global}}$ is the global coordinate of the square and $C_{\text{relative}}$ is a relative coordinate from the list [S2].
    *   **Correction Step:** Test line equations to filter out erroneous pixels projected due to the reuse of relative coordinate lists across different cells within the same subdivision [S2].
    *   Generate fragments corresponding to the rays intersecting the cell [S1, unofficial].
    *   For every ray intersecting the cell:
        *   Sample colors and opacities along the ray [S2].
        *   Accumulate the results into the equivalent pixel in front-to-back order [S2].
        *   Store intermediate luminance and density values as 16-bit unsigned integers for sequential line integral estimation [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hardware_adapted_object_order_ray_casting(volume, octree, viewpoint):
    # Step 1: Precompute projection lists (recomputed on viewpoint change)
    projection_lists = precompute_projection_lists(viewpoint)
    
    # Step 2: Traverse octree to get visible cells in order
    visible_cells = traverse_octree_for_visibility(octree)
    
    for cell in visible_cells:
        # Load vertex data
        vertex_values, vertex_gradients = load_cell_data(cell)
        
        # Determine projection
        cell_center_proj = project_cell_center(cell, viewpoint)
        square_global, subdivision = find_square_and_subdivision(cell_center_proj, projection_lists)
        
        # Get relative coordinates for this subdivision
        relative_coords = projection_lists[subdivision]
        
        # Generate fragments and accumulate
        for rel_coord in relative_coords:
            # Calculate pixel index
            pixel_idx = square_global + rel_coord
            
            # Correction: Test line equations to filter erroneous pixels [S2]
            if is_valid_pixel(cell, pixel_idx):
                # Sample and accumulate along ray for this pixel
                accumulate_pixel(pixel_idx, cell, vertex_values, vertex_gradients)

def is_valid_pixel(cell, pixel_idx):
    # Test line equations to correct for erroneous projections [S2]
    # Implementation detail: Check if pixel_idx lies within the projected hexagon boundary
    # of the cell using half-plane tests.
    return check_line_equations(cell, pixel_idx)

def precompute_projection_lists(viewpoint):
    # Subdivide a square of four neighboring pixels [S2]
    # Associate list of relative coordinates (template hexagon) with each subdivision [S2]
    # Store global coordinate of the square where cell center projects [S2]
    # Return dictionary mapping (square_global, subdivision_id) -> list of relative_coords
    lists = {}
    # Standard implementation: grid subdivision based on viewpoint projection matrix
    return lists

def traverse_octree_for_visibility(octree):
    # Use min/max octree to implicitly determine correct order [S1, unofficial]
    # Divide sub-volumes into pre-computed layers [S1, unofficial]
    # Return list of visible cells in front-to-back order
    return []

def accumulate_pixel(pixel_idx, cell, vertex_values, vertex_gradients):
    # For every ray intersecting the cell, sample colors/opacities [S2]
    # Update equivalent pixel in front-to-back order [S2]
    # Store intermediate values as 16-bit unsigned integers [S2]
    pass
```

**Parameters:**
*   **Viewpoint:** The projection lists and preprocessed rays are recomputed every time the viewpoint changes [S2].
*   **Macro-Cell Size:** While not explicitly defined for this specific hardware adaptation in the notes, related macro-cell methods use sizes $m^3$ where $m$ is usually between four and ten [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Erroneous Pixels:** The same relative coordinate list is used for all cells within the same subdivision, which can project erroneous pixels. This is corrected by testing line equations [S2].
*   **Viewpoint Changes:** Performance may degrade if the viewpoint changes frequently due to the need to recompute projection lists [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of visible cells and rays intersecting them. Pre-computation of projection lists is $O(1)$ per viewpoint change but requires recomputation [S2].
*   **Memory Complexity:** Requires storage for the min/max octree, projection lists, and intermediate values (16-bit unsigned integers) [S1, unofficial][S2].
*   **Strengths:** Efficient for orthogonal projections; leverages hardware fragment generation; reduces visibility ordering complexity with pre-computed layers [S1, unofficial][S2].
*   **Limitations:** Projection lists must be recomputed for viewpoint changes; potential for erroneous pixels requiring correction [S2].
*   **When to Use:** Orthogonal visualization scenarios where hardware acceleration is desired [S2].
*   **When Not to Use:** Perspective views (projection lists are optimized for orthogonal) or frequent viewpoint changes [S2].

## Background
Originating from Hong et al. [Hong05], this method adapts object-order ray casting for current graphics hardware [S1, unofficial]. It builds on the concept of using hierarchical occlusion maps and min/max octrees for visibility determination [S1, unofficial]. It is related to iso-surface rendering algorithms based on object-order ray casting, such as those by Neubauer et al. [Neubauer02] [S1, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] A new object-order ray-casting algorithm, https://doi.org/10.5555/602099.602130, fetched 2026-10-26
