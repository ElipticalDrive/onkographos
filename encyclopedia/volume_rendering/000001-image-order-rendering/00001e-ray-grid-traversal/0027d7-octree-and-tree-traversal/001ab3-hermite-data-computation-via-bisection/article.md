# Hermite Data Computation via Bisection

Also called: Bisection-based Hermite Data Computation, Edge Crossing Detection with Bisection

## Overview
A method to find all edge crossings in an adaptive octree for an implicit function and compute Hermite data (intersection points and normals) using bisection [S1, unofficial]. It refines linear interpolation guesses with bisection to achieve higher precision than standard Marching Cubes, which is essential for dual contouring methods that rely on accurate vertex placement via Quadratic Error Functions (QEF) [S1, unofficial]. Unlike [Octree Path Computation](897-octree-path-computation.md) which focuses on traversal efficiency, this method focuses on geometric accuracy of the isosurface intersection.

## How it works
**Inputs:**
- `octree`: An adaptive octree structure containing leaf cells. Each leaf cell has:
  - `bounds`: Cubic bounds defining the cell's spatial extent.
  - `corner_values`: Array of 8 scalar values representing the implicit function at the cell's corners.
  - `corner_mask`: Bitmask indicating which corners are inside the isosurface.
  - `depth`: The depth level of the leaf in the octree.
- `iso_value`: The scalar value defining the isosurface.
- `func`: A function evaluator capable of batch evaluation of the implicit function and its gradients at a list of 3D points.

**Outputs:**
- `hermite_data`: A collection of `HermitePoint` structs, one for each detected edge crossing.

**Data Structures:**
- `EdgeCrossing`: A temporary struct used during collection.
  - `p0: [f64; 3]`: Start point of the edge.
  - `p1: [f64; 3]`: End point of the edge.
  - `v0: f64`: Signed function value at `p0` (i.e., `value - iso_value`).
  - `v1: f64`: Signed function value at `p1`.
  - `leaf_idx: usize`: Index of the leaf cell containing the edge.
  - `edge: u8`: Index of the edge (0–11) within the cell.
  - `comp_id: u8`: Component ID for multi-component cells (from connected component analysis).
- `HermitePoint`: The final output struct.
  - `pos: [f64; 3]`: The computed intersection point.
  - `normal: [f64; 3]`: The normalized gradient at the intersection point.
  - `t: f64`: The parameter along the edge $[0, 1]$ where the intersection occurs.

**Steps:**

1. **Initialization:**
   - Extract `iso_value` from `octree.iso_value`.
   - Extract `max_depth` from `octree.max_depth`.
   - Calculate `cell_size_at_max` as `octree.bounds.size / (1u32 << max_depth)` [S1, unofficial].

