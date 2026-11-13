# Hermite Dual Marching Cubes

Also called: Dual Marching Cubes with Hermite data, Hermite Dual Contouring, Hermite extension to Dual MC, Nielson's Dual Marching Cubes with QEF

## Overview
Hermite Dual Marching Cubes is an isosurface extraction method that combines the topological guarantees of Dual Marching Cubes with the feature-preserving capabilities of Dual Contouring. It solves the problem of non-manifold vertices and edges that arise in standard Dual Contouring when processing ambiguous sign configurations on uniform grids, while simultaneously reproducing sharp features that standard Marching Cubes fails to capture. Unlike standard Dual Contouring, which may produce non-manifold topology, this method always produces a topological manifold and can reproduce sharp features by incorporating Hermite data (normals) into the vertex placement process.

## How it works
The method operates on a uniform grid where each cell contains scalar values at its vertices and Hermite data (intersection points and normal vectors) on its edges. The output is a quadrilateral mesh that is dual to the Marching Cubes surface.

**Inputs:**
*   A uniform 3D grid of scalar field values at vertices.
*   Hermite data on grid edges: for each edge crossing the isosurface, the intersection point $\mathbf{p}$ and the surface normal $\mathbf{n}$ at that point.

**Outputs:**
*   A set of vertices $\mathbf{V}$ and a set of quadrilateral faces $\mathbf{F}$.

**Data Structures:**
*   **Grid:** A 3D array of scalar values.
*   **Edge Data:** For each grid edge, store the intersection point $\mathbf{p}$ and normal $\mathbf{n}$ if the edge crosses the isosurface.
*   **Lookup Table:** A modified Marching Cubes lookup table [26] that identifies cycles of edges for ambiguous cases. Standard cases map to triangles; ambiguous cases map to cycles of edges.
*   **Vertex Map:** A mapping from each identified cycle (or standard edge) to a single vertex position $\mathbf{v}$.

**Steps:**

1.  **Identify Sign-Change Edges:** For each cell in the grid, determine which edges cross the isosurface (i.e., connect vertices with opposite scalar signs).
2.  **Determine Topology:** For each cell, use the modified Marching Cubes lookup table to determine the contour topology.
    *   If the case is non-ambiguous, the table returns a set of triangles defined by the midpoints of the sign-change edges.
    *   If the case is ambiguous, the table returns one or more cycles of edges. Each cycle represents a contour component within the cell.
3.  **Construct Quadratic Error Functions (QEF):** For each unique vertex to be placed (associated with a cycle or a standard edge), collect all Hermite data from the edges associated with that vertex.
    *   Let $E_v$ be the set of edges associated with vertex $v$.
    *   For each edge $e \in E_v$, let $\mathbf{p}_e$ be the intersection point and $\mathbf{n}_e$ be the normal.
    *   Construct the QEF for vertex $v$:
        $$ Q_v(\mathbf{x}) = \sum_{e \in E_v} ((\mathbf{x} - \mathbf{p}_e) \cdot \mathbf{n}_e)^2 $$
    *   This minimizes the distance from $\mathbf{x}$ to the planes defined by the Hermite data.
4.  **Minimize QEF:** Solve for the vertex position $\mathbf{v}$ that minimizes $Q_v(\mathbf{x})$. This is a linear least-squares problem.
    *   The gradient is $\nabla Q_v(\mathbf{x}) = 2 \sum_{e \in E_v} ((\mathbf{x} - \mathbf{p}_e) \cdot \mathbf{n}_e) \mathbf{n}_e$.
    *   Setting the gradient to zero yields the linear system:
        $$ \left( \sum_{e \in E_v} \mathbf{n}_e \mathbf{n}_e^T \right) \mathbf{x} = \sum_{e \in E_v} (\mathbf{n}_e^T \mathbf{p}_e) \mathbf{n}_e $$
    *   Let $A_v = \sum_{e \in E_v} \mathbf{n}_e \mathbf{n}_e^T$ and $\mathbf{b}_v = \sum_{e \in E_v} (\mathbf{n}_e^T \mathbf{p}_e) \mathbf{n}_e$.
    *   Solve $A_v \mathbf{x} = \mathbf{b}_v$ for $\mathbf{x}$. If $A_v$ is singular (e.g., all normals are parallel), use a pseudo-inverse or fallback to the centroid of intersection points [G].
