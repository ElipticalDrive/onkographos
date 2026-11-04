# Marching Cubes Lookup Table

Also called: MC LUT, MC Lookup Table, Tri Table, Case Table, Precomputed Triangulation Table

## Overview
The Marching Cubes Lookup Table (MC LUT) is a precomputed data structure that maps the 256 possible binary configurations of a voxel's eight corner signs (relative to an isovalue) to the specific sequence of triangle vertices or edges required to triangulate the isosurface within that voxel [S3, S8, S9, unofficial]. It solves the problem of runtime topological ambiguity by replacing conditional branching logic with a direct array index, enabling O(1) determination of mesh geometry per voxel [S7, S8, unofficial]. Unlike dynamic topology resolution methods, the standard MC LUT uses a fixed triangulation for ambiguous cases, which may produce artifacts if adjacent cells resolve ambiguities inconsistently [S7, S8, unofficial].

## How it works
**Inputs:**
- A 3D scalar field $V$ sampled on a regular grid.
- An isovalue $I$.
- The precomputed lookup tables: `edgeTable` (or `cubeEdgeFlags`) and `triangleTable` (or `triangleConnectionTable`) [S2, S12, unofficial].

**Outputs:**
- A list of triangles, where each triangle is defined by three 3D vertex coordinates.

**Data Structures:**
1.  **`edgeTable`**: An array of 256 integers (or shorts). Each entry is a 12-bit bitmask. Bit $k$ (0–11) is set if the isosurface intersects the $k$-th edge of the cube [S2, S11, S12, unofficial].
    -   Edge numbering convention (standard):
        -   Bottom face ($z=0$): $e_0=(0,1), e_1=(1,2), e_2=(2,3), e_3=(3,0)$
        -   Top face ($z=1$): $e_4=(4,5), e_5=(5,6), e_6=(6,7), e_7=(7,4)$
        -   Vertical edges: $e_8=(0,4), e_9=(1,5), e_{10}=(2,6), e_{11}=(3,7)$ [S4, S12, unofficial].
2.  **`triangleTable`**: A 2D array of size $256 \times 16$ [S8, S12, unofficial].
    -   Rows are indexed by the case index (0–255).
    -   Columns store integer indices.
    -   Indices 0–11 refer to the 12 cube edges.
    -   Indices are grouped in triplets $(a, b, c)$ to form triangles.
    -   The sequence is terminated by the value `-1` [S8, S12, unofficial].
    -   Maximum triangles per voxel is 5 (15 indices + 1 terminator) [S8, S12, unofficial].

**Algorithm Steps:**
1.  **Case Index Calculation**: For each voxel, determine the 8-bit case index $C$. For each vertex $v_i$ ($i=0..7$), if $V(v_i) < I$ (or $> I$, depending on convention), set bit $i$ of $C$ to 1; otherwise 0 [S7, S8, S12, unofficial].
2.  **Edge Intersection Check**: Retrieve the bitmask $M = \text{edgeTable}[C]$. If $M == 0$, the voxel is entirely inside or outside the isosurface; no triangles are generated [S12, unofficial].
3.  **Triangle Topology Retrieval**: Retrieve the row $T = \text{triangleTable}[C]$ [S8, unofficial].
4.  **Triangle Generation Loop**: Iterate through $T$ in steps of 3. Let the triplet be $(e_a, e_b, e_c)$.
    -   If any element is `-1`, terminate the loop for this voxel [S9, S12, unofficial].
    -   For each edge index $e$, calculate the intersection point $P_e$ on the corresponding cube edge using linear interpolation between the vertex values [S6, S12, unofficial].
    -   Form a triangle with vertices $P_{e_a}, P_{e_b}, P_{e_c}$ [S12, unofficial].
5.  **Output**: Append the generated triangles to the mesh buffer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
NUM_EDGES = 12
NUM_VERTICES = 8
TERMINATOR = -1

# Edge definitions: list of (vertex_index_0, vertex_index_1)
EDGES = [
    (0, 1), (1, 2), (2, 3), (3, 0), # Bottom face
    (4, 5), (5, 6), (6, 7), (7, 4), # Top face
    (0, 4), (1, 5), (2, 6), (3, 7)  # Vertical edges
]

def interpolate(v0, v1, iso, p0, p1):
    """Linearly interpolate the intersection point on an edge."""
    if v1 == v0:
        return p0 # Avoid division by zero
    t = (iso - v0) / (v1 - v0)
    return p0 + t * (p1 - p0)

def get_triangle_vertices(case_index, edge_table, triangle_table, voxel_vertices, voxel_coords):
    """
    Generate triangles for a single voxel.
    
    Args:
        case_index: int (0-255)
        edge_table: array[256] of int (bitmasks)
        triangle_table: array[256][16] of int
        voxel_vertices: array[8] of float (scalar values at corners)
        voxel_coords: array[8] of vec3 (3D coordinates of corners)
        
    Returns:
        list of tuples (vec3, vec3, vec3)
    """
    triangles = []
    
    # 1. Get intersected edges bitmask
    edge_mask = edge_table[case_index]
    
    # 2. If no edges intersected, return empty
    if edge_mask == 0:
        return triangles

    # 3. Precompute intersection points for all intersected edges
    edge_intersections = [None] * NUM_EDGES
    for edge_idx in range(NUM_EDGES):
        # Check if bit is set
        if (edge_mask >> edge_idx) & 1:
            v0_idx, v1_idx = EDGES[edge_idx]
            val0 = voxel_vertices[v0_idx]
            val1 = voxel_vertices[v1_idx]
            p0 = voxel_coords[v0_idx]
            p1 = voxel_coords[v1_idx]
            # Note: The case_index bit assignment must match the vertex ordering
            # assumed by the edge_table generation. Standard assumes bit i corresponds to vertex i.
            edge_intersections[edge_idx] = interpolate(val0, val1, iso, p0, p1)

    # 4. Retrieve triangle topology
    tri_row = triangle_table[case_index]
    
    # 5. Iterate through triplets
    i = 0
    while i < 16:
        e_a = tri_row[i]
        if e_a == TERMINATOR:
            break
        
        e_b = tri_row[i+1]
        e_c = tri_row[i+2]
        
        # Retrieve precomputed points
        p_a = edge_intersections[e_a]
        p_b = edge_intersections[e_b]
        p_c = edge_intersections[e_c]
        
        triangles.append((p_a, p_b, p_c))
        
        i += 3
        
    return triangles
