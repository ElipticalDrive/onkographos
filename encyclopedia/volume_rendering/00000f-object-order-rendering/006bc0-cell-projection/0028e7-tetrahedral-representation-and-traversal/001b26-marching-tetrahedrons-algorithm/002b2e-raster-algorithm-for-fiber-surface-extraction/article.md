# Raster Algorithm for Fiber Surface Extraction

Also called: Range rasterization algorithm, algorithm

## Overview
The Raster Algorithm for Fiber Surface Extraction approximates fiber surfaces by extracting the 0 level-set of a signed range distance field to the Fiber Surface Control Polygon (FSCP) [S1]. It reduces the computational cost of distance field evaluation by rasterizing the scalar field range into a 2D grid, trading geometric accuracy for speed [S1]. Unlike exact methods, it imposes a lower bound on Hausdorff distance near FSCP bends and only handles closed FSCPs [S1].

## How it works
**Inputs:**
*   A tetrahedral mesh domain $Dom_f$ containing vertices $V$ and tetrahedra $T$ [S1].
*   The Fiber Surface Control Polygon (FSCP), defined as a set of control points or segments [S1].
*   A raster resolution $R$ (tested $16^2$ to $1024^2$; $1024^2$ is optimal for balancing speed and quality) [S1].

**Outputs:**
*   A triangle mesh representing the extracted fiber surface [S1].

**Data Structures:**
*   **Range Raster:** A 2D grid of size $R \times R$ [S1]. Each cell stores the minimum scalar value (distance) from the FSCP observed during projection [S1]. The grid is initialized to infinity [S1].
*   **Marching Tetrahedra Lookup Table:** Standard tables mapping scalar values at tetrahedron vertices to edge intersections and output triangles [S1].

