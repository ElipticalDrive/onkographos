# Adaptive Octree Construction with 2:1 Balance and Surface Propagation

Also called: Algorithm Improvements and Extensions, Implicit Function Octree Construction, Isotopic Octree Approximation, adaptive grid refinement for isosurfaces, isomesh octree building

## Overview
This method constructs an adaptive octree representation of an implicit surface by refining a 3D grid based on implicit function sign changes and geometric features, enforcing a 2:1 balance constraint (no face-adjacent leaves differ by more than 1 depth level) and ensuring manifold connectivity through surface propagation rules [S1, unofficial]. It solves the problem of generating a watertight, isotopic mesh approximation of complex implicit functions while minimizing vertex count compared to uniform grids [S1, unofficial]. Unlike [Homogeneity-Based KD-Tree Construction](1325-homogeneity-based-kd-tree-construction.md) which uses axis-aligned splits, or [MFA-DVR](1091-mfa-dvr.md) which focuses on direct volume rendering, this method produces a polygonal mesh via Dual Contouring [S1, unofficial]. It differs from [Constrained Octree Refinement](2426-constrained-octree-refinement.md) by explicitly enforcing the 2:1 balance during construction to prevent excessive geometric simplification, and from [Topological Octree Construction](4698-topological-octree-construction.md) by combining topological criteria with the specific balancing mechanism [S1, unofficial].

## How it works
The algorithm builds an octree where leaf cells are refined until they satisfy isotopy conditions (containing the surface or being clearly empty/inside) and balance constraints. It uses Dual Contouring for mesh extraction, ensuring manifold output [S1, unofficial].

**Inputs:**
- `func`: A vectorized implicit function $f: \mathbb{R}^3 \to \mathbb{R}$ and optionally its gradient $\nabla f$. Signature: `func(positions: np.ndarray) -> (values: np.ndarray, gradients: np.ndarray | None)` [S1, unofficial].
- `bbox_min`, `bbox_max`: Bounding box of the domain [S1, unofficial].
- `min_depth`: Minimum subdivision depth (default 3) [S1, unofficial].
- `max_depth`: Maximum subdivision depth (default 7) [S1, unofficial].
- `angle_threshold_deg`: Threshold for sharp feature detection (default 30.0) [S1, unofficial].
- `iso_value`: Isosurface value (default 0.0) [S1, unofficial].

**Outputs:**
- `vertices`: Array of mesh vertices [S1, unofficial].
- `faces`: Array of mesh face indices [S1, unofficial].

**Data Structures:**
- **Octree Node**: Contains a bounding box, depth, and a list of children. Leaves store cell data.
- **EvalCache**: Stores computed values and gradients for grid corners to avoid redundant evaluations.
- **AllLeaves**: A vector of leaf cells with coordinates $(cx, cy, cz)$ and depth, used for balancing [S1, unofficial].

**Parameters:**
- `min_depth`: u8. Typical: 3.
- `max_depth`: u8. Typical: 7.
- `angle_threshold_deg`: f64. Typical: 30.0.
- `iso_value`: f64. Typical: 0.0.
- `fd_step`: f64. Finite difference step if gradients not provided. Typical: $10^{-5}$ or `cell_size_at_max * 0.01` [S1, unofficial].

**Steps:**

1. **Initialization**: Initialize the octree with the bounding box. Build a uniform grid up to `min_depth`. If adaptive mode is enabled, call `build_adaptive` which internally calls `enforce_balance_2to1` [S1, unofficial].

2. **Adaptive Refinement Loop**:
   For each leaf cell $C$ in the octree:
   a. **Evaluation**: Compute the implicit function values at the 8 corners of cell $C$. Convert grid positions to world coordinates using `cache::grid_to_world`. Batch evaluate using `bridge::batch_evaluate` [S1, unofficial].
   b. **Gradient Handling**: If `func` does not return gradients, estimate them using finite differences with step size `fd_step`. Store values and gradients in `EvalCache` [S1, unofficial].
   c. **Refinement Criteria**: Mark cell $C$ for subdivision if:
      - The cell contains the surface (sign changes on edges or faces).
      - The normal angle spread across the cell exceeds `angle_threshold_deg`.
      - The gradient variation is high.
      - **Rule 3 (Surface Propagation)**: If current cell $C$ is a surface cell, a face-neighbor $N$ is at the same depth but is a Branch containing surface descendants, and $depth(C) < max\_depth$, then $C$ must subdivide to prevent cross-depth gaps [S1, unofficial].
   d. **Subdivision**: For each marked cell, compute the 8 child octants. Recursively apply refinement to children until leaf conditions are met or `max_depth` is reached [S1, unofficial].

