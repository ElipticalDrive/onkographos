# Projected Mesh Filtering

Also called: Multi-Plane Mesh Cutting

## Overview
Projected Mesh Filtering is a post-processing technique that removes mesh geometry lying outside the spatial extent of a ground-truth reference by projecting vertices onto the XY, XZ, and YZ planes and iteratively cutting the mesh based on 2D containment [S1, unofficial]. It differs from general simplification methods by relying on external reference data for spatial culling rather than intrinsic geometric metrics, and from vertex optimization by modifying topology (removing vertices/triangles) rather than adjusting positions [S1, unofficial].

## How it works
**Inputs:**
- Prediction mesh $M_{pred}$: Vertices $V_{pred}$, Triangles $T_{pred}$.
- Ground-truth mesh $M_{gt}$: Vertices $V_{gt}$ (used to define the valid region).
- `kernel_size`: Integer size for morphological dilation (typical value: 15) [S1, unofficial].

**Outputs:**
- Filtered mesh $M_{filtered}$: Subset of $V_{pred}$ and $T_{pred}$ where all vertices lie within the projected bounds of $M_{gt}$.

**Data Structures:**
- `contour_points`: List of 2D points defining the boundary of the projected ground-truth shape.
- `old_to_new_indices`: Map from original vertex index to new contiguous index.
- `valid_mask`: Boolean array indicating which vertices are retained.

**Steps:**
1. **Projection and Contour Generation:**
   For each axis-aligned plane ($P \in \{XY, XZ, YZ\}$):
   a. Project $V_{gt}$ onto $P$ to obtain 2D points.
   b. Compute the convex hull or contour of these projected points to form `contour_points` [S1, unofficial].
   c. Dilate `contour_points` using a kernel of size `kernel_size` to create a tolerance buffer [S1, unofficial].

2. **Vertex Filtering:**
   For each plane $P$:
   a. Project all vertices $v \in V_{pred}$ onto $P$.
   b. Test if each projected vertex lies inside the dilated `contour_points` using a point-in-polygon algorithm (e.g., `matplotlib.patches.Path.contains_point`) [S1, unofficial].
   c. Update `valid_mask`: a vertex is valid only if it passed the containment test for all three planes (XY, XZ, and YZ) [S1, unofficial].

3. **Mesh Reconstruction:**
   a. Create `old_to_new_indices` map for vertices where `valid_mask` is true.
   b. Iterate through $T_{pred}$. For each triangle $(i, j, k)$:
      - If $i, j, k$ are all in `old_to_new_indices`, create new triangle $(old\_to\_new[i], old\_to\_new[j], old\_to\_new[k])$.
      - Otherwise, discard the triangle.
   c. Construct $M_{filtered}$ with retained vertices and triangles [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def projected_mesh_filtering(mesh_pred, mesh_gt, kernel_size=15):
    """
    Filter mesh_pred based on the projected bounds of mesh_gt.
    """
    v_pred = mesh_pred.vertices
    t_pred = mesh_pred.triangles
    
    # Initialize validity mask
    valid = np.ones(len(v_pred), dtype=bool)
    
    # Define projection planes: (keep_indices, drop_index)
    planes = [
        ([0, 1], 2),  # XY
        ([0, 2], 1),  # XZ
        ([1, 2], 0)   # YZ
    ]
    
    for keep_idx, drop_idx in planes:
        # 1. Project Ground Truth
        v_gt_proj = mesh_gt.vertices[:, keep_idx]
        
        # 2. Compute Contour (Convex Hull)
        # Note: Implementation of contour generation is assumed to produce a closed polygon
        hull = ConvexHull(v_gt_proj)
        contour_points = v_gt_proj[hull.vertices]
        
        # 3. Dilate Contour
        # Apply morphological dilation with kernel_size
        dilated_contour = dilate_polygon(contour_points, kernel_size)
        
        # 4. Create Path for Point-in-Polygon Test
        path = Path(dilated_contour)
        
        # 5. Project Prediction Vertices and Test Containment
        v_pred_proj = v_pred[:, keep_idx]
        inside = path.contains_points(v_pred_proj)
        
        # 6. Update Validity Mask (Intersection across all planes)
        valid = valid & inside
        
    # 7. Filter Vertices
    v_filtered = v_pred[valid]
    
    # 8. Map Old Indices to New Indices
    old_to_new = {old_idx: new_idx for new_idx, old_idx in enumerate(np.where(valid)[0])}
    
    # 9. Filter Triangles
    t_filtered = []
    for t in t_pred:
        if all(idx in old_to_new for idx in t):
            new_t = tuple(old_to_new[idx] for idx in t)
            t_filtered.append(new_t)
            
    return TriangleMesh(v_filtered, np.array(t_filtered))
```

**Parameters:**
- `kernel_size`: Size of the dilation kernel applied to the projected ground-truth contours. Typical value: 15 [S1, unofficial].

**Edge Cases:**
- **Degenerate Ground Truth:** If $M_{gt}$ has no volume or is empty, the contour is undefined. [G] Handle by returning the original mesh or raising an error.
- **Precision:** Point-in-polygon tests are sensitive to floating-point errors near boundaries. [G] The dilation step mitigates this by providing a margin.
- **Non-Manifold Input:** If $M_{pred}$ is non-manifold, the triangle filtering step may produce isolated vertices or degenerate triangles. [G] Post-process to remove degenerate triangles (area < epsilon).

## Subtopics

### [Mesh Simplification and Optimization](8588-mesh-simplification-and-optimization.md)
Reduces triangle count via edge collapse; choose when memory is constrained, not when spatial accuracy relative to a reference is needed.

### [Vertex Position Optimization and Error Minimization](10991-vertex-position-optimization-and-error-minimization.md)
Adjusts vertex positions to fit the isosurface; choose when geometric fidelity is critical, not when topology removal is required.

### [Marching Cubes Boundary Cap](13166-marching-cubes-boundary-cap.md)
Closes open boundaries; choose when the mesh is incomplete, not when filtering external noise.

### [Voxel-based Object Simplification via Multi-layered Marching Cubes](19772-voxel-based-object-simplification-via-multi-layered-marching.md)
Simplifies via voxel coarsening; choose for hierarchical simplification, not for reference-based culling.

## Operating characteristics
- **Time Complexity:** $O(|V_{pred}| + |T_{pred}|)$ for filtering, plus $O(|V_{gt}| \log |V_{gt}|)$ for convex hull computation per plane. Point-in-polygon test is $O(\log N)$ or $O(N)$ depending on implementation.
- **Memory Complexity:** $O(|V_{pred}|)$ for the validity mask and index mapping.
- **Strengths:** Effective at removing "flying debris" or artifacts outside the object's true extent; simple to implement.
- **Limitations:** Requires a ground-truth mesh; sensitive to the alignment of $M_{pred}$ and $M_{gt}$; dilation parameter tuning is necessary to avoid over-cropping.
- **When to use:** Post-processing predicted meshes from neural reconstruction methods where ground-truth geometry is available for validation.
- **When not to use:** When no reference mesh exists; when the goal is general simplification rather than spatial culling.

## Background
Originated in the context of neural surface reconstruction evaluation, specifically within the `dn-splatter` framework for filtering meshes extracted from 2D Gaussian Splatting or similar implicit representations [S1, unofficial]. It addresses the issue of noisy or extended geometry in predicted meshes that exceeds the bounds of the ground-truth object.

## Sources
[S1] maturk/dn-splatter, https://github.com/maturk/dn-splatter, fetched 2026-10-27 (unofficial)