5.  **Generate Polygons:** For each edge in the grid that crosses the isosurface, identify the four adjacent cells sharing that edge.
    *   Retrieve the vertex associated with that edge from each of the four cells.
    *   Connect these four vertices to form a quadrilateral face.
    *   This creates a quadrilateral surface dual to the Marching Cubes surface.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hermite_dual_marching_cubes(grid, edge_hermite_data):
    """
    grid: 3D array of scalar values at vertices.
    edge_hermite_data: dict mapping edge_id to (point, normal).
    Returns: vertices list, faces list (quads).
    """
    vertices = {}  # Map from (cell_id, cycle_id) to vertex position
    faces = []
    
    # Step 1 & 2: Identify edges and determine topology per cell
    for cell in grid.cells:
        sign_change_edges = get_sign_change_edges(cell, grid)
        topology = lookup_modified_mc_table(cell.sign_pattern)
        
        if topology.type == 'triangle':
            # Standard case: triangles are formed by midpoints of edges
            # Note: Dual MC typically outputs quads. For non-ambiguous cases,
            # the "cycle" is effectively the set of edges forming the triangle.
            # However, standard Dual MC often handles ambiguous cases with cycles
            # and non-ambiguous cases with single vertices per edge or similar.
            # Based on [S2, unofficial], polygons are generated by connecting vertices associated
            # with each sign-change edge across four adjacent cells.
            # This implies every sign-change edge gets a vertex.
            pass # Handled in Step 3/4 via edge association
        
        elif topology.type == 'cycle':
            # Ambiguous case: multiple vertices per cell
            for cycle in topology.cycles:
                cycle_id = get_cycle_id(cycle)
                vertices[(cell.id, cycle_id)] = None # Placeholder

    # Step 3 & 4: Construct and minimize QEF for each vertex
    for key, _ in vertices.items():
        cell_id, cycle_id = key
        cell = grid.get_cell(cell_id)
        
        # Collect Hermite data for edges associated with this vertex
        # If cycle_id is None, it's a standard edge vertex.
        # If cycle_id exists, it's a vertex for that cycle.
        if cycle_id is None:
            # Find the edge associated with this cell and vertex position?
            # In Dual MC, each sign-change edge is associated with exactly one vertex.
            # We need to map edges to vertices.
            # Let's assume we iterate over all sign-change edges in the grid
            # and assign them to vertices.
            pass
        
        # Re-structure: Iterate over all sign-change edges in the grid
        # and assign them to vertices based on the lookup table results.
        
    # Revised Step 3 & 4:
    edge_to_vertex_map = {} # Map edge_id to (cell_id, cycle_id)
    
    for cell in grid.cells:
        sign_change_edges = get_sign_change_edges(cell, grid)
        topology = lookup_modified_mc_table(cell.sign_pattern)
        
        if topology.type == 'triangle':
            # Each edge in the triangle is associated with a vertex.
            # In Dual MC, non-ambiguous cases are often treated as single cycles
            # or the vertices are placed at edge midpoints if no Hermite data.
            # With Hermite data, we still use QEF.
            for edge in topology.edges: # Assuming topology returns edges
                edge_to_vertex_map[edge.id] = (cell.id, None)
        elif topology.type == 'cycle':
            for cycle in topology.cycles:
                cycle_id = get_cycle_id(cycle)
                for edge in cycle.edges:
                    edge_to_vertex_map[edge.id] = (cell.id, cycle_id)
    
    # Now compute vertex positions
    # Group edges by their target vertex (cell_id, cycle_id)
    vertex_edges = {} # (cell_id, cycle_id) -> list of edge_ids
    for edge_id, vertex_key in edge_to_vertex_map.items():
        if vertex_key not in vertex_edges:
            vertex_edges[vertex_key] = []
        vertex_edges[vertex_key].append(edge_id)
        
    for vertex_key, edge_ids in vertex_edges.items():
        cell_id, cycle_id = vertex_key
        A = zero_matrix(3, 3)
        b = zero_vector(3)
        
        for edge_id in edge_ids:
            p, n = edge_hermite_data[edge_id]
            # QEF term: ((x - p) . n)^2
            # Gradient contribution: 2 * ((x - p) . n) * n
            # A += n * n^T
            # b += (p . n) * n
            A += outer_product(n, n)
            b += dot(n, p) * n
            
        # Solve A * x = b
        if is_singular(A):
            # Fallback: centroid of intersection points
            x = mean([edge_hermite_data[eid][0] for eid in edge_ids])
        else:
            x = solve_linear_system(A, b)
            
        vertices[vertex_key] = x

    # Step 5: Generate polygons
    # For each sign-change edge, get the 4 adjacent cells
    for edge_id in edge_hermite_data.keys():
        adjacent_cells = get_adjacent_cells(edge_id)
        if len(adjacent_cells) != 4:
            continue # Boundary case
            
        quad_vertices = []
        for cell_id in adjacent_cells:
            # Determine which vertex key corresponds to this edge in this cell
            # This requires reversing the mapping or storing it per cell
            # We need to know if this edge is part of a cycle or a standard edge
            # in this specific cell.
            
            # Find the topology for this cell
            cell = grid.get_cell(cell_id)
            topology = lookup_modified_mc_table(cell.sign_pattern)
            
            vertex_key = None
            if topology.type == 'triangle':
                if edge_id in topology.edges:
                    vertex_key = (cell_id, None)
            elif topology.type == 'cycle':
                for cycle in topology.cycles:
                    if edge_id in cycle.edges:
                        vertex_key = (cell_id, get_cycle_id(cycle))
                        break
            
            if vertex_key and vertex_key in vertices:
                quad_vertices.append(vertices[vertex_key])
            else:
                # Fallback or error
                quad_vertices.append(zero_vector(3))
                
        if len(quad_vertices) == 4:
            faces.append(quad_vertices)
            
    return vertices.values(), faces
