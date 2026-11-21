# Adaptive Dual Contouring

Also called: 3D Adaptive Dual Contouring, ADC3D, Adaptive Octree Dual Contouring, Dual Contouring with Octree Simplification and QEF Constraints, Octree-Based Dual Contouring (3D), QEF-based Octree Marching

## Overview
Adaptive Dual Contouring extracts a manifold mesh from a scalar density function by building an adaptive octree, solving Quadratic Error Functions (QEF) at nodes to determine vertex positions, and generating a mesh from the resulting constrained vertices [S1, S2, S3, unofficial]. It solves a Quadratic Error Function (QEF) at each node to determine vertex positions, then recursively collapses internal octree nodes into 'pseudo-leaf' nodes if the resulting QEF error is below a threshold, preserving sharp features defined by the sign configuration of child nodes [S1, S4, unofficial]. Unlike standard Dual Contouring which places one vertex per leaf voxel, this method reduces vertex count by merging internal nodes, producing a coarser but topologically consistent mesh suitable for real-time rendering or further processing [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A scalar density function $f(x,y,z)$ where positive values indicate solid material and negative values indicate air [S2, unofficial]; a bounding cuboid defining the extraction volume [S1, unofficial]; an iso-level (typically 0) [S1, unofficial]; a simplification tolerance threshold [S1, S3, unofficial]; and a maximum octree depth [S1, unofficial].
*   **Output:** A list of meshes (triangles) representing the isosurface [S1, unofficial].

**Data Structures**
*   **NodeData:** Contains fields: `QEF qef` (the quadratic error function data), `Vector3 vertex` (position), `int cornerEncoding` (8-bit mask of solid/empty corners), `int index` (vertex buffer index), `float size` (node dimension) [S1, unofficial].
*   **OctreeNode:** Contains fields: `type` (enum: None, Internal, Pseudo, Leaf), `position` (Vector3, world coordinates of the node's minimum corner), `size` (int, side length of the cube), `children` (array of 8 OctreeNode pointers in Z-order), `draw_info` (pointer to OctreeDrawInfo) [S3, unofficial].
*   **OctreeDrawInfo:** Contains fields: `index` (int), `corners` (int, corner encoding), `position` (Vector3), `averageNormal` (Vector3), `qef` (QEF solver instance) [S3, unofficial].
*   **QEFData:** Contains fields: `positions` (Vector3 array), `normals` (Vector3 array), `point` (Vector3, solution), `error` (float) [S2, unofficial].
*   **Lookup Tables:**
    *   `CHILD_MIN_OFFSETS`: Static Vector3 array of size 8. Indices 0-3 are X=0 plane, 4-7 are X=1 plane. Values: $(0,0,0), (0,0,1), (0,1,0), (0,1,1), (1,0,0), (1,0,1), (1,1,0), (1,1,1)$ [S2, unofficial].
    *   `CHILD_MIN_OFFSETS_REVERSE`: Static int[2,2,2] array mapping child position $(x,y,z)$ to index in parent's children array. Index 0 is $(0,0,0)$, 1 is $(0,0,1)$, 2 is $(0,1,0)$, 3 is $(0,1,1)$, 4 is $(1,0,0)$, 5 is $(1,0,1)$, 6 is $(1,1,0)$, 7 is $(1,1,1)$ [S2, unofficial].
    *   `edgevmap`: Static int[12, 2] array defining 12 edges. Rows 0-3 (x-axis): $\{0,4\}, \{1,5\}, \{2,6\}, \{3,7\}$. Rows 4-7 (y-axis): $\{0,2\}, \{1,3\}, \{4,6\}, \{5,7\}$. Rows 8-11 (z-axis): $\{0,1\}, \{2,3\}, \{4,5\}, \{6,7\}$ [S2, unofficial].
    *   Additional lookup tables for face/edge processing masks are required for mesh generation but exact values are not provided in the sources [S3, unofficial].

**Parameters**
*   `BIAS_STRENGTH`: Float constant, value $0.01f$. Weights bias terms added to QEF [S1, unofficial].
*   `BIAS_FIX`: Boolean constant, `true`. Enables addition of bias terms to QEF [S1, unofficial].
*   `CONSTRAINTS_FIX`: Boolean constant, `true`. Enables boundary constraints on QEF solution [S1, unofficial].
*   `steps`: Integer constant in QEFSolver, value $8$. Number of sampling points along an edge for zero-crossing approximation [S2, unofficial].
*   `offset`: Float constant in `CalculateSurfaceNormal`, value $0.001f$. Step size for finite difference gradient calculation [S2, unofficial].
*   `threshold`: Error threshold for simplification. If QEF error exceeds this, the node is not collapsed [S3, unofficial].
*   `randomize`: Boolean flag. If true, simplification threshold is randomized [S3, unofficial].
*   `threaded`: Controls multithreading depth. Safe values are $0$ or $1$. Values $>1$ generate $8^{threaded}$ tasks and typically degrade performance [S3, unofficial].

**Algorithm Steps**

1.  **Construct Octree:**
    Recursively subdivide the bounding volume. If a node's size is 1, evaluate the density at its 8 corners. If all corners are solid (encoding 255) or all are empty (encoding 0), discard the node (return null) [S2, S3, unofficial]. Otherwise, mark it as a Leaf and compute its vertex position [S2, unofficial]. If size > 1, create 8 children with positions offset by `CHILD_MIN_OFFSETS[i] * childSize` [S2, unofficial]. If no children are created, return null [S2, unofficial].

2.  **Solve QEF for Leaf Nodes:**
    For each leaf node:
    a.  Identify edge crossings. Iterate over the 12 edges defined by `edgevmap`. For each edge, check if the corner bits differ. If they do, approximate the crossing position.
    b.  **Approximate Crossing Position:** Sample the density function along the edge at `steps` intervals (increment $1.0/steps$). Return the position with the minimum absolute function value [S2, unofficial]. Note: This finds the minimum, not necessarily the zero crossing [S2, unofficial].
    c.  Calculate the surface normal at the crossing position using central finite differences with `offset` $0.001f$:
        $$ n_x = f(x+\delta, y, z) - f(x-\delta, y, z) $$
        $$ n_y = f(x, y+\delta, z) - f(x, y-\delta, z) $$
        $$ n_z = f(x, y, z+\delta) - f(x, y, z-\delta) $$
        $$ \mathbf{n} = \text{normalize}(-n_x, -n_y, -n_z) $$
    d.  Accumulate intersection points and normals into a QEF.
    e.  **Apply Bias:** If `BIAS_FIX` is true, calculate the mass point (average of intersections). Add the mass point to the intersections list three times, and add three normals $(\pm \text{BIAS\_STRENGTH}, 0, 0), (0, \pm \text{BIAS\_STRENGTH}, 0), (0, 0, \pm \text{BIAS\_STRENGTH})$ [S1, unofficial].
    f.  **Solve QEF:** Use a solver to find the point minimizing the error. The provided solver is a bruteforce offset search over a grid, not a linear solver [S3, unofficial].
    g.  **Apply Constraints:** If `CONSTRAINTS_FIX` is true, constrain the solution to the node bounds.
        i.  Try constraining to the 6 bounding planes (min/max x, y, z). Filter results to keep only those inside bounds.
        ii. If no valid points, try constraining to the 12 bounding edges (chaining two axis fixes). Filter results.
        iii. If no valid points, use the 8 cube corners as candidates.
        iv. Select the candidate with the lowest QEF error [S1, unofficial].

3.  **Simplify Octree (Recursive):**
    For each internal node, attempt to collapse its children into a single "Pseudo" node.
    a.  Initialize `signs` array to $\{-1, -1, -1, -1, -1, -1, -1, -1\}$ and `midSign` to $-1$ [S1, unofficial].
    b.  Sum the QEFs of all non-null children. If any child has no QEF data, abort simplification for this node [S1, unofficial].
    c.  Update `signs[i]` based on the child's corner encoding bit $i$. Update `midSign` based on bit $7-i$ [S1, unofficial].
    d.  Solve the summed QEF. If the solution is outside the node bounds, apply `ApplyConstraint` [S1, unofficial].
    e.  If the solution error exceeds `simplificationToleranceValue`, abort simplification [S1, unofficial].
    f.  Build a new `rootCorners` encoding: for each bit $i$, if `signs[i]` is $-1$, use `midSign`, else use `signs[i]` [S1, unofficial].
    g.  Delete children. Create a new `NodeData` with the summed QEF, solution vertex, `rootCorners`, and node size. Set node type to Pseudo [S1, unofficial].
    h.  *Note:* Some implementations use randomization for the threshold and check if children are Internal (if so, not collapsible) [S3, unofficial].

4.  **Generate Mesh:**
    Traverse the octree. For each leaf or pseudo node:
    a.  Determine the connectivity based on the `corners` encoding and the `draw_info` index of the node.
    b.  Use lookup tables (`cellProcFaceMask`, `cellProcEdgeMask`, etc.) to determine which triangles to generate based on the corner configuration and the configuration of neighboring nodes [S3, unofficial].
    c.  Add triangles to the index buffer using the node's vertex index [S3, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_dual_contouring(density_function, bounds, iso_level, simplification_tolerance, max_depth):
    root = OctreeNode(position=bounds.min, size=bounds.size, type=Internal)
    build_octree(root, density_function, iso_level, simplification_tolerance, max_depth)
    meshes = generate_mesh(root)
    return meshes

def build_octree(node, density_function, iso_level, simplification_tolerance, depth):
    if depth == 0:
        process_node(node, density_function, iso_level)
        return

    node.subdivide()
    child_size = node.size / 2
    live_children = 0

    for i in range(8):
        offset = CHILD_MIN_OFFSETS[i]
        child_pos = node.position + offset * child_size
        child = OctreeNode(position=child_pos, size=child_size, type=Internal)
        node.children[i] = child
        
        if build_octree(child, density_function, iso_level, simplification_tolerance, depth - 1):
            live_children += 1

    if live_children == 0:
        node = None
        return False
    
    # Attempt simplification if children exist
    simplify_node(node, density_function, simplification_tolerance)
    return True

def process_node(node, density_function, iso_level):
    if node.size == 1:
        corners = 0
        samples = []
        for i in range(8):
            pos = node.position + CHILD_MIN_OFFSETS[i]
            val = density_function(pos.x, pos.y, pos.z)
            samples.append(val)
            if val > iso_level:
                corners |= (1 << i)
        
        if corners == 0 or corners == 255:
            node = None
            return False

        node.type = Leaf
        intersections = []
        normals = []
        
        for edge_idx in range(12):
            c1, c2 = edgevmap[edge_idx]
            m1 = (corners >> c1) & 1
            m2 = (corners >> c2) & 1
            
            if m1 == m2:
                continue
            
            p1 = node.position + CHILD_MIN_OFFSETS[c1]
            p2 = node.position + CHILD_MIN_OFFSETS[c2]
            d1 = samples[c1]
            d2 = samples[c2]
            
            # Approximate crossing
            crossing = approximate_crossing(p1, p2, d1, d2)
            normal = calculate_surface_normal(crossing, density_function)
            
            intersections.append(crossing)
            normals.append(normal)

        if BIAS_FIX:
            add_bias(intersections, normals)

        qef = create_qef(intersections, normals)
        solution = solve_qef(qef, node.bounds)
        
        node.draw_info = OctreeDrawInfo(
            position=solution,
            corners=corners,
            average_normal=normalize(sum(normals)),
            qef=qef
        )
        return True
    return False

def simplify_node(node, density_function, tolerance):
    if node.type != Internal:
        return

    signs = [-1] * 8
    mid_sign = -1
    sum_qef = None

    for i in range(8):
        child = node.children[i]
        if child is None:
            continue
        if child.type == Internal:
            return # Cannot simplify if child is internal
        
        qef = child.draw_info.qef
        if qef is None:
            return # No QEF data
        
        if sum_qef is None:
            sum_qef = qef
        else:
            sum_qef += qef
            
        encoding = child.draw_info.corners
        signs[i] = (encoding >> i) & 1
        mid_sign = (encoding >> (7 - i)) & 1

    if sum_qef is None:
        return

    solution = sum_qef.solve()
    if not node.bounds.contains(solution):
        solution = apply_constraint(node.bounds, sum_qef)

    error = sum_qef.evaluate(solution)
    if error > tolerance:
        return

    root_corners = 0
    for i in range(8):
        s = signs[i] if signs[i] != -1 else mid_sign
        root_corners |= (s << i)

    node.type = Pseudo
    node.draw_info = OctreeDrawInfo(
        position=solution,
        corners=root_corners,
        average_normal=normalize(sum(c.draw_info.average_normal for c in node.children if c is not None)),
        qef=sum_qef
    )
    node.children = [None] * 8

def generate_mesh(node, indexes):
    if node is None:
        return
    
    if node.type == Internal:
        for child in node.children:
            if child is not None:
                generate_mesh(child, indexes)
        return

    # Process faces and edges to generate triangles
    # This requires complex lookup table logic based on neighbor configurations
    # Simplified representation:
    process_cell(node, indexes)

def process_cell(node, indexes):
    # Iterate over faces and edges of the cube
    # For each face/edge, determine connectivity with neighbors
    # Add triangles to indexes based on corner encoding
    pass

def approximate_crossing(p1, p2, d1, d2):
    # Linear interpolation is standard, but sources use min-value search
    min_val = float('inf')
    best_pos = p1
    increment = 1.0 / steps
    
    for t in range(steps + 1):
        pos = lerp(p1, p2, t * increment)
        val = density_function(pos.x, pos.y, pos.z)
        if abs(val) < min_val:
            min_val = abs(val)
            best_pos = pos
    return best_pos

def calculate_surface_normal(pos, density_function):
    offset = 0.001
    dx = density_function(pos.x + offset, pos.y, pos.z) - density_function(pos.x - offset, pos.y, pos.z)
    dy = density_function(pos.x, pos.y + offset, pos.z) - density_function(pos.x, pos.y - offset, pos.z)
    dz = density_function(pos.x, pos.y, pos.z + offset) - density_function(pos.x, pos.y, pos.z - offset)
    return normalize(Vector3(-dx, -dy, -dz))

def add_bias(intersections, normals):
    mass_point = sum(intersections) / len(intersections)
    intersections.extend([mass_point, mass_point, mass_point])
    normals.extend([
        Vector3(BIAS_STRENGTH, 0, 0),
        Vector3(0, BIAS_STRENGTH, 0),
        Vector3(0, 0, BIAS_STRENGTH)
    ])

def apply_constraint(bounds, qef):
    candidates = []
    # Try 6 planes
    for axis in range(3):
        for limit in [bounds.min[axis], bounds.max[axis]]:
            constrained_qef = qef.fix_axis(axis, limit)
            sol = constrained_qef.solve()
            if bounds.contains(sol):
                candidates.append(sol)
    
    if not candidates:
        # Try 12 edges
        for axis1 in range(3):
            for axis2 in range(axis1 + 1, 3):
                for l1 in [bounds.min[axis1], bounds.max[axis1]]:
                    for l2 in [bounds.min[axis2], bounds.max[axis2]]:
                        constrained_qef = qef.fix_axis(axis1, l1).fix_axis(axis2, l2)
                        sol = constrained_qef.solve()
                        if bounds.contains(sol):
                            candidates.append(sol)
                            
    if not candidates:
        # Try 8 corners
        candidates = bounds.get_corners()
        
    best = None
    best_err = float('inf')
    for c in candidates:
        err = qef.evaluate(c)
        if err < best_err:
            best_err = err
            best = c
    return best
```

## Subtopics

### [Dual Contouring with Octree Simplification](3685-dual-contouring-with-octree-simplification.md)
Dual Contouring with Octree Simplification is a surface extraction method that combines adaptive octree traversal with topological simplification to generate a manifold mesh from a scalar field. It solves a Quadratic Error Function (QEF) at each node to determine vertex positions, then recursively collapses internal octree nodes into 'pseudo-leaf' nodes if the resulting QEF error is below a threshold, preserving sharp features defined by the sign configuration of child nodes. Unlike standard Dual Contouring which places one vertex per leaf voxel, this method reduces vertex count by merging internal nodes, producing a coarser but topologically consistent mesh suitable for real-time rendering or further processing.

## Operating characteristics
*   **Time Complexity:** Depends on the octree depth and the complexity of the isosurface. Each leaf node requires QEF solving, which is $O(E)$ where $E$ is the number of edge crossings (max 12). Simplification is recursive.
*   **Memory Complexity:** Proportional to the number of octree nodes. Internal nodes store pointers to children; leaf/pseudo nodes store QEF data.
*   **Performance:** Threading is only effective for `threaded` values 0 or 1; higher values degrade performance [S3, unofficial]. The bruteforce QEF solver is slow and should be replaced with a proper linear solver for production [S3, unofficial].
*   **Strengths:** Produces manifold meshes with fewer vertices than Marching Cubes by simplifying the octree. Preserves sharp features better than linear interpolation methods [S2, unofficial].
*   **Limitations:** The provided QEF solver is inaccurate and slow [S3, unofficial]. 2D implementations in the source code do not connect properly [S3, unofficial]. High error tolerance in QEF solving causes sharp feature loss and mesh deformation [S3, unofficial].
*   **When to use:** When a manifold mesh with reduced vertex count is needed from a scalar field, and sharp features must be preserved.
*   **When not to use:** When high performance is critical and a proper QEF solver is not available, or when non-manifold geometry is acceptable.

## Background
Adaptive Dual Contouring is an extension of Dual Contouring that uses an octree for adaptive subdivision and simplification. It was implemented in open-source projects such as `Dual-contouring` by nivshalomlom [S1, unofficial], `DualContouring-3` by Alan-Baylis [S2, unofficial], and `isosurface` by Lin20 [S3, unofficial]. The method builds on the principles of Dual Contouring [Parent](1271-dual-contouring-with-octree.md).

## Sources
[S1] nivshalomlom/Dual-contouring, https://github.com/nivshalomlom/Dual-contouring, fetched 2026-10-26 (unofficial)
[S2] Alan-Baylis/DualContouring-3, https://github.com/Alan-Baylis/DualContouring-3, fetched 2026-10-26 (unofficial)
[S3] isosurface, https://github.com/Lin20/isosurface, fetched 2026-10-29 (unofficial)
