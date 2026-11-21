# Hermite Data Computation via Bisection

Also called: Bisection-based Hermite Data Computation, Edge Crossing Detection with Bisection

## Overview
Hermite Data Computation via Bisection is a method for extracting precise intersection points and surface normals from an adaptive octree scalar field by refining linear interpolation guesses with iterative bisection [S1, unofficial]. It solves the problem of geometric inaccuracy in adaptive grids where linear interpolation between coarse leaf corners introduces visible jaggedness on curved surfaces, providing the high-precision Hermite data (position and normal) required for Dual Contouring [S1, unofficial]. Unlike [Tree Marcher](7534-tree-marcher.md) or [Tetrahedralization of Octree Cells](9049-tetrahedralization-of-octree-cells.md), which may rely on simpler linear estimates or topological tetrahedralization, this method explicitly refines edge crossings to sub-cell accuracy to ensure manifold quality and sharp feature preservation.

## How it works
**Inputs:**
*   `octree`: An adaptive octree structure containing leaf cells with scalar field values (`corner_values`) and bounds.
*   `iso_value`: The scalar threshold defining the isosurface.
*   `func`: An implicit function evaluator capable of batch evaluation of scalar values and gradients at 3D points.

**Outputs:**
*   `hermite_data`: A collection of `HermitePoint` structures, each containing the refined intersection position, the surface normal, and the parameter `t` along the edge.

**Data Structures:**
*   `EdgeCrossing` (or `SignChangeEdge`): A struct storing the geometry and values of a detected edge intersection.
    *   `p0`: `[f64; 3]` - World coordinates of the first endpoint of the edge.
    *   `p1`: `[f64; 3]` - World coordinates of the second endpoint of the edge.
    *   `v0`: `f64` - Signed scalar value at `p0` (i.e., `field(p0) - iso_value`).
    *   `v1`: `f64` - Signed scalar value at `p1` (i.e., `field(p1) - iso_value`).
    *   `leaf_idx`: `usize` - Index of the leaf cell containing this edge.
    *   `edge`: `u8` - Index of the edge within the cell (0-11).
    *   `comp_id`: `u8` - Component ID for multi-component cells (used in Dual Contouring context).
*   `HermitePoint`:
    *   `pos`: `[f64; 3]` - Refined intersection point.
    *   `normal`: `[f64; 3]` - Normalized gradient at `pos`.
    *   `t`: `f64` - Parameter along the edge segment `[0.0, 1.0]`.

**Algorithm Steps:**

1.  **Initialization:**
    *   Retrieve `iso_value` from `octree.iso_value`.
    *   Retrieve `max_depth` from `octree.max_depth`.
    *   Calculate `cell_size_at_max` as `octree.bounds.size / (2^max_depth)` [S1, unofficial].