3. **2:1 Balance Enforcement**:
   a. Collect all leaf cells into `all_leaves` vector [S1, unofficial].
   b. Iterate through `all_leaves`. If a leaf $L$ has a face-adjacent neighbor $N$ with $depth(N) < depth(L) - 1$, subdivide $L$ (and potentially its siblings) until the depth difference is $\le 1$. Use `subdivide_cell_for_balance` [S1, unofficial].
   c. Note: Balancing is not strictly necessary for isotopy but simplifies leaf approximation and ensures uniformity [S2].

4. **Edge Completeness**:
   Call `ensure_edge_completeness` to ensure every sign-change edge has all 4 sharing cells present as surface leaves. This eliminates gaps at depth boundaries. Check non-canonical edges (1-3, 5-7, 9-11) for sign changes, as surface cells not owning a canonical edge still participate in quads generated by neighbors [S1, unofficial].

5. **Mesh Extraction (Dual Contouring)**:
   a. **QEF Vertex Placement**: For each surface leaf, construct a Quadric Error Function (QEF) based on the interpolated Hermite data (values and gradients) at the cell boundaries. Solve for the vertex position using Jacobi SVD eigendecomposition. Apply mass-point bias for rank-deficient cases to preserve sharp features [S1, unofficial].
   b. **Newton Projection**: Perform one Newton step to project each vertex exactly onto the zero-isosurface $f(x)=0$ [S1, unofficial].
   c. **Quad Generation**: Iterate over canonical edges. Generate quads based on the flatness of the surface within the cell. Triangulate quads if necessary [S1, unofficial].
   d. **Watertight Guarantee**: All surface leaves are forced to `max_depth` during extraction to eliminate T-junction holes [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_implicit_surface(func, bbox_min, bbox_max, min_depth=3, max_depth=7, angle_threshold=30.0, iso_value=0.0):
    """
    Extracts a watertight mesh from an implicit function using adaptive octree construction.
    """
    # 1. Initialize Octree
    octree = Octree(bbox_min, bbox_max, min_depth)
    
    # 2. Adaptive Refinement
    if adaptive_mode:
        refine_octree(octree, func, min_depth, max_depth, angle_threshold, iso_value)
        enforce_balance_2to1(octree)
        ensure_edge_completeness(octree)
    
    # 3. Mesh Extraction
    vertices, faces = dual_contour_extract(octree, func, iso_value)
    return vertices, faces

def refine_octree(octree, func, min_depth, max_depth, angle_threshold, iso_value):
    """
    Recursively refines the octree based on implicit function properties.
    """
    for leaf in octree.leaves:
        if leaf.depth >= max_depth:
            continue
            
        # Evaluate corners
        corners = get_corner_positions(leaf)
        values, gradients = evaluate_func_batch(func, corners)
        
        # Check refinement criteria
        needs_refine = False
        
        # Criterion 1: Surface presence (sign changes)
        if has_sign_changes(values, iso_value):
            needs_refine = True
            
        # Criterion 2: Feature detection (angle threshold)
        if gradients is not None:
            normals = normalize(gradients)
            if normal_spread(normals) > angle_threshold:
                needs_refine = True
        else:
            # Estimate gradients if not provided
            gradients = estimate_gradients(func, corners, step_size=1e-5)
            normals = normalize(gradients)
            if normal_spread(normals) > angle_threshold:
                needs_refine = True
                
        # Criterion 3: Surface Propagation (Rule 3)
        if is_surface_cell(values, iso_value):
            for neighbor in leaf.face_neighbors:
                if neighbor.depth == leaf.depth and neighbor.is_branch_with_surface_descendants():
                    if leaf.depth < max_depth:
                        needs_refine = True
                        break
                        
        if needs_refine:
            subdivide_cell(leaf)
            # Recursively refine children
            for child in leaf.children:
                refine_octree(child, func, min_depth, max_depth, angle_threshold, iso_value)

def enforce_balance_2to1(octree):
    """
    Ensures no face-adjacent leaves differ by more than 1 depth level.
    """
    all_leaves = octree.get_all_leaves()
    changed = True
    while changed:
        changed = False
        for leaf in all_leaves:
            for neighbor in leaf.face_neighbors:
                if neighbor.is_leaf:
                    depth_diff = leaf.depth - neighbor.depth
                    if depth_diff > 1:
                        subdivide_cell(leaf)
                        changed = True
                        # Update all_leaves or handle dynamically
                    elif neighbor.depth - leaf.depth > 1:
                        subdivide_cell(neighbor)
                        changed = True

def ensure_edge_completeness(octree):
    """
    Ensures every sign-change edge has all 4 sharing cells present as surface leaves.
    """
    for edge in octree.sign_change_edges:
        sharing_cells = edge.get_sharing_cells()
        for cell in sharing_cells:
            if not cell.is_surface_leaf:
                if cell.depth < max_depth:
                    subdivide_cell(cell)
                # Ensure it becomes a surface leaf

def dual_contour_extract(octree, func, iso_value):
    """
    Extracts mesh using Dual Contouring.
    """
    vertices = []
    faces = []
    
    for leaf in octree.surface_leaves:
        # QEF Vertex Placement
        qef_data = collect_hermite_data(leaf, func)
        vertex = solve_qef_jacobi_svd(qef_data)
        
        # Newton Projection
        vertex = newton_project(vertex, func, iso_value)
        
        vertices.append(vertex)
        
        # Quad Generation
        quads = generate_quads(leaf, vertex)
        for quad in quads:
            faces.extend(triangulate_quad(quad))
            
    return vertices, faces
```

## Subtopics

### [Octree Balance Subdivision](847-octree-balance-subdivision.md)
The atomic step for increasing spatial resolution by splitting a node into eight children. Choose this when you need the fundamental geometric decomposition operation.

### [Error Data Generation](1324-error-data-generation.md)
Computes quantitative approximation error. Choose this when you need to quantify data loss during adaptive resolution transition.

### [Constrained Octree Refinement](2426-constrained-octree-refinement.md)
Restricts depth-disparity between face-adjacent nodes to a maximum of one. Choose this when you need to prevent excessive geometric simplification in adaptive octrees.

### [Topological Octree Construction](4698-topological-octree-construction.md)
Refines based on topological complexity to ensure manifold connectivity. Choose this when you need explicit topological criteria for refinement beyond scalar value heterogeneity.

## Operating characteristics
- **Time Complexity**: Depends on the number of leaves $L$. Refinement is $O(L)$ per level. Balancing is $O(L)$ with potential repeated passes. Extraction is $O(L)$. Total time is dominated by function evaluations at grid corners.
- **Memory Complexity**: $O(L)$ for storing octree nodes and leaf data. The octree structure itself is sparse.
- **Measured Performance**: On a sphere, adaptive refinement (depth 4->6) uses 536 vertices vs 8,600 for uniform depth 6, a 16x reduction [S1, unofficial]. On complex mechanical parts, adaptive mode is essential for quality [S1, unofficial].
- **Strengths**: Produces watertight, manifold meshes. Handles sharp features better than Marching Cubes via QEF. Adaptive resolution reduces vertex count for smooth surfaces. Supports local updates for dynamic scenes (e.g., metaballs) [S2].
- **Limitations**: Balancing is not strictly necessary for isotopy but increases complexity [S2]. Singularities may require max depth limits, sacrificing isotopy locally [S2]. Finite difference gradients are unreliable for neural SDFs; autodiff is required [S1, unofficial].
- **When to use**: Extracting meshes from implicit functions, especially SDFs from neural networks or CAD data, where manifold output and sharp feature preservation are critical.
- **When not to use**: When Hausdorff distance minimization is the primary goal (this method prioritizes isotopy) [S2]. When the implicit function is unbounded and the bounding box cannot be reasonably defined [S2].

## Background
The method is based on the `isomesh` library, which implements Dual Contouring of Hermite Data [S1, unofficial]. The theoretical foundation for isotopic approximation of implicit surfaces is provided by [S2], which guarantees isotopy between the mesh and the surface. The algorithm builds on [Adaptive Octree Construction with 2:1 Balance and Surface Propagation](830-adaptive-octree-construction-with-2-1-balance-and-surface-pr.md) by providing the specific topological criteria for refinement and the balancing mechanism [S1, unofficial]. It is designed for use with neural SDF models like DeepSDF and SDF-Diffusion [S1, unofficial].

## Sources
[S1] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S2] Isotopic approximation of implicit curves and surfaces, https://doi.org/10.1145/1057432.1057465, fetched 2026-10-29
