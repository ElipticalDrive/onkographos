# Marching Cubes Lookup Table

Also called: MC LUT, MC Lookup Table, Tri Table, Case Table, Precomputed Triangulation Table

## Overview

The Marching Cubes Lookup Table (MC LUT) is a precomputed data structure used in the Marching Cubes algorithm to determine the topology of the polygon mesh generated within a single voxel (cube) based on the binary state of its eight corner vertices relative to an isovalue [S1, S8, S9, unofficial]. It maps the 256 possible configurations of scalar field signs (the "case index") to a list of triangle vertex indices, solving the problem of determining which edges of a voxel are intersected by the isosurface and how to connect them into triangles without runtime branching logic for topology [S8, S9, unofficial]. It sets itself apart from sibling methods like [Symmetry Mapping](869-symmetry-mapping.md) by using a direct, fixed-size array lookup for all 256 cases, prioritizing O(1) retrieval speed over memory reduction or symmetry-based compression [S7, S8, unofficial].

## How it works

**Inputs and Outputs**
*   **Input:** A scalar field value $V_i$ at each of the 8 vertices $v_i$ ($i=0..7$) of a voxel, and a scalar isovalue $I$.
*   **Output:** A list of triangles, where each triangle is defined by three 3D coordinates. The coordinates are derived by linearly interpolating the scalar field values along the cube edges where the isosurface intersects.

**Data Structures**

The method relies on two static lookup tables defined in the parent article [Lookup Table and Configuration](4978-lookup-table-and-configuration.md):

1.  **Edge Intersection Table (`edgeTable`)**: An array of 256 integers (or shorts). Each entry is a 12-bit bitmask. Bit $k$ ($0 \le k \le 11$) is set to 1 if the isosurface intersects the $k$-th edge of the cube, and 0 otherwise. The standard edge numbering convention is:
    *   Bottom face ($z=0$): $e_0=(0,1), e_1=(1,2), e_2=(2,3), e_3=(3,0)$
    *   Top face ($z=1$): $e_4=(4,5), e_5=(5,6), e_6=(6,7), e_7=(7,4)$
    *   Vertical edges: $e_8=(0,4), e_9=(1,5), e_{10}=(2,6), e_{11}=(3,7)$

2.  **Triangle Connection Table (`triangleTable`)**: A 2D array of size $256 \times 16$ [S6, S7, S8, S12, unofficial].
    *   Rows are indexed by the case index $C \in [0, 255]$.
    *   Columns store integer indices. Indices $0–11$ refer to the 12 cube edges.
    *   Indices are grouped in triplets $(a, b, c)$ to form triangles.
    *   The sequence is terminated by the value `-1`.
    *   The maximum number of triangles per voxel is 5, requiring 15 indices plus 1 terminator [S7, S12, unofficial].

**Case Index Calculation**

For each voxel, the 8-bit case index $C$ is determined by the scalar field values $V$ at the 8 vertices $v_i$ ($i=0..7$). If $V(v_i) < I$ (or $> I$, depending on convention), bit $i$ of $C$ is set to 1; otherwise 0. The bit assignment must strictly match the vertex ordering assumed by the LUT generation [S7, S8, unofficial]. The standard vertex ordering corresponds to the binary representation where $v_0$ is the least significant bit and $v_7$ is the most significant bit, or vice versa, provided the `edgeTable` and `triangleTable` are consistent with that ordering [S12, unofficial].

**Algorithm Steps**

1.  **Compute Case Index**: For the current voxel, iterate over its 8 vertices. For each vertex $i$, compare its scalar value $V_i$ with the isovalue $I$. Set bit $i$ of the case index $C$ to 1 if $V_i < I$ (assuming "inside" is below isovalue), else 0.
2.  **Lookup Edge Mask**: Retrieve the 12-bit bitmask $M = \text{edgeTable}[C]$.
3.  **Identify Intersected Edges**: Iterate over the 12 edges $e_k$. If bit $k$ of $M$ is set, the isosurface intersects edge $e_k$.
4.  **Intersect Vertices**: For each intersected edge $e_k$ connecting vertices $v_a$ and $v_b$, compute the intersection point $P_k$ using linear interpolation:
    $$ P_k = v_a + \frac{I - V_a}{V_b - V_a} (v_b - v_a) $$
    Store these points in an array `intersectPoints[12]`, indexed by edge number.