2.  **Sign Change Detection:**
    *   Iterate over all leaf cells in the octree.
    *   For each leaf cell, iterate over its 12 edges.
    *   For each edge, retrieve the signed scalar values `v0` and `v1` at its endpoints `p0` and `p1`.
    *   If `v0` and `v1` have opposite signs (i.e., `(v0 < 0.0) != (v1 < 0.0)`), a sign change exists.
    *   Record the edge in a list of `SignChangeEdge` structs, including `p0`, `p1`, `v0`, `v1`, `leaf_idx`, `edge`, and `comp_id` (derived from connected component analysis of the cell's corners) [S1, unofficial].

3.  **Bisection Refinement:**
    *   Initialize bisection intervals for all crossings: `lo_t[i] = 0.0`, `hi_t[i] = 1.0`, `lo_val[i] = v0` for each crossing `i`.
    *   Perform a fixed number of bisection iterations (typically 8 to 10) [S1, unofficial]:
        *   Compute the midpoint parameter for each crossing: `t_mid[i] = (lo_t[i] + hi_t[i]) * 0.5`.
        *   Compute the 3D position for each crossing: `pos[i] = lerp3(p0[i], p1[i], t_mid[i])`.
        *   Batch evaluate the implicit function at all `pos[i]` to get scalar values.
        *   Compute `mid_val[i] = scalar(pos[i]) - iso_value`.
        *   Update the interval:
            *   If `mid_val[i]` and `lo_val[i]` have the same sign, set `lo_t[i] = t_mid[i]` and `lo_val[i] = mid_val[i]`.
            *   Otherwise, set `hi_t[i] = t_mid[i]`.
    *   After iterations, compute the final refined parameter: `t_final[i] = (lo_t[i] + hi_t[i]) * 0.5`.
    *   Compute the final intersection position: `pos[i] = lerp3(p0[i], p1[i], t_final[i])`.

4.  **Normal Computation:**
    *   Batch evaluate the implicit function at all final `pos[i]` to obtain gradients.
    *   If gradients are available, normalize them:
        *   If `|gradient| > 1e-12`, `normal[i] = gradient / |gradient|`.
        *   Else, `normal[i] = [0.0, 0.0, 1.0]` [S1, unofficial].
    *   If gradients are not directly available, estimate them using finite differences with step size `h = cell_size_at_max * 0.01`. Normalize the estimated gradients using the same logic [S1, unofficial].

5.  **Storage:**
    *   Store the `HermitePoint` (pos, normal, t) in the octree's `hermite_data` structure, associated with the leaf and edge index.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_hermite_data(octree, iso_value, func):
    """
    Computes Hermite data (intersection points and normals) for an adaptive octree
    using bisection refinement on edge crossings.
    """
    max_depth = octree.max_depth
    cell_size_at_max = octree.bounds.size / (1 << max_depth)
    
    # Step 1: Collect all sign-change edges from leaf cells
    all_edges = []
    for leaf_idx, leaf in enumerate(octree.leaves):
        # Get connected components for this leaf to assign comp_id
        components = get_cell_components(leaf)
        
        for edge_idx in range(12):
            # Get corner indices for this edge
            c0, c1 = CubicBounds.edge_corners(edge_idx)
            
            # Get signed values at endpoints
            v0 = leaf.corner_values[c0] - iso_value
            v1 = leaf.corner_values[c1] - iso_value
            
            # Check for sign change
            if (v0 < 0.0) != (v1 < 0.0):
                p0 = leaf.bounds.corner(c0)
                p1 = leaf.bounds.corner(c1)
                comp_id = edge_component(components, leaf.corner_mask, edge_idx)
                
                all_edges.append({
                    'p0': p0,
                    'p1': p1,
                    'v0': v0,
                    'v1': v1,
                    'leaf_idx': leaf_idx,
                    'edge': edge_idx,
                    'comp_id': comp_id
                })
    
    num_crossings = len(all_edges)
    if num_crossings == 0:
        return []

    # Step 2: Bisection Refinement
    lo_t = [0.0] * num_crossings
    hi_t = [1.0] * num_crossings
    lo_val = [edge['v0'] for edge in all_edges]
    
    # Perform 10 bisection iterations
    for _ in range(10):
        if num_crossings == 0:
            break
            
        # Compute midpoints
        mid_points = []
        for i in range(num_crossings):
            t = (lo_t[i] + hi_t[i]) * 0.5
            pos = lerp3(all_edges[i]['p0'], all_edges[i]['p1'], t)
            mid_points.append(pos)
        
        # Batch evaluate implicit function
        results = func.batch_evaluate(mid_points)
        
        # Update intervals
        for i in range(num_crossings):
            mid_val = results.values[i] - iso_value
            mid_t = (lo_t[i] + hi_t[i]) * 0.5
            
            if (mid_val < 0.0) == (lo_val[i] < 0.0):
                lo_t[i] = mid_t
                lo_val[i] = mid_val
            else:
                hi_t[i] = mid_t

    # Step 3: Compute Final Positions and Normals
    hermite_points = []
    final_positions = []
    
    for i in range(num_crossings):
        t_final = (lo_t[i] + hi_t[i]) * 0.5
        pos = lerp3(all_edges[i]['p0'], all_edges[i]['p1'], t_final)
        final_positions.append(pos)
        
        hermite_points.append({
            'pos': pos,
            't': t_final,
            'leaf_idx': all_edges[i]['leaf_idx'],
            'edge': all_edges[i]['edge'],
            'comp_id': all_edges[i]['comp_id']
        })
    
    # Batch evaluate gradients at final positions
    if final_positions:
        grad_results = func.batch_evaluate(final_positions, return_gradients=True)
        
        for i in range(num_crossings):
            if grad_results.gradients is not None:
                g = grad_results.gradients[i]
                norm_g = length(g)
                if norm_g > 1e-12:
                    normal = g / norm_g
                else:
                    normal = [0.0, 0.0, 1.0]
            else:
                # Finite difference estimation if gradients not available
                h = cell_size_at_max * 0.01
                g = estimate_gradient_fd(func, final_positions[i], h)
                norm_g = length(g)
                if norm_g > 1e-12:
                    normal = g / norm_g
                else:
                    normal = [0.0, 0.0, 1.0]
            
            hermite_points[i]['normal'] = normal
    else:
        for i in range(num_crossings):
            hermite_points[i]['normal'] = [0.0, 0.0, 1.0]
            
    return hermite_points

def lerp3(p0, p1, t):
    return [p0[j] + t * (p1[j] - p0[j]) for j in range(3)]

def length(v):
    return (v[0]**2 + v[1]**2 + v[2]**2)**0.5

def estimate_gradient_fd(func, point, h):
    # Central difference approximation
    # Note: This is a simplified placeholder for the actual finite difference logic
    # which would require 6 evaluations per point.
    # For implementation, one would call func.evaluate at point + h*e_i and point - h*e_i
    pass
```

**Parameters:**
*   `bisection_iters`: Number of bisection iterations. Typical value: 10. Provides approximately $1/1024$ of a cell accuracy [S1, unofficial].
*   `iso_value`: The scalar value defining the isosurface.
*   `cell_size_at_max`: The physical size of a cell at the maximum octree depth, used for finite difference step size estimation.

**Edge Cases and Pitfalls:**
*   **Division by Zero in Linear Interpolation:** If $|v1 - v0| < 1e-15$, the denominator in linear interpolation is near zero. The method handles this by defaulting the initial guess `t` to 0.5 [S1, unofficial].
*   **Zero-Length Gradient:** If the gradient magnitude is $< 1e-12$, normalization fails. The method defaults the normal to $[0.0, 0.0, 1.0]$ [S1, unofficial].
*   **No Sign Change:** If no sign change exists on an edge, no Hermite data is computed for that edge. This is expected behavior but must be handled to avoid gaps [S1, unofficial].

## Subtopics

### [Edge Completeness Enforcement](825-edge-completeness-enforcement.md)
A post-processing step that ensures all edges with sign changes possess complete 4-cell neighborhoods at the same depth, eliminating geometric gaps at depth boundaries. It focuses on topological consistency rather than precise intersection computation.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot I)$, where $N$ is the number of sign-change edges and $I$ is the number of bisection iterations. Each iteration requires a batch evaluation of the implicit function.
*   **Memory Complexity:** $O(N)$ to store the edge crossings and Hermite data.
*   **Performance:** The method is efficient due to batched evaluations of the implicit function and gradients. 8-10 iterations provide sufficient accuracy for most visual applications, reducing jaggedness on curved surfaces compared to linear interpolation [S1, unofficial].
*   **Strengths:** High precision intersection points, manifold guarantee when combined with Dual Contouring, handles adaptive grids effectively.
*   **Limitations:** Requires batch evaluation capability for efficiency; slightly more computationally expensive than simple linear interpolation.
*   **When to Use:** When high-quality, manifold meshes are required from adaptive octrees, especially for Dual Contouring where precise normals and positions are critical for QEF solving.
*   **When Not to Use:** When performance is critical and lower precision is acceptable, or when the scalar field is linear within cells (where bisection adds no value).

## Background
The method originates from implementations of Dual Contouring on adaptive octrees, specifically as described in the `isomesh` library by Mason [S1, unofficial]. It addresses the limitations of standard Marching Cubes on adaptive grids, which can produce non-manifold vertices or self-intersections, by providing the precise Hermite data needed for robust vertex placement via Quadratic Error Functions (QEF) [S1, unofficial]. The use of bisection refinement is a standard technique in implicit surface rendering to improve accuracy over linear interpolation, particularly for sharp features and curved surfaces.

## Sources
[S1] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
