# Incremental Contouring via Power Diagrams

Also called: Power Diagram Surface Extraction, Regular Triangulation Contouring, Incremental Power Diagram Extraction, Power Diagram-based Isosurface Extraction

## Overview
Incremental Contouring via Power Diagrams is a progressive isosurface extraction algorithm that constructs a mesh from a Signed Distance Field (SDF) by iteratively refining a weighted triangulation (regular triangulation or power diagram) [S1][S2, unofficial]. Unlike static methods like Marching Cubes or Dual Contouring that operate on a fixed grid, this method starts with a coarse set of samples and incrementally inserts new vertices at locations of high geometric deviation, updating the local triangulation and surface mesh without global recomputation [S1]. It leverages the dual relationship between power diagrams and regular Delaunay tetrahedralizations to ensure topological correctness and geometric fidelity, particularly for complex or high-genus surfaces [S1].

## How it works
### Inputs and Outputs
**Inputs:**
1.  A Signed Distance Function (SDF) $\phi(\mathbf{x})$ and its gradient $\nabla \phi(\mathbf{x})$ [S1].
2.  An initial uniform grid of resolution $N \times N \times N$ (default $N=10$) [S2, unofficial].
3.  Parameters: `max_refinement` (default 1000), `is_delaunay` (boolean, default false), and optionally a target point count $k_{max}$ [S1][S2, unofficial].

**Outputs:**
1.  A triangular mesh $V_{mt}, F_{mt}$ representing the zero-level set of the SDF [S2, unofficial].