```

**Parameters:**
-   `iso`: The isovalue threshold. Typical range depends on the scalar field normalization (e.g., -1.0 to 1.0 for signed distance fields) [S9, unofficial].
-   `edge_table` / `triangle_table`: Static, precomputed constants. Values are not generated at runtime in standard implementations [S7, S12, unofficial].

**Edge Cases and Pitfalls:**
-   **Ambiguous Cases**: Cases 46, 47, 98, 99, 210, 213, 214, 215 contain saddle points where the topology is ambiguous (the isosurface passes through the cube center) [S7, S8, unofficial]. The standard LUT uses a fixed triangulation for these, which can cause cracks or holes if adjacent voxels resolve the ambiguity differently [S7, S8, unofficial].
-   **Terminator Check**: The loop must check for `-1` to prevent reading garbage memory or out-of-bounds data [S9, S12, unofficial].
-   **Vertex Ordering**: The case index bit assignment must strictly match the vertex ordering used to generate the LUT. Mismatches cause inverted normals or incorrect topology [S12, unofficial].
-   **Division by Zero**: Linear interpolation requires handling cases where vertex values are equal to the isovalue or to each other [G].

## Subtopics

### [Smart Vertex Interpolation with Auxiliary Arrays](1137-smart-vertex-interpolation-with-auxiliary-arrays.md)
A technique to avoid duplicate vertex calculation in Marching Cubes by computing vertex coordinates only the first time an active edge is considered, storing the index in auxiliary arrays to be reused by adjacent cells [1137-smart-vertex-interpolation-with-auxiliary-arrays.md]. It sets itself apart by reducing computational overhead for shared edges, trading increased memory access for reduced interpolation calculations. Choose this when interpolation cost is significant relative to lookup cost.

### [Layer-Based Cube Traversal](1358-layer-based-cube-traversal.md)
Layer-Based Cube Traversal is a memory-optimized implementation of the Marching Cubes algorithm that processes a 3D scalar field by iterating through it one $z$-slice at a time. It solves the problem of high memory consumption in naive voxel traversal by maintaining only two layers of voxel data in memory at any given time, rather than the entire volume. This approach sets it apart from standard Marching Cubes implementations by significantly reducing the spatial complexity from $O(N^3)$ to $O(N^2)$ for a grid of size $N$, making it suitable for large datasets or constrained memory environments [1358-layer-based-cube-traversal.md].

### [Marching Cubes Edge Intersection and Topology Test](2251-marching-cubes-edge-intersection-and-topology-test.md)
[2251-marching-cubes-edge-intersection-and-topology-test.md]

## Operating characteristics
-   **Time Complexity**: O(1) per voxel for topology determination, as it requires only a direct array lookup and sequential reading of precomputed indices [S7, unofficial]. Total time is O(N) where N is the number of voxels.
-   **Memory Complexity**: O(1) per voxel for the table storage, with a fixed size of 256 entries for the LUTs [S7, unofficial]. The LUTs themselves occupy negligible memory (e.g., $256 \times 16 \times 4$ bytes $\approx$ 16 KB) [S8, S12, unofficial].
-   **Measured Performance**: In hardware-accelerated pipelines (e.g., GPU compute shaders), the LUT approach enables efficient parallel processing. Benchmarks show ~4.12 ms/frame for a $128^3$ volume on M-series hardware using WebGPU [S14, unofficial].
-   **Strengths**: No runtime branching for topology; highly parallelizable; simple implementation; fast lookup.
-   **Limitations**: Ambiguous cases may produce topological artifacts (holes, spikes) if not handled by symmetry mapping or dynamic resolution [S7, S8, unofficial]. Fixed table size does not adapt to complex local geometry.
-   **When to Use**: Real-time volume rendering, medical imaging, game engines where performance is critical and topological ambiguity is rare or acceptable [S7, unofficial].
-   **When Not to Use**: When exact topological correctness is required for ambiguous cases (e.g., scientific visualization requiring precise manifold surfaces) without additional ambiguity resolution steps [S7, S8, unofficial].

## Background
The Marching Cubes algorithm was introduced by William E. Lorensen and Harvey E. Cline in 1987 [S1, S7, unofficial]. The lookup table approach was developed to optimize the algorithm by precomputing the 256 possible triangulation cases, eliminating the need for runtime decision trees. The standard table handles ambiguous cases by fixing a specific triangulation, though later work (e.g., [S4, S13, unofficial]) proposed methods to resolve ambiguities using inverse configurations or sub-block subdivision. The method is widely used in real-time graphics and scientific visualization [S7, unofficial].

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
[S13] (No) More Marching Cubes, https://doi.org/10.2312/vg/vg07/081-090, fetched 2026-10-26 (unofficial)
[S14] Renderer: marching-cubes isosurface mesh (r=mesh, WebGPU), https://github.com/nikolai3d/waterfall/pull/17, fetched 2026-10-27 (unofficial)