5.  **Lookup Triangulation**: Retrieve the row `triangles = triangleTable[C]`.
6.  **Generate Triangles**: Iterate through `triangles` in steps of 3. Let the triplet be $(e_a, e_b, e_c)$. If the terminator `-1` is encountered, stop. Otherwise, form a triangle using the pre-computed intersection points:
    $$ \text{Triangle} = (P_{e_a}, P_{e_b}, P_{e_c}) $$
7.  **Output**: Append the generated triangles to the mesh.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_lookup_table(volume, isovalue, grid_dims):
    """
    volume: 3D array of scalar values [x, y, z]
    isovalue: float threshold
    grid_dims: tuple (nx, ny, nz)
    Returns: list of triangles, each triangle is a list of 3 (x, y, z) tuples
    """
    triangles = []
    
    # Predefined tables (simplified representation, actual values from sources)
    # edgeTable: 256 integers
    # triangleTable: 256 x 16 integers
    # edgeVertices: 12 pairs of vertex indices (0-7)
    edge_vertices = [
        (0, 1), (1, 2), (2, 3), (3, 0), # Bottom
        (4, 5), (5, 6), (6, 7), (7, 4), # Top
        (0, 4), (1, 5), (2, 6), (3, 7)  # Vertical
    ]
    
    # Helper to get vertex position in world space
    def get_vertex_pos(v_idx, x, y, z):
        # Map v_idx 0-7 to local cube coordinates (0 or 1)
        # v0=(0,0,0), v1=(1,0,0), v2=(1,1,0), v3=(0,1,0)
        # v4=(0,0,1), v5=(1,0,1), v6=(1,1,1), v7=(0,1,1)
        # Note: Vertex ordering must match the LUT generation convention.
        # Standard convention often maps bits to vertices as follows:
        # bit 0 -> v0, bit 1 -> v1, etc.
        # We assume the LUT is consistent with this mapping.
        vx = (v_idx & 1)
        vy = (v_idx & 2) >> 1
        vz = (v_idx & 4) >> 2
        return (x + vx, y + vy, z + vz)

    for z in range(grid_dims[2] - 1):
        for y in range(grid_dims[1] - 1):
            for x in range(grid_dims[0] - 1):
                # 1. Compute Case Index
                case_index = 0
                for i in range(8):
                    # Determine vertex index in the 3D grid
                    # Mapping: 0->(0,0,0), 1->(1,0,0), 2->(1,1,0), 3->(0,1,0)
                    #          4->(0,0,1), 5->(1,0,1), 6->(1,1,1), 7->(0,1,1)
                    # This mapping must match the edgeTable/triangleTable definition.
                    # Common mapping:
                    # v0: x, y, z
                    # v1: x+1, y, z
                    # v2: x+1, y+1, z
                    # v3: x, y+1, z
                    # v4: x, y, z+1
                    # v5: x+1, y, z+1
                    # v6: x+1, y+1, z+1
                    # v7: x, y+1, z+1
                    
                    # Extract local coordinates for vertex i
                    vx = (i & 1)
                    vy = (i & 2) >> 1
                    vz = (i & 4) >> 2
                    
                    # Get scalar value at this vertex
                    # Note: Array indexing order depends on implementation (x,y,z vs z,y,x)
                    val = volume[x + vx][y + vy][z + vz]
                    
                    if val < isovalue:
                        case_index |= (1 << i)
                
                # 2. Lookup Edge Mask
                edge_mask = edgeTable[case_index]
                
                # 3. Identify Intersected Edges and Compute Intersection Points
                intersect_points = [None] * 12
                for k in range(12):
                    if (edge_mask >> k) & 1:
                        v_a, v_b = edge_vertices[k]
                        pos_a = get_vertex_pos(v_a, x, y, z)
                        pos_b = get_vertex_pos(v_b, x, y, z)
                        val_a = volume[pos_a[0]][pos_a[1]][pos_a[2]]
                        val_b = volume[pos_b[0]][pos_b[1]][pos_b[2]]
                        
                        # Linear interpolation
                        # t = (isovalue - val_a) / (val_b - val_a)
                        # P = pos_a + t * (pos_b - pos_a)
                        # Handle division by zero if val_a == val_b (edge case)
                        if abs(val_b - val_a) < 1e-9:
                            t = 0.5
                        else:
                            t = (isovalue - val_a) / (val_b - val_a)
                        
                        intersect_points[k] = (
                            pos_a[0] + t * (pos_b[0] - pos_a[0]),
                            pos_a[1] + t * (pos_b[1] - pos_a[1]),
                            pos_a[2] + t * (pos_b[2] - pos_a[2])
                        )
                
                # 4. Lookup Triangulation
                tri_row = triangleTable[case_index]
                
                # 5. Generate Triangles
                idx = 0
                while idx < 16:
                    e1 = tri_row[idx]
                    if e1 == -1:
                        break
                    e2 = tri_row[idx + 1]
                    e3 = tri_row[idx + 2]
                    
                    # Add triangle
                    triangles.append([intersect_points[e1], intersect_points[e2], intersect_points[e3]])
                    
                    idx += 3
                    
    return triangles
