# Marching Tetrahedra

Also called: 5-tetrahedron decomposition, MarchTetrahedron, Marching Tetrahedrons, Tetrahedral Marching, sliced Marching Cubes

## Overview

Marching Tetrahedra is an isosurface extraction algorithm that decomposes each cubic voxel into tetrahedra to resolve the topological ambiguities inherent in [Marching Cubes](774-marching-cubes.md) [S5, unofficial]. By processing each tetrahedron independently, it guarantees a unique, unambiguous intersection for every configuration, producing a crack-free mesh without the need for special case handling [S1, unofficial, S2, unofficial, S3, unofficial, S4, unofficial, S5, unofficial].

## How it works

### Inputs and Outputs
- **Input**: A 3D scalar field $V(x,y,z)$ defined on a regular grid, an isovalue $I$, and grid dimensions $(N_x, N_y, N_z)$ [S5, unofficial].
- **Output**: A polygonal mesh (list of vertices and triangles) representing the isosurface $V(x,y,z) = I$ [S1, unofficial].

### Data Structures

**Voxel Decomposition**
Each voxel (cube) is decomposed into 6 tetrahedra [S2, unofficial]. The vertices of the cube are indexed 0–7 [S3, unofficial]. A common decomposition shares the main diagonal between vertex 0 and vertex 7 [S2, unofficial, S5, unofficial, S6, unofficial]. The 6 tetrahedra are defined by the following vertex indices (0-based):
1. $T_1: (0, 1, 3, 2)$
2. $T_2: (0, 7, 3, 2)$
3. $T_3: (0, 7, 2, 6)$
4. $T_4: (0, 4, 6, 7)$
5. $T_5: (0, 6, 1, 2)$
6. $T_6: (0, 6, 1, 4)$
*Note: Source [S5, unofficial] lists a slightly different set $\{0,7,3,2\}, \{0,7,2,6\}, \{0,4,6,7\}, \{0,6,1,2\}, \{0,6,1,4\}, \{5,6,1,4\}$. The specific decomposition must be consistent across all voxels to ensure shared edges are identified correctly [S1, unofficial].*

**Edge Intersection Cache**
To prevent gaps (cracks) due to floating-point precision errors, intersection points on shared edges must be identical for adjacent tetrahedra/voxels [S5, unofficial]. A cache maps a unique edge identifier to the computed 3D position [S5, unofficial].
- **Edge Key**: A unique integer or tuple identifying an edge. It is typically derived from the voxel coordinates and the edge index within the voxel, or simply the sorted pair of global vertex indices if the grid vertices are shared [S5, unofficial].
- **Cache Structure**: `Map<EdgeKey, Vector3>` or a pre-allocated array if the number of edges is known [S5, unofficial].

**Lookup Tables**
- **Tetrahedron Configuration**: A 4-bit mask (16 entries) indicating which of the 4 vertices are above the isovalue [S6, unofficial].
- **Triangle Output**: For each of the 16 configurations, a table specifies which edges are intersected. Each intersected edge yields a vertex. The table outputs 0, 1, or 2 triangles defined by these edge indices [S1, unofficial]. The lookup table uses local edge indices (0–5) for the tetrahedron [S6, unofficial], while global voxel edges are indexed 1–19 [S1, unofficial].

### Steps

1. **Precompute Scalar Values**:
   Sample the scalar field at all grid points $(i,j,k)$ for $0 \le i < N_x, 0 \le j < N_y, 0 \le k < N_z$ [S1, unofficial]. Store in a 3D array `grid[N_x][N_y][N_z]` [S5, unofficial].

2. **Initialize Cache and Output**:
   Create an empty vertex cache `edge_cache` and an empty list `triangles` [S1, unofficial, S5, unofficial].

