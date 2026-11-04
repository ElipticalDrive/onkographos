# Dual Contouring Leaf Processing

Also called: Leaf Voxel Contouring, QEF Vertex Placement

## Overview
Dual Contouring Leaf Processing is the base-case algorithm for generating a mesh vertex within a leaf voxel of an octree during isosurface extraction. It solves the problem of determining a single representative vertex position and normal for the implicit surface intersection contained within a unit-sized voxel. It differs from its siblings by operating exclusively on leaf nodes of size 1, relying on edge intersection detection and Quadratic Error Function (QEF) minimization, whereas variants like [Dual Contouring with Octree](1271-dual-contouring-with-octree.md) handle hierarchical data structures and [Simplified Dual Contouring](1580-simplified-dual-contouring.md) may use heuristic approximations.

## How it works
**Inputs:** A leaf voxel node from an octree, representing a unit cube in world space. The voxel contains an implicit function $f(\mathbf{x})$ (e.g., density).
**Outputs:** An `OctreeDrawInfo` structure containing the vertex position $\mathbf{p}$, normal $\mathbf{n}$, and metadata, or `nullptr` if the voxel is empty or fully inside.

**Data Structures:**
*   **Corners Bitmask:** An 8-bit integer where bit $i$ is set if corner $i$ is `MATERIAL_SOLID` ($f < 0$) and clear if `MATERIAL_AIR` ($f \ge 0$).
*   **QEF Solver:** Accumulates linear constraints $A_i \mathbf{x} = b_i$ derived from surface normals and intersection points.
*   **Edge Map:** A predefined mapping from edge index (0-11) to the two corner indices (0-7) it connects.

**Steps:**
1.  **Validation:** If the voxel size is not 1, return `nullptr` [S1, unofficial].
2.  **Corner Classification:** Iterate over the 8 corners of the voxel. Evaluate $f(\mathbf{c}_i)$. If $f(\mathbf{c}_i) < 0$, mark as `MATERIAL_SOLID`; otherwise `MATERIAL_AIR`. Pack these into the corners bitmask [S1, unofficial].
3.  **Empty/Full Check:** If the bitmask is `0x00` (all AIR) or `0xFF` (all SOLID), the voxel contains no surface. Delete the leaf and return `nullptr` [S1, unofficial].
4.  **Initialization:** Initialize `edgeCount = 0`, `averageNormal = (0,0,0)`, and an empty QEF solver [S1, unofficial].
5.  **Edge Intersection:** Loop over the 12 edges of the cube. For each edge $j$:
    a. Retrieve corner indices $c_1, c_2$ from the edge map.
    b. Extract materials $m_1, m_2$ from the bitmask. If $m_1 == m_2$, skip (no zero crossing) [S1, unofficial].
    c. Compute endpoints $\mathbf{p}_1, \mathbf{p}_2$.
    d. Find intersection point $\mathbf{p}$ using `ApproximateZeroCrossingPosition` [S1, unofficial].
    e. Compute normal $\mathbf{n} = \nabla f(\mathbf{p})$ using `CalculateSurfaceNormal` [S1, unofficial].
    f. Add constraint $(\mathbf{n}, \mathbf{p} \cdot \mathbf{n})$ to the QEF solver.
    g. Accumulate $\mathbf{n}$ to `averageNormal`.
    h. Increment `edgeCount`. Stop if `edgeCount` exceeds `MAX_CROSSINGS` [S1, unofficial].