```

**Parameters**
*   `isovalue`: The threshold scalar value. Typical range depends on the data (e.g., -1.0 to 1.0 for normalized data) [S9, unofficial].
*   `edgeTable` and `triangleTable`: Static constants.

**Edge Cases and Pitfalls**
*   **Ambiguity**: Cases 46, 47, 98, 99, 210, 213, 214, 215 contain saddle points where the topology is ambiguous (the isosurface passes through the cube center) [S1, S7, S8, unofficial]. The standard LUT uses a fixed triangulation for these, which can cause cracks or holes if adjacent voxels resolve the ambiguity differently [S7, S8, unofficial].
*   **Terminator Check**: The implementation must check for `-1` in the `triangleTable` row to avoid reading garbage memory or out-of-bounds data [S9, S12, unofficial].
*   **Vertex Ordering**: The bit assignment for the case index must strictly match the vertex ordering assumed by the LUT generation [S7, S8, unofficial]. Mismatched ordering leads to incorrect topology.
*   **Division by Zero**: If scalar values at both ends of an edge are equal to the isovalue, linear interpolation is undefined. A guard (e.g., defaulting to midpoint) is required [G].

## Subtopics

### [Smart Vertex Interpolation with Auxiliary Arrays](1137-smart-vertex-interpolation-with-auxiliary-arrays.md)
Smart Vertex Interpolation with Auxiliary Arrays is a technique for Marching Cubes isosurface extraction that eliminates redundant vertex coordinate calculations by storing computed vertex indices in auxiliary data structures for reuse by adjacent cells. Unlike standard implementations that compute the same vertex up to four times for shared edges, this method ensures that a vertex's coordinates are calculated only once, reducing computational time and vertex table size.

### [Layer-Based Cube Traversal](1358-layer-based-cube-traversal.md)
Layer-Based Cube Traversal is a memory-optimized Marching Cubes implementation that processes a 3D scalar field by iterating through it one $z$-slice at a time. It reduces space complexity by maintaining only two layers of voxel data in memory at any given time, rather than the entire volume. This approach sets it apart from standard Marching Cubes implementations by significantly reducing spatial complexity for a grid of size $N$, making it suitable for large datasets or constrained memory environments.

### [Marching Cubes Edge Intersection and Topology Test](2251-marching-cubes-edge-intersection-and-topology-test.md)
This method resolves topological ambiguities for edge indices 0 through 10 (cases 1–11) in Marching Cubes by computing a bitmask from scalar field values at intersection points on an adjacent face. It replaces a static lookup table entry with a runtime bitwise test and arithmetic evaluation.

### [Voting Marching Cubes](11825-voting-marching-cubes.md)
Voting Marching Cubes (VMC) is a variant of the standard Marching Cubes algorithm designed to mitigate surface reconstruction errors caused by noise in silhouette images. While standard Marching Cubes requires unanimous agreement (all silhouettes must contain a projected vertex) to classify a voxel corner as "inside," VMC employs a voting threshold to tolerate individual classification errors, thereby reducing the occurrence of "lost octants" where noise prevents surface generation. It differs from its siblings by modifying the boolean classification logic of the lookup table inputs rather than the topology handling or traversal order.

### Two-Body-Saddle Block Subdivision
Two-Body-Saddle Block Subdivision is a topological analysis technique that partitions a cube containing two body saddles into simple sub-blocks using axis-aligned planes through the saddle points, enabling exact case enumeration via finite state machines [S13]. It identifies two body saddles $s_u$ and $s_l$ with distinct isovalues $u > l$, defines displacement vectors, and labels vertex isovalues to deduce gradients and resolve ambiguities that the standard LUT cannot handle deterministically [S13].

### Load-Time Marching Cubes Table Generation
Load-Time Marching Cubes Table Generation is a technique to generate the Marching Cubes triangle table at load time rather than hardcoding it, ensuring consistency between the generator and the kernel [S14, unofficial]. It generates marching squares faces for each cube face, forms closed loops, triangulates them using fan triangulation, and applies an ambiguity rule that always separates the diagonal for ambiguous faces to ensure rotation invariance and crack-free results [S14, unofficial]. It validates the generated table against the cut-edge set of every case [S14, unofficial].

## Operating characteristics

*   **Time Complexity**: O(1) per voxel for case resolution and triangle generation, as it requires only a direct array lookup and sequential reading of precomputed indices [S7, unofficial]. The overall complexity is O(N) where N is the number of voxels.
*   **Memory Complexity**: O(1) per voxel for the table, with a fixed size of 256 entries for each table [S7, unofficial]. The `triangleTable` requires $256 \times 16 \times 4$ bytes (assuming 32-bit integers) = 16 KB, and `edgeTable` requires $256 \times 4$ bytes = 1 KB [S9, S12, unofficial].
*   **Measured Performance**: In hardware-accelerated pipelines (e.g., GPU compute shaders), the LUT approach is highly efficient. For a $128^3$ volume, performance benchmarks show ~4.12 ms/frame on M-series hardware using WebGPU [S14, unofficial].
*   **Strengths**: Fast, branchless topology determination; easy to implement; suitable for real-time rendering and GPU acceleration [S7, S8, unofficial].
*   **Limitations**: Does not resolve topological ambiguities (cases 46, 47, etc.) dynamically, potentially leading to cracks or holes at saddle points if adjacent cells resolve ambiguities inconsistently [S7, S8, unofficial]. Fixed memory footprint regardless of scene complexity.
*   **When to Use**: Real-time volume rendering, medical imaging viewers, game engines, and any scenario requiring fast, deterministic isosurface extraction [S7, unofficial].
*   **When Not to Use**: Scenes with high-frequency noise causing ambiguous cases, or when memory is extremely constrained and symmetry-based compression is preferred [S1, S8, unofficial].

## Background

The Marching Cubes algorithm was introduced by William E. Lorensen and Harvey E. Cline in 1987 [S12, S14, unofficial]. The lookup table approach, specifically the `edgeTable` and `triangleTable`, was derived from the original paper and has become the standard implementation due to its simplicity and speed [S12, unofficial]. The table maps the 256 possible configurations of vertex signs to triangle indices, avoiding the need for complex runtime logic to determine topology [S8, unofficial]. Variants and improvements, such as handling ambiguous cases and optimizing memory, have been developed since, but the core LUT remains the foundation [S1, S4, unofficial][S13].

## Sources

[S1] 3D Volume Calculation For the Marching Cubes Algorithm in Cartesian Coordinates, https://arxiv.org/abs/1308.0387v1, fetched 2026-10-26 (unofficial)
[S2] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S3] keijiro/ComputeMarchingCubes, https://github.com/keijiro/ComputeMarchingCubes, fetched 2026-10-26 (unofficial)
[S4] BorisTheBrave/mc-dc, https://github.com/BorisTheBrave/mc-dc, fetched 2026-10-26 (unofficial)
[S5] Scrawk/Marching-Cubes-On-The-GPU, https://github.com/Scrawk/Marching-Cubes-On-The-GPU, fetched 2026-10-26 (unofficial)
[S6] Magnus2/MeshReconstruction, https://github.com/Magnus2/MeshReconstruction, fetched 2026-10-26 (unofficial)
[S7] CHINA-JIGE/MarchingCube, https://github.com/CHINA-JIGE/MarchingCube, fetched 2026-10-26 (unofficial)
[S8] nihaljn/marching-cubes, https://github.com/nihaljn/marching-cubes, fetched 2026-10-26 (unofficial)
[S9] mwalczyk/marching-cubes, https://github.com/mwalczyk/marching-cubes, fetched 2026-10-26 (unofficial)
[S10] ForeverDavid/marching-cubes-cuda, https://github.com/ForeverDavid/marching-cubes-cuda, fetched 2026-10-26 (unofficial)
[S11] HeartofTheForce/Isosurface, https://github.com/HeartofTheForce/Isosurface, fetched 2026-10-26 (unofficial)
[S12] Marching Cubes algorithm, https://github.com/thejasonfisher/marchingCubes/blob/master/MarchingCubes.cs, fetched 2026-10-30 (unofficial)
[S13] (No) More Marching Cubes, https://doi.org/10.2312/vg/vg07/081-090, fetched 2026-10-26
[S14] Renderer: marching-cubes isosurface mesh (r=mesh, WebGPU), https://github.com/nikolai3d/waterfall/pull/17, fetched 2026-10-27 (unofficial)