3. **Iterate Voxels**:
   For each voxel at grid coordinates $(i,j,k)$:
   
   a. **Read Vertices**:
      Retrieve the 8 scalar values from `grid` at the voxel corners. Map them to local indices 0–7 based on the chosen decomposition [S3, unofficial, S5, unofficial].
      
   b. **Decompose into Tetrahedra**:
      For each of the 6 tetrahedra $T_m$ ($m=1..6$):
      
      i. **Get Vertex Values**:
         Retrieve scalar values $v_0, v_1, v_2, v_3$ for the 4 vertices of $T_m$ [S2, unofficial, S5, unofficial].
         
      ii. **Compute Configuration Mask**:
         Create a 4-bit integer `mask`. If vertex $k$ is above the isovalue ($v_k > I$), set bit $k$.
         $$ \text{mask} = \sum_{k=0}^{3} \begin{cases} 1 & \text{if } v_k > I \\ 0 & \text{otherwise} \end{cases} \cdot 2^k $$
         We assume $>$ for "inside" or "positive" for this description, but the lookup table must match the sign convention [S2, unofficial, S5, unofficial].
         
      iii. **Lookup Triangles**:
         Use `mask` to index into the triangle lookup table to get a list of edge indices that are intersected. The table returns 0, 1, or 2 triangles. Each triangle is defined by 3 edge indices (0–5) [S1, unofficial, S6, unofficial].
         
      iv. **Generate Vertices**:
         For each edge index $e$ in the triangle definition:
         - Identify the two vertices of the tetrahedron connected by edge $e$. Let them be $P_a$ and $P_b$ with scalar values $S_a$ and $S_b$ [S6, unofficial].
         - Compute the global edge key for this edge (based on voxel $(i,j,k)$ and edge $e$).
         - If the key is in `edge_cache`, retrieve the cached vertex position $P_{int}$.
         - Else, compute $P_{int}$ using linear interpolation:
           $$ t = \frac{I - S_a}{S_b - S_a} $$
           $$ P_{int} = P_a + t \cdot (P_b - P_a) $$
           *Guard*: If $S_a == S_b$, use the midpoint to avoid division by zero [S5, unofficial]. Clamp $t$ to $[\epsilon, 1-\epsilon]$ to prevent vertices from being placed exactly on voxel corners [S1, unofficial].
           Store $P_{int}$ in `edge_cache` and add to the vertex list if not already present (depending on implementation; caching by edge key is sufficient for consistency).
         
      v. **Add Triangles**:
         For each triangle defined by edge indices $(e_0, e_1, e_2)$, retrieve the corresponding intersection points $P_{e_0}, P_{e_1}, P_{e_2}$ from the cache. Add the triangle $(P_{e_0}, P_{e_1}, P_{e_2})$ to `triangles`.
         *Note*: Ensure consistent winding order. The decomposition and lookup table must be consistent [S3, unofficial].