```

**Parameters:**
*   Isosurface value: Typically 0.
*   Hermite data accuracy: Depends on the gradient estimation method used to generate normals.

**Edge Cases and Pitfalls:**
*   **Singular QEF:** If all normals associated with a vertex are parallel, the matrix $A_v$ is singular. Use a pseudo-inverse or fallback to the centroid of intersection points [G].
*   **Boundary Conditions:** Edges on the grid boundary have fewer than four adjacent cells. Handle by clamping, mirroring, or omitting faces [G].
*   **Degenerate Input:** If Hermite data is missing for an edge, the method cannot place the vertex accurately. Use midpoint as fallback [G].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of grid cells, assuming constant-time lookup and small fixed-size QEF systems (typically 3x3).
*   **Memory Complexity:** $O(N)$ for storing grid data, edge data, and vertex positions.
*   **Strengths:** Always produces a manifold surface; reproduces sharp features; robust on uniform grids.
*   **Limitations:** Requires Hermite data; quadrilateral output may require triangulation for some renderers; more complex than standard Marching Cubes.
*   **When to use:** When sharp features must be preserved and manifold topology is required, especially on uniform grids where standard Dual Contouring fails.
*   **When not to use:** If Hermite data is unavailable or if triangular output is strictly required without post-processing.

## Background
Hermite Dual Marching Cubes was introduced as an extension to Nielson's Dual Marching Cubes algorithm. It addresses the limitations of standard Dual Contouring, which produces non-manifold surfaces on uniform grids for ambiguous sign configurations, and standard Marching Cubes, which fails to reproduce sharp features. The method was developed to provide a robust, manifold-preserving contouring algorithm that leverages Hermite data for feature preservation.

## Sources
[S1] Manifold Dual Contouring, https://doi.org/10.1109/tvcg.2007.1012, fetched 2026-10-26
[S2] PDF Manifold Dual Contouring, https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf, fetched 2026-10-27 (unofficial)
