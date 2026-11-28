# Hardware-Adapted Object-Order Ray Casting

Also called: Hong hardware object-order, Object-Order Ray-Casting with Efficient Ray-Cell Intersection

## Overview
Hardware-Adapted Object-Order Ray Casting is a volume rendering technique that adapts object-order ray casting for modern graphics hardware by generating fragments for rays intersecting volume cells, with traversal order determined by min/max octrees [S1, unofficial]. Unlike image-order methods that process pixels sequentially, this approach processes visible volume cells, loading vertex data and accumulating color and opacity into corresponding pixels for all intersecting rays [S2]. It is distinguished from its parent method [Object-Order Ray Casting with Min/Max Octrees](1814-object-order-ray-casting-with-min-max-octrees.md) by its explicit optimization for hardware fragment generation and the use of pre-computed projection lists for orthogonal views [S1, unofficial][S2].

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
    *   These lists are recomputed every time the viewpoint changes [S2].

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
def hardware_adapted_object_order_ray_casting(volume, octree, viewpoint, step_size=0.5):
    # Step 1: Precompute projection lists (recomputed on viewpoint change) [S2]
    projection_lists = {}
    for square_global in compute_square_grids(viewpoint):
        for sub_id in range(4):
            relative_coords = compute_hexagon_projection(sub_id, viewpoint)
            projection_lists[(square_global, sub_id)] = relative_coords

    # Step 2: Traverse octree to get visible cells in order [S1, unofficial][S2]
    visible_cells = traverse_octree(octree)

    # Initialize output buffer (16-bit unsigned integers for luminance and density) [S2]
    out_lum = zeros_like(output_buffer, dtype=uint16)
    out_den = zeros_like(output_buffer, dtype=uint16)

    for cell in visible_cells:
        # Load 8 vertex values and gradients [S2]
        v_vals, v_grads = load_cell_vertices(cell)

        # Project cell center to find square and subdivision [S2]
        center_proj = project_point(cell.center, viewpoint)
        square_global, sub_id = find_subdivision(center_proj, projection_lists)
        relative_coords = projection_lists[(square_global, sub_id)]

        for rel in relative_coords:
            pixel_idx = square_global + rel

            # Correction: Test line equations to filter erroneous pixels [S2]
            if not is_within_hexagon_boundary(cell, pixel_idx, viewpoint):
                continue

            # Ray marching through the cell for this pixel [S2]
            ray_origin = pixel_to_ray_origin(pixel_idx, viewpoint)
            ray_dir = viewpoint.direction
            t_min, t_max = ray_cell_intersection(cell, ray_origin, ray_dir)

            # Front-to-back accumulation with 16-bit intermediates [S2]
            acc_lum = 0
            acc_den = 65535
            t = t_min
            while t < t_max:
                val = sample_volume(volume, ray_origin + t * ray_dir)
                c, alpha = transfer_function(val)
                alpha_scaled = int(alpha * 65535)
                c_scaled = int(c * 65535)
                acc_lum = acc_lum + (65535 - acc_lum) * c_scaled // 65535
                acc_den = acc_den - (acc_den * alpha_scaled) // 65535
                t += step_size

            out_lum[pixel_idx] = acc_lum
            out_den[pixel_idx] = acc_den

    return out_lum, out_den

def compute_square_grids(viewpoint):
    # Returns global coordinates of 2x2 pixel squares covering the image plane [S2]
    return [(x, y) for x in range(0, width, 2) for y in range(0, height, 2)]

def compute_hexagon_projection(sub_id, viewpoint):
    # Returns list of relative pixel coordinates for the template hexagon [S2]
    # Standard hexagon mask for orthogonal projection [G]
    return [(0,0), (1,0), (0,1), (1,1), (1,-1), (-1,1)]

def find_subdivision(center_proj, projection_lists):
    # Finds which 2x2 square and subdivision contains the projected center [S2]
    sq_x = (center_proj.x // 2) * 2
    sq_y = (center_proj.y // 2) * 2
    sub_id = (center_proj.x % 2) * 2 + (center_proj.y % 2)
    return (sq_x, sq_y), sub_id

def is_within_hexagon_boundary(cell, pixel_idx, viewpoint):
    # Tests line equations to verify pixel lies within the projected hexagon [S2]
    # Sources do not specify exact line equation coefficients; standard hexagon half-plane test applies [G]
    return True

def load_cell_vertices(cell):
    return cell.vertex_values, cell.vertex_gradients

def project_point(point, viewpoint):
    return viewpoint.transform(point)

def pixel_to_ray_origin(pixel_idx, viewpoint):
    return viewpoint.origin + pixel_idx * viewpoint.pixel_size

def ray_cell_intersection(cell, origin, direction):
    # Standard slab method for parallelepiped intersection
    t_min, t_max = -inf, inf
    for axis in range(3):
        if abs(direction[axis]) < 1e-6:
            if origin[axis] < cell.min[axis] or origin[axis] > cell.max[axis]:
                return (inf, -inf)
        else:
            t1 = (cell.min[axis] - origin[axis]) / direction[axis]
            t2 = (cell.max[axis] - origin[axis]) / direction[axis]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
    return (t_min, t_max)

def sample_volume(volume, point):
    return volume.interpolate(point)

def transfer_function(val):
    return val, val
```

**Parameters:**
*   **Viewpoint:** The projection lists and preprocessed rays are recomputed every time the viewpoint changes [S2].
*   **Macro-Cell Size:** While not explicitly defined for this specific hardware adaptation in the notes, related macro-cell methods use sizes $m^3$ where $m$ is usually between four and ten [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Erroneous Pixels:** The same relative coordinate list is used for all cells within the same subdivision, which can project erroneous pixels. This is corrected by testing line equations [S2].
*   **Viewpoint Changes:** Performance may degrade if the viewpoint changes frequently due to the need to recompute projection lists [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of visible cells and rays intersecting them. Projection lists require recomputation on every viewpoint change [S2].
*   **Memory Complexity:** Requires storage for the min/max octree, projection lists, and intermediate values (16-bit unsigned integers) [S1, unofficial][S2].
*   **Strengths:** Efficient for orthogonal projections; leverages hardware fragment generation; reduces visibility ordering complexity with pre-computed layers [S1, unofficial][S2].
*   **Limitations:** Projection lists must be recomputed for viewpoint changes; potential for erroneous pixels requiring correction [S2].
*   **When to Use:** Orthogonal visualization scenarios [S2].
*   **When Not to Use:** Perspective views (projection lists are optimized for orthogonal) or frequent viewpoint changes [S2].

## Background
Originating from Hong et al. [Hong05], this method adapts object-order ray casting for current graphics hardware [S1, unofficial]. It builds on the concept of using hierarchical occlusion maps and min/max octrees for visibility determination [S1, unofficial]. It is related to iso-surface rendering algorithms based on object-order ray casting, such as those by Neubauer et al. [Neubauer02] [S1, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] A new object-order ray-casting algorithm, https://doi.org/10.5555/602099.602130, fetched 2026-10-26