### Data Structures
The algorithm maintains a weighted triangulation $T$ (using CGAL's `Regular_triangulation_3` or equivalent) [S2, unofficial].

**Weighted Point:**
A point $\mathbf{p}$ with an associated weight $w$.
*   If `is_delaunay` is true: $w = 0$.
*   If `is_delaunay` is false (Power Diagram mode): $w = \phi(\mathbf{p})^2$ [S2, unofficial].

**Candidate:**
Used in the refinement priority queue.
*   `m_v0, m_v1, m_v2, m_v3`: Handles to the four vertices of the tetrahedron containing the sign flip [S2, unofficial].
*   `m_point`: The 3D position of the dual point (circumcenter) of the tetrahedron [S2, unofficial].
*   `m_score`: A priority value (e.g., deviation metric) for sorting the queue [S2, unofficial].

**Site Classification:**
Sites are categorized into four types to guide surface extraction:
1.  $P^-$: Interior points ($\phi < 0$).
2.  $P^+$: Exterior points ($\phi > 0$).
3.  $P^\perp_-$: Surface projections of interior points.
4.  $P^\perp_+$: Surface projections of exterior points [S1].

### Algorithm Steps

#### 1. Initialization
1.  **Normalize Input Mesh:** If the SDF is derived from a mesh, normalize vertices $\mathbf{V}$ to the range $[-1, 1]$ centered at origin:
    $$ \mathbf{V} \leftarrow \frac{\mathbf{V} - \min(\mathbf{V})}{\max(\mathbf{V}) - \min(\mathbf{V})} - 0.5 \times \frac{\max(\mathbf{V}) - \min(\mathbf{V})}{\max(\mathbf{V}) - \min(\mathbf{V})} \times 0.5 \times \dots $$
    *Implementation Note:* The source code normalizes by subtracting min, dividing by max, centering, and scaling by $0.75/0.5$ [S2, unofficial].
2.  **Sample SDF:** Evaluate $\phi$ on a regular grid of resolution $N$. Store points $\mathbf{p}_i$ and values $\phi_i$ in matrix $G$ [S2, unofficial].
3.  **Construct Initial Triangulation:**
    *   For each grid point $\mathbf{p}_i$, compute weight $w_i = \phi_i^2$ (if `is_delaunay` is false) or $0$ (if true).
    *   Insert weighted points $(\mathbf{p}_i, w_i)$ into $T$ [S2, unofficial].
    *   *Optional:* If using the full Power Diagram method, compute surface projections for initial points and insert them as additional sites [S1].
4.  **Extract Initial Surface:**
    *   Iterate over all tetrahedra in $T$.
    *   Identify tetrahedra containing sites of different types (e.g., $P^-$ and $P^+$) [S1].
    *   For each edge connecting different site types, compute a dual vertex $\mathbf{v}_{dual}$ using the rules below.
    *   Connect dual vertices to form the initial mesh $M$ [S1].

#### 2. Deviation Metric Calculation
For each triangle $\Delta$ in the current mesh $M$, calculate the deviation $\delta(\Delta)$ from the ideal isosurface alignment:
$$ \delta(\Delta) \approx \sum_{i=1}^3 A_i \left| \nabla \phi(\mathbf{c}_i) - (\nabla \phi(\mathbf{c}_i) \cdot \hat{\mathbf{n}}) \hat{\mathbf{n}} \right| $$
where:
*   $\hat{\mathbf{n}}$ is the triangle normal.
*   The triangle is partitioned into three sub-triangles by angle bisectors.
*   $A_i$ is the area of the $i$-th sub-triangle.
*   $\mathbf{c}_i$ is the centroid of the $i$-th sub-triangle.
*   $\nabla \phi(\mathbf{c}_i)$ is the SDF gradient at the centroid [S1].

Store triangles in a max-heap $Q$ ordered by $\delta$ [S1].

#### 3. Incremental Refinement Loop
Repeat up to `max_refinement` times or until $Q$ is empty/deviation is negligible:

1.  **Select Candidate:** Pop the triangle $\Delta_c$ with the maximum deviation $\delta_c$ from $Q$ [S1].
2.  **Identify Insertion Point:**
    *   Determine the dual point (circumcenter) $\mathbf{p}_{dual}$ of the tetrahedron in $T$ that contains $\Delta_c$ [S2, unofficial].
    *   *Power Diagram Mode:* Calculate the weight for the new point. If using the specific "Incremental Contouring" implementation from [S2, unofficial], the weight is computed as:
        $$ w_{dual} = \frac{1}{4} \sum_{j=0}^3 \left( \| \mathbf{p}_{dual} - \mathbf{p}_j \|^2 - w_j \right) $$
        where $\mathbf{p}_j$ are the vertices of the tetrahedron and $w_j$ their weights [S2, unofficial].
    *   *Alternative Strategy (S1):* Insert the dual point $\mathbf{p}_c$ and its surface projection $\mathbf{q}_c$ as new sites [S1].
3.  **Insert Point:**
    *   Insert the weighted point $(\mathbf{p}_{dual}, w_{dual})$ into $T$.
    *   This triggers a localized update of the triangulation [S1].
4.  **Update State:**
    *   Assign a unique index to the new vertex.
    *   Find all incident cells (tetrahedra) to the new vertex [S2, unofficial].
5.  **Re-evaluate Neighbors:**
    *   For each incident tetrahedron, check if it contains a sign flip in SDF values (vertices with $\phi \ge 0$ and $\phi < 0$) [S2, unofficial].
    *   If a sign flip exists, compute its dual point, create a `Candidate` object with the tetrahedron handles, dual point, and score, and push it to $Q$ [S2, unofficial].
    *   *Note:* If `is_delaunay` is true, skip weight calculation and use $w=0$ [S2, unofficial].

#### 4. Final Surface Extraction
1.  **Extract Dual Contour:**
    *   Iterate over all finite edges in the final triangulation $T$.
    *   For each edge with endpoints having opposite SDF signs, traverse adjacent cells to form a facet (contour cell handles) [S2, unofficial].
2.  **Map to Dual Vertices:**
    *   For each facet, map unique cell handles to dual vertex indices.
    *   Calculate dual vertex positions using the triangulation's dual function [S2, unofficial].
3.  **Triangulate Facets:**
    *   For each polygonal facet, create a central vertex by averaging the facet's vertices.
    *   Connect the central vertex to each edge of the polygon to form triangles [S2, unofficial].
4.  **Marching Tetrahedra (Primal):**
    *   Alternatively, extract the primal contouring mesh by calling a marching tetrahedra routine on the final triangulation with threshold 0.0 to generate the final vertex set $V_{mt}$ and face set $F_{mt}$ [S2, unofficial].

### Dualization Rules for Surface Extraction (S1 Method)
When extracting the mesh from the regular Delaunay tetrahedralization $D$, for an edge connecting vertices $v_1$ and $v_2$ of different types:
*   If $v_1 \in P^-$ and $v_2 \in P^\perp_-$: Vertex is $v_2$.
*   If $v_1 \in P^+$ and $v_2 \in P^\perp_+$: Vertex is $v_2$.
*   If $v_1 \in P^-$ and $v_2 \in P^+$: Vertex is $\frac{v_1 \phi(v_2) - v_2 \phi(v_1)}{\phi(v_2) - \phi(v_1)}$.
*   If $v_1 \in P^\perp_-$ and $v_2 \in P^\perp_+$: Vertex is $v_1 + v_2$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def incremental_contouring_via_power_diagrams(sdf_function, gradient_function, N=10, max_refinement=1000, is_delaunay=False):
    """
    Extracts isosurface from SDF using incremental power diagram refinement.
    
    Args:
        sdf_function: Callable returning signed distance value.
        gradient_function: Callable returning gradient vector.
        N: Initial grid resolution per dimension.
        max_refinement: Max number of refinement steps.
        is_delaunay: If True, use standard Delaunay (w=0); else Power Diagram (w=d^2).
        
    Returns:
        vertices, faces: Final mesh.
    """
    # 1. Initialization
    # Sample SDF on regular grid
    grid_points = []
    grid_weights = []
    for x in range(N):
        for y in range(N):
            for z in range(N):
                # Normalize coordinates to [-1, 1]
                x_norm = (x / (N - 1)) * 2 - 1
                y_norm = (y / (N - 1)) * 2 - 1
                z_norm = (z / (N - 1)) * 2 - 1
                
                p = Vector3(x_norm, y_norm, z_norm)
                phi = sdf_function(p.x, p.y, p.z)
                
                if is_delaunay:
                    w = 0.0
                else:
                    w = phi * phi
                
                grid_points.append(p)
                grid_weights.append(w)

    # Initialize triangulation T with grid points
    T = RegularTriangulation()
    for i, p in enumerate(grid_points):
        T.insert(WeightedPoint(p, grid_weights[i]))

    # 2. Initial Surface Extraction (Simplified for brevity, assumes dual contouring logic)
    # In practice, this involves iterating tetrahedra, finding sign flips,
    # and constructing initial mesh M.
    M = extract_initial_mesh(T, sdf_function)
    
    # 3. Refinement Loop
    Q = MaxHeap() # Priority queue for triangles by deviation
    
    # Calculate initial deviations
    for triangle in M:
        dev = calculate_deviation(triangle, sdf_function, gradient_function)
        Q.push(triangle, dev)

    for step in range(max_refinement):
        if Q.empty():
            break
            
        # Pop highest deviation triangle
        best_triangle, best_dev = Q.pop()
        
        # Find the tetrahedron in T containing this triangle
        tet = T.find_tetrahedron_containing(best_triangle)
        if tet is None:
            continue
            
        # Calculate dual point (circumcenter)
        dual_point = T.dual(tet)
        
        # Calculate weight for new point
        if is_delaunay:
            new_weight = 0.0
        else:
            # Formula from [S2, unofficial]
            sum_val = 0.0
            for vertex_handle in tet.vertices():
                v_pos = T.position(vertex_handle)
                v_weight = T.weight(vertex_handle)
                dist_sq = (dual_point - v_pos).length_squared()
                sum_val += (dist_sq - v_weight)
            new_weight = sum_val / 4.0
            
        # Insert new point into triangulation
        new_vertex_handle = T.insert(WeightedPoint(dual_point, new_weight))
        
        # Update mesh M locally around new_vertex_handle
        # This involves re-triangulating affected regions
        M = update_mesh_after_insertion(M, T, new_vertex_handle, sdf_function)
        
        # Re-evaluate neighbors for refinement candidates
        incident_tets = T.incident_tetrahedra(new_vertex_handle)
        for inc_tet in incident_tets:
            if contains_sign_flip(inc_tet, sdf_function):
                inc_dual = T.dual(inc_tet)
                inc_dev = calculate_deviation_for_tet(inc_tet, sdf_function, gradient_function)
                candidate = Candidate(inc_tet, inc_dual, inc_dev)
                Q.push(candidate, inc_dev)

    # 4. Final Extraction
    # Extract primal contouring mesh via marching tets on final T
    vertices, faces = marching_tetrahedra(T, threshold=0.0)
    
    return vertices, faces

def calculate_deviation(triangle, sdf_func, grad_func):
    """
    Calculates gradient-alignment deviation for a triangle.
    """
    n_hat = triangle.normal()
    # Partition triangle into 3 sub-triangles via angle bisectors
    sub_triangles = partition_by_angle_bisectors(triangle)
    
    total_dev = 0.0
    for sub_tri in sub_triangles:
        c_i = sub_tri.centroid()
        grad = grad_func(c_i.x, c_i.y, c_i.z)
        
        # Orthogonal component of gradient
        proj = dot(grad, n_hat) * n_hat
        ortho = grad - proj
        
        area_i = sub_tri.area()
        total_dev += area_i * ortho.length()
        
    return total_dev

def contains_sign_flip(tet, sdf_func):
    """
    Checks if a tetrahedron has vertices with opposite SDF signs.
    """
    signs = []
    for vh in tet.vertices():
        p = tet.position(vh)
        phi = sdf_func(p.x, p.y, p.z)
        signs.append(1 if phi >= 0.0 else -1)
        
    return not all(s == signs[0] for s in signs)
```

### Edge Cases and Pitfalls
1.  **Geometric Instabilities:** Direct dualization of power diagrams can produce zigzag artifacts. The method avoids this by extracting the mesh from the regular Delaunay tetrahedralization (the dual of the power diagram) rather than the power diagram itself [S1].
2.  **Neural SDF Gradients:** Neural SDFs (e.g., SIREN, VolSDF) do not guarantee unit-length gradients, making surface projections inaccurate. The method remains robust because adaptive sampling prioritizes the zero-level set, balancing approximation errors [S1].
3.  **Gradient-Alignment Metric Bias:** The deviation metric may favor triangles far from the zero-level set if they are normal-aligned. This is mitigated by the surface projection step, which keeps triangles close to the surface, and by adjacent high-energy triangles triggering insertion [S1].
4.  **Sign Flip Detection:** Must check all four vertices of a tetrahedron. The implementation uses $\phi \ge 0.0$ for the positive threshold [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Scales approximately linearly with the number of points $O(n)$ due to the localized update strategy of the Delaunay triangulation [S1].
*   **Memory Complexity:** Proportional to the number of vertices and tetrahedra in the triangulation.
*   **Performance:** Outperforms Marching Cubes, Dual Contouring, and RFTA in reconstruction quality (Chamfer Distance) for both low-genus and high-genus models at comparable or lower sample counts [S1].
*   **Strengths:**
    *   Adaptive resolution: Focuses sampling on complex features.
    *   Topology preservation: Successfully handles high-genus models and intricate cavities [S1].
    *   Incremental: Supports batch processing for neural SDFs to reduce query overhead [S1].
*   **Limitations:**
    *   Requires SDF gradients for the deviation metric and projection points [S1].
    *   More complex to implement than uniform grid methods due to dynamic triangulation management [S2, unofficial].

## Background
The method was introduced in "Power Diagram Enhanced Adaptive Isosurface Extraction From Signed Distance Fields" (2026) [S1]. It builds on the theoretical foundation that power diagrams with sphere sites (centers at samples, radii equal to distances) form boundaries that do not intersect the spheres, providing a valid surface approximation [S1]. The implementation details and code repository were provided by Max Kohlbrenner (2025) [S2, unofficial]. It addresses limitations of static methods like Marching Cubes (aliasing, topology issues) and other adaptive methods like McGrids (which relies only on sign changes) [S1].

## Sources
[S1] Power Diagram Enhanced Adaptive Isosurface Extraction From Signed Distance Fields., https://doi.org/10.1109/tvcg.2026.3685989, fetched 2026-10-26
[S2] maxkohlbrenner/sdf-weighted-delaunay, https://github.com/maxkohlbrenner/sdf-weighted-delaunay, fetched 2026-10-27 (unofficial)