2. **Collect Sign-Change Edges:**
   - Iterate over all leaf cells in the octree.
   - For each leaf cell, iterate over its 12 edges.
   - For each edge, identify the two corner vertices `c0` and `c1`.
   - Compute signed values: `v0 = corner_values[c0] - iso_value` and `v1 = corner_values[c1] - iso_value`.
   - If `(v0 < 0.0) != (v1 < 0.0)`, a sign change exists.
   - Record the edge endpoints `p0 = bounds.corner(c0)`, `p1 = bounds.corner(c1)`, `v0`, `v1`, `leaf_idx`, `edge`, and `comp_id` (derived from connected component analysis of the cell's corner mask) in a list of `EdgeCrossing` structs [S1, unofficial].

3. **Bisection Refinement:**
   - Initialize arrays `lo_t` and `hi_t` of size `num_crossings` to `[0.0, 1.0]`.
   - Initialize `lo_val` with the `v0` values from the collected edges.
   - Perform 10 bisection iterations (or 8 as per some implementations, see [Operating characteristics]):
     - For each crossing `i`:
       - Compute midpoint parameter: `mid_t = (lo_t[i] + hi_t[i]) * 0.5`.
       - Compute 3D position: `pos = p0 + mid_t * (p1 - p0)`.
     - Batch evaluate the implicit function at all `pos` points.
     - For each crossing `i`:
       - Compute `mid_val = result.values[i] - iso_value`.
       - If `(mid_val < 0.0) == (lo_val[i] < 0.0)`:
         - `lo_t[i] = mid_t`
         - `lo_val[i] = mid_val`
       - Else:
         - `hi_t[i] = mid_t`

4. **Compute Final Intersection Points:**
   - For each crossing `i`:
     - `t = (lo_t[i] + hi_t[i]) * 0.5`
     - `pos = p0 + t * (p1 - p0)`

5. **Compute Normals:**
   - Batch evaluate the implicit function at all final `pos` points to obtain gradients.
   - If gradients are available:
     - For each gradient `g`:
       - If `len(g) > 1e-12`: `normal = g / len(g)`
       - Else: `normal = [0.0, 0.0, 1.0]` [G] (Guard against zero-length gradients)
   - If gradients are not available:
     - Estimate gradients using finite differences with step size `cell_size_at_max * 0.01`.
     - Normalize estimated gradients using the same logic.

6. **Store Hermite Data:**
   - Create `HermitePoint` structs with `pos`, `normal`, and `t`.
   - Store in `octree.hermite_data` associated with the leaf and edge index.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_hermite_data(octree, iso_value, func):
    max_depth = octree.max_depth
    cell_size_at_max = octree.bounds.size / (1 << max_depth)
    
    # Step 2: Collect sign-change edges
    all_edges = []
    for leaf_idx, leaf in enumerate(octree.leaves):
        components = get_components(leaf.corner_mask) # From connected component analysis
        for edge in range(12):
            c0, c1 = CubicBounds.edge_corners(edge)
            v0 = leaf.corner_values[c0] - iso_value
            v1 = leaf.corner_values[c1] - iso_value
            
            if (v0 < 0.0) != (v1 < 0.0):
                p0 = leaf.bounds.corner(c0)
                p1 = leaf.bounds.corner(c1)
                comp_id = edge_component(components, leaf.corner_mask, edge)
                all_edges.append({
                    'p0': p0, 'p1': p1, 'v0': v0, 'v1': v1,
                    'leaf_idx': leaf_idx, 'edge': edge, 'comp_id': comp_id
                })
    
    num_crossings = len(all_edges)
    if num_crossings == 0:
        return []
    
    # Step 3: Bisection Refinement
    lo_t = [0.0] * num_crossings
    hi_t = [1.0] * num_crossings
    lo_val = [edge['v0'] for edge in all_edges]
    
    # 10 iterations for ~1/1024 cell accuracy
    for _ in range(10):
        mid_points = []
        for i in range(num_crossings):
            t = (lo_t[i] + hi_t[i]) * 0.5
            mid_points.append(lerp3(all_edges[i]['p0'], all_edges[i]['p1'], t))
        
        result = func.batch_evaluate(mid_points)
        
        for i in range(num_crossings):
            mid_val = result.values[i] - iso_value
            mid_t = (lo_t[i] + hi_t[i]) * 0.5
            if (mid_val < 0.0) == (lo_val[i] < 0.0):
                lo_t[i] = mid_t
                lo_val[i] = mid_val
            else:
                hi_t[i] = mid_t
    
    # Step 4: Final Points
    crossing_points = []
    for i in range(num_crossings):
        t = (lo_t[i] + hi_t[i]) * 0.5
        crossing_points.append(lerp3(all_edges[i]['p0'], all_edges[i]['p1'], t))
    
    # Step 5: Normals
    if crossing_points:
        result = func.batch_evaluate(crossing_points)
        if result.gradients:
            normals = [normalize3(g) for g in result.gradients]
        else:
            fd_grads = func.estimate_gradients_fd(crossing_points, cell_size_at_max * 0.01)
            normals = [normalize3(g) for g in fd_grads]
    else:
        normals = []
    
    # Step 6: Store Hermite Data
    hermite_data = []
    for i in range(num_crossings):
        t = (lo_t[i] + hi_t[i]) * 0.5
        hermite_data.append({
            'pos': crossing_points[i],
            'normal': normals[i],
            't': t,
            'leaf_idx': all_edges[i]['leaf_idx'],
            'edge': all_edges[i]['edge']
        })
    
    return hermite_data

def lerp3(p0, p1, t):
    return [p0[j] + t * (p1[j] - p0[j]) for j in range(3)]

def normalize3(v):
    len_v = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    if len_v > 1e-12:
        return [v[j] / len_v for j in range(3)]
    return [0.0, 0.0, 1.0]
```

**Parameters:**
- `iso_value`: The scalar value defining the isosurface.
- `max_depth`: The maximum depth of the octree.
- `bisection_iters`: Number of bisection iterations. Value: 10. Context: Provides ~1/1024 cell accuracy [S1, unofficial].
- `cell_size_at_max`: The physical size of a cell at the maximum octree depth. Used for finite difference step size.

**Edge Cases and Pitfalls:**
- **No Sign Change:** If no sign change exists on an edge, no Hermite data is computed for that edge [S1, unofficial].
- **Division by Zero:** In linear interpolation (if used as initial guess), if `|v1 - v0| < 1e-15`, the denominator is near zero. Handled by defaulting `t` to 0.5 [S1, unofficial].
- **Zero-Length Gradient:** If the gradient magnitude is `< 1e-12`, normalization fails. Handled by defaulting the normal to `[0.0, 0.0, 1.0]` [S1, unofficial].

## Subtopics

### [Edge Completeness Enforcement](825-edge-completeness-enforcement.md)
Edge Completeness Enforcement is a post-processing step in adaptive octree surface extraction that ensures all edges with sign changes possess complete 4-cell neighborhoods at the same depth, thereby eliminating geometric gaps at depth boundaries. It differs from its parent, Hermite Data Computation via Bisection, by focusing on topological consistency of the grid structure rather than the precise computation of intersection points and normals.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of leaf cells and edges. Bisection adds a constant factor (10 iterations) per edge crossing.
- **Memory Complexity:** Linear with respect to the number of edge crossings.
- **Measured Performance:** 8 bisection iterations provide ~1/256 cell accuracy, which is sufficient for many cases, but 10 iterations are recommended for curved sharp edges (cylinders, chamfers) to avoid visible jaggedness [S1, unofficial].
- **Strengths:** High precision intersection points, essential for dual contouring and QEF-based vertex placement. Handles adaptive octrees effectively.
- **Limitations:** Requires batch evaluation of the implicit function. Finite difference gradient estimation adds overhead if analytical gradients are not available.
- **When to Use:** When high-precision isosurface extraction is required, particularly for dual contouring.
- **When Not to Use:** When performance is critical and lower precision is acceptable (e.g., simple Marching Cubes).

## Background
Originated in the context of adaptive octree surface extraction for implicit functions, specifically for dual contouring methods [S1, unofficial]. The method was developed to address the limitations of linear interpolation in finding edge crossings, which can introduce visible artifacts on curved surfaces. The implementation is found in the `isomesh` library [S1, unofficial].

## Sources
[S1] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