6.  **QEF Solution:** Solve the QEF to find optimal position $\mathbf{p}_{qef}$ [S1, unofficial].
7.  **Boundary Check:** If $\mathbf{p}_{qef}$ lies outside the voxel bounds $[\mathbf{min}, \mathbf{max}]$, replace $\mathbf{p}_{qef}$ with the QEF mass point (centroid of constraints) to prevent out-of-bounds artifacts [S1, unofficial].
8.  **Normal Calculation:** Compute final normal $\mathbf{n}_{final} = \text{normalize}(\text{averageNormal} / \text{edgeCount})$ [S1, unofficial].
9.  **Result:** Create `OctreeDrawInfo` with position $\mathbf{p}_{final}$, normal $\mathbf{n}_{final}$, type `Node_Leaf`, and return the leaf [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_leaf(voxel_node):
    # 1. Validation
    if voxel_node.size != 1:
        return None

    # 2. Corner Classification
    corners_bitmask = 0
    for i in range(8):
        corner_pos = voxel_node.get_corner_position(i)
        density = density_func(corner_pos)
        if density < 0:
            corners_bitmask |= (1 << i)

    # 3. Empty/Full Check
    if corners_bitmask == 0 or corners_bitmask == 255:
        return None

    # 4. Initialization
    edge_count = 0
    average_normal = vec3(0, 0, 0)
    qef_solver = QefSolver()

    # 5. Edge Intersection
    for edge_idx in range(12):
        if edge_count >= MAX_CROSSINGS:
            break
        
        c1, c2 = edge_map[edge_idx]
        m1 = (corners_bitmask >> c1) & 1
        m2 = (corners_bitmask >> c2) & 1

        if m1 == m2:
            continue

        p1 = voxel_node.get_corner_position(c1)
        p2 = voxel_node.get_corner_position(c2)
        
        # 5d. Intersection
        p = approximate_zero_crossing(p1, p2)
        
        # 5e. Normal and QEF
        n = calculate_surface_normal(p)
        qef_solver.add_constraint(n, dot(n, p))
        average_normal += n
        edge_count += 1

    # 6. QEF Solution
    qef_position, qef_error = qef_solver.solve()

    # 7. Boundary Check
    if not is_inside_voxel(qef_position, voxel_node):
        qef_position = qef_solver.get_mass_point()

    # 8. Normal Calculation
    if edge_count > 0:
        final_normal = normalize(average_normal / edge_count)
    else:
        final_normal = vec3(0, 0, 1) # [G] Fallback for degenerate case

    # 9. Result
    draw_info = OctreeDrawInfo()
    draw_info.position = qef_position
    draw_info.normal = final_normal
    draw_info.type = Node_Leaf
    draw_info.corners = corners_bitmask
    return draw_info

def approximate_zero_crossing(p0, p1):
    min_val = 100000.0
    best_t = 0.0
    steps = 8
    increment = 1.0 / steps
    current_t = 0.0
    
    while current_t <= 1.0:
        p = p0 + (p1 - p0) * current_t
        density = abs(density_func(p))
        if density < min_val:
            min_val = density
            best_t = current_t
        current_t += increment
        
    return p0 + (p1 - p0) * best_t

def calculate_surface_normal(p):
    h = 0.001
    dx = density_func(p + vec3(h, 0, 0)) - density_func(p - vec3(h, 0, 0))
    dy = density_func(p + vec3(0, h, 0)) - density_func(p - vec3(0, h, 0))
    dz = density_func(p + vec3(0, 0, h)) - density_func(p - vec3(0, 0, h))
    return normalize(vec3(dx, dy, dz))
```

**Parameters:**
*   `MAX_CROSSINGS`: Maximum number of intersections to process per voxel. Not given in the sources.
*   `QEF_SWEEPS`: Number of sweeps for the QEF solver. Not given in the sources.
*   `QEF_ERROR`: Convergence threshold for the QEF solver. Not given in the sources.
*   `H`: Finite difference step size for normal calculation. Value: $0.001$ [S1, unofficial].
*   `steps`: Discretization steps for zero-crossing approximation. Value: $8$ [S1, unofficial].

**Edge Cases:**
*   **No Intersections:** Handled by the bitmask check (return `nullptr`) [S1, unofficial].
*   **Out-of-Bounds Vertex:** If the QEF solution falls outside the voxel, the mass point is used [S1, unofficial].
*   **Degenerate Voxel:** If `edgeCount` is 0 after loop (should be caught by bitmask, but guarded), a fallback normal is used [G].

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(1)$ per leaf voxel, as it processes a fixed 8 corners and 12 edges.
*   **Memory Complexity:** $O(1)$ auxiliary memory per leaf.
*   **Strengths:** Simple to implement; robust for leaf-level processing in adaptive octrees.
*   **Limitations:** The zero-crossing approximation is coarse (8 steps), potentially leading to inaccurate normals or positions compared to analytical root finding. The QEF solver details are abstracted away.
*   **When to use:** As the base case for recursive dual contouring on octrees.
*   **When not to use:** For uniform grids where simpler marching cubes may be faster, or when high-precision surface normals are required without post-processing.

## Background
The method is derived from the Dual Contouring algorithm by Yu [2005], specifically the leaf processing stage implemented in the `nickgildea/DualContouringSample` repository. It addresses the need to extract a single vertex per leaf voxel in an adaptive octree structure, minimizing the QEF formed by surface constraints.

## Sources
[S1] nickgildea/DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