**Steps:**
1.  **Initialize Raster:** Create a 2D array `raster[R][R]` and fill with $\infty$ [S1].
2.  **Project Vertices:** For each vertex $v_i$ in $Dom_f$ with scalar value $s_i$: project $v_i$ onto the 2D raster plane to obtain coordinates $(u_i, v_i)$ in raster space [S1]. Update the raster cell at $(u_i, v_i)$ with $s_i$ if $s_i$ is smaller than the current value: $\text{raster}[u_i][v_i] = \min(\text{raster}[u_i][v_i], s_i)$ [S1].
3.  **Evaluate Distance Field:** For each vertex $v_j$ in $Dom_f$: retrieve the projected coordinates $(u_j, v_j)$ [S1]. Perform bilinear interpolation on the `raster` grid at $(u_j, v_j)$ to obtain the signed distance value $d_j$ [S1]. Bilinear interpolation is used instead of piecewise constant interpolation to ensure smoother results [S1].
4.  **Extract Level-Set:** Apply the Marching Tetrahedra algorithm to the domain $Dom_f$ using the computed distance values $d_j$ at each vertex [S1]. For each tetrahedron, determine the sign of $d_j$ at its four vertices [S1]. Use the Marching Tetrahedra tables to generate triangles where the 0 level-set crosses the tetrahedron edges [S1].
5.  **Output:** Collect all generated triangles [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raster_algorithm_fiber_extraction(tet_mesh, fscp, raster_res):
    # Step 1: Initialize raster
    raster = [[float('inf') for _ in range(raster_res)] for _ in range(raster_res)]

    # Helper: Project 3D vertex to 2D raster coordinates (orthographic projection to XY plane) [G]
    def project_to_raster(point):
        u = int((point.x - tet_mesh.min_x) / (tet_mesh.max_x - tet_mesh.min_x) * (raster_res - 1))
        v = int((point.y - tet_mesh.min_y) / (tet_mesh.max_y - tet_mesh.min_y) * (raster_res - 1))
        return max(0, min(raster_res - 1, u)), max(0, min(raster_res - 1, v))

    # Step 2: Project vertices into range raster
    for vertex in tet_mesh.vertices:
        u, v = project_to_raster(vertex.position)
        if vertex.scalar_value < raster[v][u]:
            raster[v][u] = vertex.scalar_value

    # Step 3: Evaluate distance field using bilinear interpolation
    distance_field = {}
    for vertex in tet_mesh.vertices:
        u, v = project_to_raster(vertex.position)
        u_frac = u - int(u)
        v_frac = v - int(v)
        u0, v0 = int(u), int(v)
        u1 = min(u0 + 1, raster_res - 1)
        v1 = min(v0 + 1, raster_res - 1)

        q00 = raster[v0][u0]
        q10 = raster[v0][u1]
        q01 = raster[v1][u0]
        q11 = raster[v1][u1]

        dist = (q00 * (1 - u_frac) * (1 - v_frac) +
                q10 * u_frac * (1 - v_frac) +
                q01 * (1 - u_frac) * v_frac +
                q11 * u_frac * v_frac)
        distance_field[vertex.id] = dist

    # Step 4: Extract 0 level-set using Marching Tetrahedra
    output_triangles = []
    for tet in tet_mesh.tetrahedra:
        verts = tet.vertices
        d_vals = [distance_field[v.id] for v in verts]

        # Compute case index: bit i set if d_vals[i] < 0
        case_index = 0
        for i, d in enumerate(d_vals):
            if d < 0:
                case_index |= (1 << i)

        # Standard MT Tetrahedra table: case -> list of edge indices (each edge is a pair of vertex indices)
        mt_edges_table = {
            0: [], 1: [(0,1)], 2: [(0,2)], 3: [(0,1),(0,2)], 4: [(0,3)], 5: [(0,1),(0,3)], 6: [(0,2),(0,3)], 7: [(0,1),(0,2),(0,3)],
            8: [(1,2)], 9: [(0,1),(1,2)], 10: [(0,2),(1,2)], 11: [(0,1),(0,2),(1,2)], 12: [(1,3)], 13: [(0,1),(1,3)], 14: [(0,2),(1,3)], 15: [(0,1),(0,2),(1,3)]
        }
        # Standard MT Tetrahedra table: case -> list of triangles (each triangle is a list of 3 edge indices)
        mt_tri_table = {
            0: [], 1: [[0, 1, 2]], 2: [[0, 1, 3]], 3: [[0, 2, 3]], 4: [[1, 2, 3]],
            5: [[0, 1], [2, 3]], 6: [[0, 2], [1, 3]], 7: [[0, 3], [1, 2]],
            8: [[1, 2], [0, 3]], 9: [[0, 2], [1, 3]], 10: [[0, 1], [2, 3]], 11: [[0, 3], [1, 2]],
            12: [[0, 1], [2, 3]], 13: [[0, 2], [1, 3]], 14: [[0, 3], [1, 2]], 15: []
        }

        def interp_edge(i, j):
            d_i, d_j = d_vals[i], d_vals[j]
            if d_i == d_j: return None
            t = d_i / (d_i - d_j)
            return verts[i].position + t * (verts[j].position - verts[i].position)

        # Generate triangles from table
        for tri_edges in mt_tri_table.get(case_index, []):
            p1 = interp_edge(*tri_edges[0])
            p2 = interp_edge(*tri_edges[1])
            p3 = interp_edge(*tri_edges[2])
            if p1 is not None and p2 is not None and p3 is not None:
                output_triangles.append([p1, p2, p3])

    return output_triangles
```

**Parameters:**
*   **Raster Resolution ($R$):** Tested values include $16^2, 32^2, 64^2, 128^2, 256^2, 512^2, 1024^2$ [S1]. $1024^2$ is selected as optimal [S1].

**Edge Cases and Pitfalls:**
*   **Parallel Reduction Imbalance:** Threads may produce unequal numbers of triangles, causing idle time during the parallel reduction step [S1].
*   **Geometric Accuracy Limit:** The distance field fails to capture sharp surface features corresponding to FSCP bends, imposing a Hausdorff distance lower bound of approximately 1% of the bounding box diagonal [S1].
*   **Closed FSCP Only:** The algorithm only handles closed FSCPs [S1].
*   **Projection Cost:** At low resolutions (below $256^2$), the algorithm spends more time projecting vertices into the range raster than evaluating the distance field [S1].

## Operating characteristics
**Time Complexity:**
*   The algorithm is faster than exact computation at low resolutions due to fewer distance field evaluations [S1].
*   The sequential regular algorithm is 55% faster on average than the raster algorithm [S1].
*   With acceleration structures, the average speedup is 113x compared to the sequential raster algorithm [S1].
*   Parallel implementation with BVH achieves a 120x average speedup [S1].
*   The parallel implementation achieves a maximum speedup of 11.71 on 12 hyperthreaded cores, with a throughput of 79 million tetrahedra per second [S1].

**Memory Complexity:**
*   Requires storage for the $R \times R$ raster grid [S1].

**Measured Performance:**
*   On the EthaneDiol dataset, exact signed distance field computation takes 3.471s [S1].
*   At $16^2$ resolution, the raster algorithm takes 0.555s for distance field computation and 1.242s for isosurface extraction [S1].
*   At $1024^2$ resolution, the raster algorithm takes 1.640s for distance field computation and 1.226s for isosurface extraction [S1].

**Strengths and Limitations:**
*   **Strengths:** Faster than exact computation at low resolutions; scales nearly linearly with the number of cores [S1].
*   **Limitations:** Loses geometric accuracy near FSCP bends; can produce disconnected structures; only handles closed FSCPs [S1].

## Background
The Raster Algorithm is an approximation method for fiber surface extraction, originally proposed in reference [6] and adapted to tetrahedral meshes in [S1]. It uses Marching Tetrahedra tables instead of Marching Cubes to extract the 0 level-set of a signed range distance field to the FSCP [S1].

## Sources
[S1] Fast and Exact Fiber Surfaces for Tetrahedral Meshes, https://hal.science/hal-01206120v2, fetched 2026-10-26