4. **Post-Processing (Optional)**:
   If disconnected shells are required, perform a connectivity traversal on the generated triangles [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_tetrahedra(grid, dims, isovalue, eps=1e-3):
    """
    grid: 3D array of scalar values, shape (Nx, Ny, Nz)
    dims: tuple (Nx, Ny, Nz)
    isovalue: float, the threshold for isosurface extraction
    eps: tolerance factor to prevent vertices on voxel corners
    """
    # Define the 6 tetrahedra per voxel. 
    # Vertices are local indices 0-7 of the cube.
    tetra_vertices = [
        [0, 1, 3, 2],
        [0, 7, 3, 2],
        [0, 7, 2, 6],
        [0, 4, 6, 7],
        [0, 6, 1, 2],
        [0, 6, 1, 4]
    ]
    
    # Define edges for a tetrahedron with vertices 0,1,2,3
    # Edge 0: (0,1), Edge 1: (0,2), Edge 2: (0,3), Edge 3: (1,2), Edge 4: (1,3), Edge 5: (2,3)
    tetra_edges = [
        (0, 1), (0, 2), (0, 3),
        (1, 2), (1, 3), (2, 3)
    ]
    
    # Lookup table: mask -> list of triangles. 
    # Each triangle is a tuple of 3 edge indices (0-5).
    # Assumes mask bit k is 1 if vertex k > isovalue.
    tet_lookup = {
        0: [],
        1: [(0, 1, 2)],
        2: [(0, 3, 4)],
        3: [(3, 4, 5), (0, 1, 2)],
        4: [(1, 3, 5)],
        5: [(0, 3, 4), (1, 3, 5)],
        6: [(0, 1, 2), (1, 3, 5)],
        7: [(0, 1, 2), (0, 3, 4), (1, 3, 5)],
        8: [(2, 4, 5)],
        9: [(0, 1, 2), (2, 4, 5)],
        10: [(0, 3, 4), (2, 4, 5)],
        11: [(0, 1, 2), (0, 3, 4), (2, 4, 5)],
        12: [(1, 3, 5), (2, 4, 5)],
        13: [(0, 1, 2), (1, 3, 5), (2, 4, 5)],
        14: [(0, 3, 4), (1, 3, 5), (2, 4, 5)],
        15: [(0, 1, 2), (0, 3, 4), (1, 3, 5), (2, 4, 5)]
    }

    edge_cache = {}
    triangles = []
    
    Nx, Ny, Nz = dims
    
    for i in range(Nx - 1):
        for j in range(Ny - 1):
            for k in range(Nz - 1):
                # Get 8 corner values
                corners = [
                    grid[i, j, k],
                    grid[i+1, j, k],
                    grid[i+1, j+1, k],
                    grid[i, j+1, k],
                    grid[i, j, k+1],
                    grid[i+1, j, k+1],
                    grid[i+1, j+1, k+1],
                    grid[i, j+1, k+1]
                ]
                
                # Get world coordinates for corners (assuming unit spacing)
                corner_coords = [
                    (i, j, k), (i+1, j, k), (i+1, j+1, k), (i, j+1, k),
                    (i, j, k+1), (i+1, j, k+1), (i+1, j+1, k+1), (i, j+1, k+1)
                ]

                for tet_idx in range(6):
                    v_indices = tetra_vertices[tet_idx]
                    v_vals = [corners[v] for v in v_indices]
                    v_coords = [corner_coords[v] for v in v_indices]
                    
                    # Compute mask
                    mask = 0
                    for idx, val in enumerate(v_vals):
                        if val > isovalue:
                            mask |= (1 << idx)
                            
                    # Get triangles for this mask
                    tri_edge_indices = tet_lookup.get(mask, [])
                    
                    for tri in tri_edge_indices:
                        e0, e1, e2 = tri
                        pts = []
                        for e in [e0, e1, e2]:
                            v_a_idx, v_b_idx = tetra_edges[e]
                            va = v_coords[v_a_idx]
                            vb = v_coords[v_b_idx]
                            sa = v_vals[v_a_idx]
                            sb = v_vals[v_b_idx]
                            
                            # Create cache key using sorted coordinates
                            key = tuple(sorted([va, vb]))
                            
                            if key in edge_cache:
                                pt = edge_cache[key]
                            else:
                                denom = sb - sa
                                if denom == 0.0:
                                    t = 0.5
                                else:
                                    t = (isovalue - sa) / denom
                                # Clamp t to [eps, 1-eps] to prevent vertices on corners
                                t = max(eps, min(1.0 - eps, t))
                                pt = (va[0] + t * (vb[0] - va[0]),
                                      va[1] + t * (vb[1] - va[1]),
                                      va[2] + t * (vb[2] - va[2]))
                                edge_cache[key] = pt
                            pts.append(pt)
                        
                        # Filter degenerate triangles (check if vertices are distinct)
                        if pts[0] != pts[1] and pts[1] != pts[2] and pts[0] != pts[2]:
                            triangles.append(pts)
                        
    return triangles
```

### Parameters
- **isovalue**: The scalar value defining the surface. Default 0.0. Typical values depend on the data (e.g., 100, 140, 150, 200 for medical CT) [S1, unofficial, S2, unofficial].
- **eps**: Tolerance factor to prevent vertices from being placed exactly on voxel corners, ensuring manifold mesh generation (default 1e-3) [S1, unofficial].

### Edge Cases and Pitfalls
- **Degenerate Triangles**: If a vertex value equals the isovalue, the interpolation parameter $t$ may be undefined or result in a zero-area triangle. Filter these out by checking if the interpolated point is a single point (i.e., if `buildTriangle` returns `NULL` or if `isPoint()` is true) [S2, unofficial].
- **Cracks**: Without caching edge intersections, floating-point errors can cause gaps between adjacent tetrahedra. Always cache and reuse intersection points for shared edges [S5, unofficial].
- **Winding Order**: Incorrect vertex ordering in the lookup table or tetrahedron decomposition can lead to inverted normals. The decomposition and lookup table must be consistent [S3, unofficial].
- **Ambiguity**: Unlike Marching Cubes, Marching Tetrahedra has no topological ambiguities. However, the choice of decomposition (which diagonal is shared) can influence the visual appearance (e.g., bumps along face diagonals) [S5, unofficial].

## Subtopics

### [Marching Cubes](774-marching-cubes.md)
The parent algorithm that MT replaces to resolve ambiguities.

### [Topological Correctness Guarantees](1796-topological-correctness-guarantees.md)
MT provides inherent topological correctness for the tetrahedral decomposition, unlike MC.

### [Block face vertex layout](2374-block-face-vertex-layout.md)
Relevant for ensuring consistent vertex ordering across voxels.

## Operating characteristics

- **Time Complexity**: $O(N)$, where $N$ is the number of voxels. Each voxel is processed independently in constant time (6 tetrahedra, 16 cases) [S5, unofficial].
- **Memory Complexity**: $O(N)$ for the scalar field. The output mesh size is proportional to the surface area. The edge cache requires $O(N)$ space if all edges are stored, or can be optimized by processing slices [S5, unofficial].
- **Performance**: Generally slower than Marching Cubes due to more triangles generated (up to 19 edge samples per cube vs 12) and the overhead of tetrahedral decomposition [S4, unofficial, S5, unofficial].
- **Strengths**: Unambiguous, crack-free, topologically correct. No special cases needed [S1, unofficial, S2, unofficial, S3, unofficial, S4, unofficial, S5, unofficial].
- **Limitations**: More triangles than MC. The choice of decomposition can introduce artifacts (bumps) along the face diagonals [S5, unofficial].

## Background

Marching Tetrahedra was introduced in 1991 by Doi and Koide as a crack-free, historically patent-free alternative to Marching Cubes [S4, unofficial]. It resolves the 15 ambiguous cases of Marching Cubes by decomposing the cube into tetrahedra, each of which has a unique intersection topology [S3, unofficial, S4, unofficial].

## Sources

[S1] JuliaGeometry/Meshing.jl, https://github.com/JuliaGeometry/Meshing.jl, fetched 2026-10-27 (unofficial)
[S2] andresbejarano/MarchingTetrahedra, https://github.com/andresbejarano/MarchingTetrahedra, fetched 2026-10-27 (unofficial)
[S3] d3x0r/MarchingTetrahedra, https://github.com/d3x0r/MarchingTetrahedra, fetched 2026-10-27 (unofficial)
[S4] RobertBoettcherSF/Ada-Marching-Tetrahedrons, https://github.com/RobertBoettcherSF/Ada-Marching-Tetrahedrons, fetched 2026-10-27 (unofficial)
[S5] 2453852330/VoxelHelper, https://github.com/2453852330/VoxelHelper, fetched 2026-10-27 (unofficial)
[S6] Marching Cubes algorithm, https://github.com/thejasonfisher/marchingCubes/blob/master/MarchingCubes.cs, fetched 2026-10-30 (unofficial)
