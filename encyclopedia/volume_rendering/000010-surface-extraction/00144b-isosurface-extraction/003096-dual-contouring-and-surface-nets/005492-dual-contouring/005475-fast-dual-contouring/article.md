# Fast Dual Contouring

Also called: Dual Contouring Plugin, Fast DC

## Overview
Fast Dual Contouring is a surface extraction method that generates a manifold mesh from a voxel grid by identifying edge intersections and computing vertex positions via Quadratic Error Function (QEF) minimization [S1, unofficial]. It distinguishes itself from standard Dual Contouring by optimizing the grid traversal and QEF assembly for real-time performance, often implemented as a C++ plugin for engines like Unity [S1, unofficial]. Unlike Marching Cubes, which produces fixed topology per cell, this method creates a single vertex per feature (edge intersection) shared by multiple cells, allowing for sharper feature preservation and potential mesh simplification [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A scalar density field defined on a regular 3D grid.
*   Grid dimensions: `voxelGridSize` (cubic, e.g., 128).
*   Grid offset: `gridOffset` (world space origin of the grid).
*   Cell size: `cellSize` (typically 1.0m).
*   World origin offset: `worldX, worldY, worldZ`.
*   Simplification parameters (optional): `targetPolygonPercent`, `maxSimplifyIterations`, `edgeFraction`, `maxEdgeSize`, `maxError`, `minAngleCosine` [S1, unofficial].

**Outputs:**
*   A mesh consisting of vertices, normals, and triangle indices.
*   Vertex buffer layout: Flat float array, processed 6 floats at a time (indices 0-2: position, 3-5: normal) [S1, unofficial].

### Data Structures

**Voxel ID Encoding (32-bit integer):**
*   Bits 0-9: X coordinate (supports 0-1023).
*   Bits 10-19: Y coordinate.
*   Bits 20-29: Z coordinate.
*   Bits 30-31: Unused [S1, unofficial].

**Axis Edge ID Encoding (32-bit integer):**
*   Bits 0-9: X coordinate of the voxel.
*   Bits 10-19: Y coordinate of the voxel.
*   Bits 20-29: Z coordinate of the voxel.
*   Bits 30-31: Axis index (0, 1, or 2) [S1, unofficial].

**EdgeInfo Struct:**
*   `pos`: `vec4` (position of the intersection).
*   `normal`: `vec4` (normal vector at the intersection; the `w` component is unused and can be optimized for winding storage) [S1, unofficial].
*   `winding`: `bool` (indicates if the starting density of the edge was $\ge 0$) [S1, unofficial].

**Active Data:**
*   `activeEdges`: Map/Dictionary keyed by Axis Edge ID, storing `EdgeInfo`.
*   `activeVoxels`: Set of Voxel IDs (32-bit integers) representing the 4 voxels sharing the current edge [S1, unofficial].

### Algorithm Steps

1.  **Initialization**: Initialize simplification options if mesh simplification is enabled [S1, unofficial].
2.  **Grid Traversal**: Iterate through all voxels in the grid for $x, y, z$ from $0$ to `voxelGridSize` - 1.
3.  **Edge Intersection Detection**: For each voxel, iterate through 3 axes ($0, 1, 2$).
    *   Calculate position $p$ in world space:
        $$ p = (x - \text{gridOffset} + \text{worldX}, y - \text{gridOffset} + \text{worldY}, z - \text{gridOffset} + \text{worldZ}) $$
    *   Calculate neighbor position $q$ by adding the axis offset vector $\vec{A}_{axis}$:
        $$ q = p + \vec{A}_{axis} $$
    *   Sample densities $d_p = \text{Density}(p)$ and $d_q = \text{Density}(q)$.
    *   Check for zero crossing: If $(d_p \ge 0 \land d_q < 0) \lor (d_p < 0 \land d_q \ge 0)$, an intersection exists. Otherwise, continue to the next axis.
4.  **Intersection Calculation**:
    *   Find the intersection parameter $t$ along the edge between $p$ and $q$. The source uses a coarse search of 16 uniform steps (or 8 if subdivision level is 0), incrementing $t$ by $1/16$ (or $1/8$) and minimizing $|\text{Density}(\text{mix}(p, q, t))|$ [S1, unofficial].
    *   Calculate intersection position:
        $$ \text{pos} = \text{mix}(p, q, t) = p + t(q - p) $$
5.  **Normal Estimation**:
    *   Calculate the normal at `pos` using central differences with step $H=0.001$:
        $$ \vec{n} = \text{normalize}\left( \begin{bmatrix} \text{Density}(\text{pos} + H\hat{x}) - \text{Density}(\text{pos} - H\hat{x}) \\ \text{Density}(\text{pos} + H\hat{y}) - \text{Density}(\text{pos} - H\hat{y}) \\ \text{Density}(\text{pos} + H\hat{z}) - \text{Density}(\text{pos} - H\hat{z}) \end{bmatrix} \right) $$
6.  **Edge Storage**:
    *   Create `EdgeInfo` with `pos`, `normal`, and `winding` ($d_p \ge 0$).
    *   Encode the edge ID using the axis and voxel coordinates.
    *   Store in `activeEdges` map [S1, unofficial].
7.  **QEF Assembly (Voxel Identification)**:
    *   Identify the 4 voxels sharing this edge using predefined `EDGE_NODE_OFFSETS`.
    *   Encode their unique 32-bit Voxel IDs and add them to the `activeVoxels` set for the current edge context.
    *   For each of the 4 voxels, add the edge's intersection position and normal to the voxel's Quadratic Error Function (QEF) solver. The QEF minimizes the distance from the voxel's vertex to the plane defined by the edge intersection and normal [S1, unofficial].
8.  **Mesh Generation**:
    *   For each voxel, solve its QEF to determine the single vertex position for that voxel's feature.
    *   Generate the mesh buffer. The source code implies a structure where faces are grouped by the 4 cells sharing an edge, but the final output is a standard triangle mesh [S1, unofficial].
    *   Copy vertices to the simplified mesh structure if simplification is active.
9.  **Simplification (Optional)**:
    *   Apply mesh simplification using the configured parameters (`targetPolygonPercent`, etc.) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_dual_contouring(density_function, grid_size, grid_offset, world_origin, cell_size=1.0, simplify=False):
    """
    Generates a mesh from a voxel grid using Fast Dual Contouring.
    
    Args:
        density_function: Callable(x, y, z) -> float
        grid_size: int, dimension of the cubic grid (e.g., 128)
        grid_offset: float, offset from world origin to grid start
        world_origin: vec3, world space offset
        cell_size: float, size of each voxel
        simplify: bool, whether to simplify the resulting mesh
        
    Returns:
        vertices: list of vec3
        normals: list of vec3
        indices: list of int
    """
    
    # Data structures
    # Map: AxisEdgeID -> EdgeInfo
    active_edges = {}
    # Map: VoxelID -> QEF (Quadratic Error Function accumulator)
    voxel_qefs = {}
    
    # Axis vectors
    axes = [
        vec3(cell_size, 0, 0),
        vec3(0, cell_size, 0),
        vec3(0, 0, cell_size)
    ]
    
    # Predefined offsets for the 4 voxels sharing an edge along axis 'a' at voxel (x,y,z)
    # These are relative offsets from the base voxel (x,y,z)
    # Note: The exact offsets depend on the axis. 
    # For axis 0 (X): neighbors are (0,0,0), (0,0,-1), (0,-1,-1), (0,-1,0) relative to the edge's "corner" logic?
    # The source code shows specific offsets for each axis direction. 
    # We will use a generic representation here as the exact offsets are axis-dependent.
    edge_node_offsets = get_edge_node_offsets() 
    
    # Step 1-3: Traverse grid and find intersections
    for x in range(grid_size):
        for y in range(grid_size):
            for z in range(grid_size):
                for axis in range(3):
                    # Calculate p and q
                    p_world = vec3(x, y, z) * cell_size + grid_offset + world_origin
                    q_world = p_world + axes[axis]
                    
                    p_density = density_function(p_world.x, p_world.y, p_world.z)
                    q_density = density_function(q_world.x, q_world.y, q_world.z)
                    
                    # Check for zero crossing
                    if (p_density >= 0 and q_density < 0) or (p_density < 0 and q_density >= 0):
                        
                        # Step 4: Find intersection
                        # Coarse search with 16 steps
                        t = 0.0
                        best_t = 0.0
                        min_abs_density = abs(p_density)
                        
                        for step in range(1, 17):
                            t = step / 16.0
                            mix_pos = mix(p_world, q_world, t)
                            d = density_function(mix_pos.x, mix_pos.y, mix_pos.z)
                            if abs(d) < min_abs_density:
                                min_abs_density = abs(d)
                                best_t = t
                        
                        # Refine or use best_t
                        intersection_pos = mix(p_world, q_world, best_t)
                        
                        # Step 5: Calculate normal
                        h = 0.001
                        grad_x = density_function(intersection_pos.x + h, intersection_pos.y, intersection_pos.z) - \
                                 density_function(intersection_pos.x - h, intersection_pos.y, intersection_pos.z)
                        grad_y = density_function(intersection_pos.x, intersection_pos.y + h, intersection_pos.z) - \
                                 density_function(intersection_pos.x, intersection_pos.y - h, intersection_pos.z)
                        grad_z = density_function(intersection_pos.x, intersection_pos.y, intersection_pos.z + h) - \
                                 density_function(intersection_pos.x, intersection_pos.y, intersection_pos.z - h)
                        
                        normal = normalize(vec3(grad_x, grad_y, grad_z))
                        
                        # Step 6: Store edge info
                        edge_id = encode_axis_edge_id(x, y, z, axis)
                        active_edges[edge_id] = EdgeInfo(pos=intersection_pos, normal=normal, winding=(p_density >= 0))
                        
                        # Step 7: Identify 4 voxels and add to QEF
                        # The 4 voxels sharing the edge between p and q
                        # We need to determine the base voxel for the edge. 
                        # In the source, it seems to iterate cells and check edges. 
                        # Here, we identify the 4 cells that share this edge.
                        # The offsets are relative to the voxel containing 'p' if we consider p as the start.
                        # However, the source code iterates cells and checks edges. 
                        # To avoid duplicates, we can use the edge ID as the key.
                        
                        # Get the 4 voxel IDs sharing this edge
                        voxel_ids = get_voxel_ids_for_edge(x, y, z, axis, grid_size)
                        
                        for vid in voxel_ids:
                            if vid not in voxel_qefs:
                                voxel_qefs[vid] = QEF()
                            
                            # Add plane constraint to QEF
                            # The plane is defined by the intersection point and normal
                            voxel_qefs[vid].add_plane(intersection_pos, normal)
                            
    # Step 8: Solve QEF and generate mesh
    vertices = []
    normals = []
    indices = []
    
    # Map from VoxelID to VertexIndex
    voxel_to_vertex = {}
    
    for vid, qef in voxel_qefs.items():
        # Solve QEF: minimize ||Ax - b||^2
        # A is a matrix of normals, b is A * intersection_points
        # The solution is x = (A^T A)^-1 A^T b
        vertex_pos = qef.solve()
        
        # Store vertex
        vertex_index = len(vertices)
        vertices.append(vertex_pos)
        normals.append(qef.normal) # Use the averaged normal or the one from the last plane? 
                                   # Source uses the normal from the edge intersection. 
                                   # For a voxel with multiple edges, the normal might be averaged or taken from the primary edge.
                                   # The source code suggests storing normal in EdgeInfo. 
                                   # Here we assume the QEF solver returns a position, and we might need to recompute or store normal.
                                   # For simplicity, we'll use the normal from the first plane added or average them.
                                   # [G] Standard Dual Contouring often uses the normal from the QEF solution's gradient or averages the plane normals.
                                   # The source code stores normal in EdgeInfo. We will assume the vertex normal is the normalized sum of the plane normals used in QEF.
        voxel_to_vertex[vid] = vertex_index
        
    # Generate triangles
    # The source code groups faces by the 4 cells sharing an edge.
    # This implies a dual graph approach where each edge intersection becomes a vertex, and faces are formed by connecting these vertices.
    # However, the final output is a triangle mesh.
    # The source code snippet shows:
    # foreach(List<GridFace> c in faces.Values) { ... GridCell c0, c1, c2, c3 ... }
    # This suggests that for each edge, the 4 surrounding voxels form a quad (or two triangles).
    
    # Re-iterate edges to form faces
    for edge_id, edge_info in active_edges.items():
        # Decode voxel IDs from edge_id? No, we need the 4 voxels.
        # We can retrieve them from the active_edges processing or store them.
        # Let's assume we have a way to get the 4 voxel IDs for this edge.
        voxel_ids = get_voxel_ids_for_edge_from_id(edge_id)
        
        # Check if all 4 voxels have vertices
        if all(vid in voxel_to_vertex for vid in voxel_ids):
            v0 = voxel_to_vertex[voxel_ids[0]]
            v1 = voxel_to_vertex[voxel_ids[1]]
            v2 = voxel_to_vertex[voxel_ids[2]]
            v3 = voxel_to_vertex[voxel_ids[3]]
            
            # Form triangles. The order depends on the winding and axis.
            # The source code sorts the faces to avoid duplicates.
            # We will assume a standard quad triangulation: (v0, v1, v2) and (v0, v2, v3)
            # [G] The exact winding order must be consistent to form a manifold.
            # The source code uses a sorting mechanism on GridCell indices to ensure consistent ordering.
            # Here we use a simple heuristic.
            indices.extend([v0, v1, v2])
            indices.extend([v0, v2, v3])
            
    return vertices, normals, indices

def get_edge_node_offsets():
    """
    Returns the relative offsets for the 4 voxels sharing an edge.
    This is axis-dependent.
    """
    # Example for Axis 0 (X):
    # The edge is along X. The 4 voxels share this edge.
    # Offsets might be (0,0,0), (0,0,-1), (0,-1,-1), (0,-1,0) relative to some base.
    # The source code has specific blocks for each axis direction.
    # We will return a placeholder structure.
    return {
        0: [(0,0,0), (0,0,-1), (0,-1,-1), (0,-1,0)],
        1: [(0,0,-1), (0,0,0), (-1,0,0), (-1,0,-1)],
        2: [(0,0,0), (0,-1,0), (-1,-1,0), (-1,0,0)]
    }

def get_voxel_ids_for_edge(x, y, z, axis, grid_size):
    """
    Returns the 4 voxel IDs sharing the edge at (x,y,z) along 'axis'.
    """
    offsets = get_edge_node_offsets()[axis]
    voxel_ids = []
    for ox, oy, oz in offsets:
        nx, ny, nz = x + ox, y + oy, z + oz
        if 0 <= nx < grid_size and 0 <= ny < grid_size and 0 <= nz < grid_size:
            voxel_ids.append(encode_voxel_id(nx, ny, nz))
    return voxel_ids

def encode_voxel_id(x, y, z):
    return (x & 0x3FF) | ((y & 0x3FF) << 10) | ((z & 0x3FF) << 20)

def encode_axis_edge_id(x, y, z, axis):
    return (x & 0x3FF) | ((y & 0x3FF) << 10) | ((z & 0x3FF) << 20) | (axis << 30)

class QEF:
    def __init__(self):
        self.planes = []
        
    def add_plane(self, point, normal):
        self.planes.append((point, normal))
        
    def solve(self):
        # Construct A and b
        # A^T A x = A^T b
        # A is a matrix where each row is the normal of a plane
        # b is a vector where each element is dot(normal, point)
        
        if not self.planes:
            return vec3(0, 0, 0)
            
        # [G] Implementing a simple 3x3 linear solver for Ax=b
        # A is n x 3, b is n x 1
        # We want to minimize ||Ax - b||^2
        # Solution: x = (A^T A)^-1 A^T b
        
        AtA = mat3(0)
        Atb = vec3(0)
        
        for point, normal in self.planes:
            # normal is vec3
            # point is vec3
            # plane equation: dot(normal, x - point) = 0 => dot(normal, x) = dot(normal, point)
            n = normal
            d = dot(n, point)
            
            # A^T A += n * n^T
            AtA += outer_product(n, n)
            # A^T b += n * d
            Atb += n * d
            
        # Solve 3x3 system
        try:
            x = solve_linear_system(AtA, Atb)
            return x
        except:
            # Fallback to centroid of points
            points = [p for p, n in self.planes]
            return sum(points) / len(points)

def outer_product(a, b):
    return mat3([a.x*b.x, a.x*b.y, a.x*b.z],
                [a.y*b.x, a.y*b.y, a.y*b.z],
                [a.z*b.x, a.z*b.y, a.z*b.z])

def solve_linear_system(A, b):
    # Gaussian elimination for 3x3
    # [G] Simplified solver
    n = 3
    # Augmented matrix
    M = [A[i] + [b[i]] for i in range(n)]
    
    for i in range(n):
        # Pivot
        max_row = i
        for k in range(i+1, n):
            if abs(M[k][i]) > abs(M[max_row][i]):
                max_row = k
        M[i], M[max_row] = M[max_row], M[i]
        
        if abs(M[i][i]) < 1e-9:
            continue
            
        for k in range(i+1, n):
            c = M[k][i] / M[i][i]
            for j in range(i, n+1):
                M[k][j] -= c * M[i][j]
                
    # Back substitution
    x = [0] * n
    for i in range(n-1, -1, -1):
        if abs(M[i][i]) < 1e-9:
            x[i] = 0
        else:
            x[i] = M[i][n]
            for j in range(i+1, n):
                x[i] -= M[i][j] * x[j]
            x[i] /= M[i][i]
            
    return vec3(x[0], x[1], x[2])
```

### Parameters
*   `voxelGridSize`: Dimension of the cubic voxel grid (e.g., 128). Cells are created on every axis. First cell starts at `center - cellSize/2`. Cells are typically 1m [S1, unofficial].
*   `targetPolygonPercent`: Parameter for mesh simplification. Note: In the Unity wrapper, this parameter is noted as not working [S1, unofficial].
*   `maxSimplifyIterations`: Integer parameter for mesh simplification [S1, unofficial].
*   `edgeFraction`: Float parameter for mesh simplification [S1, unofficial].
*   `maxEdgeSize`: Float parameter for mesh simplification [S1, unofficial].
*   `maxError`: Float parameter for mesh simplification [S1, unofficial].
*   `minAngleCosine`: Float parameter for mesh simplification [S1, unofficial].

### Edge Cases and Pitfalls
*   **Memory Leaks**: The C++ plugin implementation may leak memory. The Unity wrapper requires manual memory freeing using `Marshal.FreeCoTaskMem` for indices, vertices, and data arrays [S1, unofficial].
*   **Winding Optimization**: The `winding` field in `EdgeInfo` can be packed into the unused `w` component of the `normal` `vec4` to save memory [S1, unofficial].
*   **Simplification Issues**: The mesh simplification feature may not work correctly in the provided Unity wrapper implementation [S1, unofficial].
*   **Precision**: The intersection search uses a coarse grid of 16 steps. This may lead to inaccuracies in highly curved surfaces. [G] Refinement steps (e.g., Newton's method) are not described in the source but are common in production implementations.

## Operating characteristics

*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels, as each voxel and edge is processed once. The QEF solving is $O(1)$ per voxel (3x3 system).
*   **Memory Complexity**: $O(E)$ where $E$ is the number of edges with zero crossings. The `activeEdges` map and `voxel_qefs` dictionary store data proportional to the surface area.
*   **Performance**: The C++ plugin implementation is moderately faster than a pure C# implementation (e.g., 12ms vs 62ms for a sphere test) [S1, unofficial].
*   **Strengths**: Sharp feature preservation due to QEF minimization; potential for mesh simplification; efficient grid traversal.
*   **Limitations**: Memory management issues in the C++ plugin; simplification parameters may not work as expected; coarse intersection search may lack precision.

## Background

Fast Dual Contouring is an implementation of the Dual Contouring algorithm, optimized for performance. It was originally developed by [Gildea et al.] (implied by the reference to `nickgildea's code`) and ported to Unity by [DMeville] [S1, unofficial]. The method is used in real-time applications where high-quality isosurface extraction is required, such as in game engines.

## Sources

[S1] DMeville/Unity-Fast-Dual-Contouring, https://github.com/DMeville/Unity-Fast-Dual-Contouring, fetched 2026-10-26 (unofficial)
