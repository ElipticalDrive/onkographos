# Marching Cubes Lookup Table

Also called: MC LUT, MC Lookup Table, Tri Table, Case Table, Precomputed Triangulation Table

## Overview
The Marching Cubes Lookup Table (MC LUT) is a precomputed data structure used in the Marching Cubes algorithm to determine the topology of the polygon mesh generated within a single voxel (cube) [S1, S6, S8, S9, unofficial]. It maps the 256 possible binary configurations of scalar field signs at the 8 cube vertices (the "case index") to a list of triangle vertex indices [S9, unofficial]. By using this table, the algorithm determines which edges of a voxel are intersected by the isosurface and how to connect them into triangles without runtime branching logic for topology [S8, unofficial]. It solves the problem of efficiently generating a continuous triangular mesh from a 3D scalar field by replacing complex geometric reasoning with a simple array lookup [S8, unofficial]. Unlike siblings that optimize memory or resolve ambiguities dynamically, the standard LUT relies on a fixed static configuration for all 256 cases, including ambiguous ones [S7, S8, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field $V$ defined on a grid, an isovalue $I$, and the voxel geometry (typically a unit cube with vertices $v_0 \dots v_7$).
*   **Output:** A list of triangles, where each triangle is defined by three 3D vertices interpolated on the cube edges.

**Data Structures:**
The method relies on two precomputed data structures derived from the 256 possible binary configurations of a voxel's eight corner signs relative to an isovalue [G].

1.  **Edge Intersection Table (`edgeTable`):** An array of 256 integers (or shorts). Each entry is a 12-bit bitmask. Bit $k$ ($0 \le k \le 11$) is set to 1 if the isosurface intersects the $k$-th edge of the cube, and 0 otherwise [G]. The standard edge numbering convention is:
    *   Bottom face ($z=0$): $e_0=(0,1), e_1=(1,2), e_2=(2,3), e_3=(3,0)$ [G].
    *   Top face ($z=1$): $e_4=(4,5), e_5=(5,6), e_6=(6,7), e_7=(7,4)$ [G].
    *   Vertical edges: $e_8=(0,4), e_9=(1,5), e_{10}=(2,6), e_{11}=(3,7)$ [G].
    *   Note: Some implementations use a 256-entry array of shorts where each entry is a bitmask [S2, S11, unofficial]. Others use a 256-entry array of hexadecimal integers [S12, unofficial].

2.  **Triangle Connection Table (`triangleTable`):** A 2D array of size $256 \times 16$ [G].
    *   Rows are indexed by the case index $C \in [0, 255]$ [G].
    *   Columns store integer indices. Indices $0–11$ refer to the 12 cube edges [G].
    *   Indices are grouped in triplets $(a, b, c)$ to form triangles [G].
    *   The sequence is terminated by the value `-1` [G].
    *   The maximum number of triangles per voxel is 5, requiring 15 indices plus 1 terminator [G].
    *   Some sources describe the table as storing vertex indices directly into an `edgeVertIndices` array (0-11) [S6, unofficial], while others store edge indices that are then mapped to vertices [S12, unofficial]. The standard interpretation is that the values 0-11 refer to the edges, and the implementation must interpolate the vertex position on those edges [S8, S12, unofficial].

**Case Index Calculation:**
For each voxel, the 8-bit case index $C$ is determined by the scalar field values $V$ at the 8 vertices $v_i$ ($i=0..7$) [G]. If $V(v_i) < I$ (or $> I$, depending on convention), bit $i$ of $C$ is set to 1; otherwise 0 [G]. The bit assignment must strictly match the vertex ordering assumed by the LUT generation [G]. The standard vertex ordering is:
*   $v_0=(0,0,0), v_1=(1,0,0), v_2=(1,1,0), v_3=(0,1,0)$
*   $v_4=(0,0,1), v_5=(1,0,1), v_6=(1,1,1), v_7=(0,1,1)$ [S4, unofficial].

**Algorithm Steps:**
1.  Iterate over each voxel in the 3D grid.
2.  For the current voxel, determine the case index $C$ by comparing the scalar field values at its 8 corners against the isovalue $I$.
3.  Use $C$ to index into `triangleTable[C]`.
4.  Iterate through the resulting integer array sequentially.
5.  Read groups of three integers $(a, b, c)$. If the first integer is `-1`, stop processing this voxel.
6.  For each triplet $(a, b, c)$, these are indices into the 12 cube edges.
7.  For each edge index $e \in \{a, b, c\}$, calculate the intersection point $P_e$ on that edge using linear interpolation between the two vertices defining the edge.
8.  Output the triangle formed by vertices $P_a, P_b, P_c$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputed data structures (simplified representation)
# edgeTable: array[256] of int (bitmask)
# triangleTable: array[256][16] of int

def interpolate_vertex(edge_index, v0_pos, v1_pos, v0_val, v1_val, isovalue):
    """
    Interpolates the position of the isosurface intersection on an edge.
    edge_index: 0-11
    v0_pos, v1_pos: 3D coordinates of the edge endpoints
    v0_val, v1_val: scalar field values at the endpoints
    isovalue: the threshold I
    """
    if v1_val == v0_val:
        # Avoid division by zero; return midpoint or v0
        return (v0_pos + v1_pos) / 2.0
    
    t = (isovalue - v0_val) / (v1_val - v0_val)
    return v0_pos + t * (v1_pos - v0_pos)

def get_edge_endpoints(edge_index):
    """
    Returns the vertex indices (0-7) that define the edge.
    Standard numbering:
    e0=(0,1), e1=(1,2), e2=(2,3), e3=(3,0)
    e4=(4,5), e5=(5,6), e6=(6,7), e7=(7,4)
    e8=(0,4), e9=(1,5), e10=(2,6), e11=(3,7)
    """
    edges = [
        (0, 1), (1, 2), (2, 3), (3, 0),
        (4, 5), (5, 6), (6, 7), (7, 4),
        (0, 4), (1, 5), (2, 6), (3, 7)
    ]
    return edges[edge_index]

def process_voxel(v0_pos, v1_pos, v2_pos, v3_pos, v4_pos, v5_pos, v6_pos, v7_pos,
                  v0_val, v1_val, v2_val, v3_val, v4_val, v5_val, v6_val, v7_val,
                  isovalue, triangle_table, edge_table):
    """
    Processes a single voxel and returns a list of triangles.
    Each triangle is a tuple of 3 3D points.
    """
    # 1. Calculate Case Index
    case_index = 0
    if v0_val < isovalue: case_index |= 1 << 0
    if v1_val < isovalue: case_index |= 1 << 1
    if v2_val < isovalue: case_index |= 1 << 2
    if v3_val < isovalue: case_index |= 1 << 3
    if v4_val < isovalue: case_index |= 1 << 4
    if v5_val < isovalue: case_index |= 1 << 5
    if v6_val < isovalue: case_index |= 1 << 6
    if v7_val < isovalue: case_index |= 1 << 7
    
    # Optional: Use edgeTable to quickly check if any intersection exists
    # if edge_table[case_index] == 0: return []

    # 2. Lookup Triangles
    tri_row = triangle_table[case_index]
    triangles = []
    
    # 3. Iterate through triplets
    i = 0
    while i < 16:
        a = tri_row[i]
        if a == -1:
            break
        
        b = tri_row[i+1]
        c = tri_row[i+2]
        i += 3
        
        # 4. Intersect Edges
        # Helper to get vertex position by index
        def get_pos(idx):
            return [v0_pos, v1_pos, v2_pos, v3_pos, v4_pos, v5_pos, v6_pos, v7_pos][idx]
        
        def get_val(idx):
            return [v0_val, v1_val, v2_val, v3_val, v4_val, v5_val, v6_val, v7_val][idx]

        # Get endpoints for each edge index
        e0_v0, e0_v1 = get_edge_endpoints(a)
        e1_v0, e1_v1 = get_edge_endpoints(b)
        e2_v0, e2_v1 = get_edge_endpoints(c)
        
        # Interpolate
        p0 = interpolate_vertex(a, get_pos(e0_v0), get_pos(e0_v1), get_val(e0_v0), get_val(e0_v1), isovalue)
        p1 = interpolate_vertex(b, get_pos(e1_v0), get_pos(e1_v1), get_val(e1_v0), get_val(e1_v1), isovalue)
        p2 = interpolate_vertex(c, get_pos(e2_v0), get_pos(e2_v1), get_val(e2_v0), get_val(e2_v1), isovalue)
        
        triangles.append((p0, p1, p2))
        
    return triangles
```

**Parameters:**
*   `isovalue`: The threshold value for the scalar field. Typical range depends on the data (e.g., -1.0 to 1.0 for normalized data) [S9, unofficial].
*   `triangleTable`: The static 256x16 array.

**Edge Cases and Pitfalls:**
*   **Ambiguity:** Cases 46, 47, 98, 99, 210, 213, 214, 215 contain saddle points where the topology is ambiguous (the isosurface passes through the cube center) [G]. The standard LUT uses a fixed triangulation for these, which can cause cracks or holes if adjacent voxels resolve the ambiguity differently [G].
*   **Terminator Check:** The implementation must check for `-1` to avoid reading garbage memory or out-of-bounds data when constructing triangles [S9, S12, unofficial].
*   **Precision:** Linear interpolation assumes the scalar field varies smoothly. Discontinuities or noise can lead to incorrect edge intersections [S7, unofficial].
*   **Case 22:** Some specific lookup table sets mark Case 22 as N/A or require special handling, indicating a specific configuration is not defined in that set [S1, unofficial].

## Subtopics

### [Smart Vertex Interpolation with Auxiliary Arrays](1137-smart-vertex-interpolation-with-auxiliary-arrays.md)
Smart Vertex Interpolation with Auxiliary Arrays is a technique for Marching Cubes isosurface extraction that eliminates redundant vertex coordinate calculations by storing computed vertex indices in auxiliary data structures for reuse by adjacent cells. Unlike standard implementations that compute the same vertex up to four times for shared edges, this method ensures that a vertex's coordinates are calculated only once, reducing computational time and vertex table size.

### [Layer-Based Cube Traversal](1358-layer-based-cube-traversal.md)
Layer-Based Cube Traversal is a memory-optimized Marching Cubes implementation that processes a 3D scalar field by iterating through it one $z$-slice at a time. It reduces space complexity by maintaining only two layers of voxel data in memory at any given time, rather than the entire volume. This approach sets it apart from standard Marching Cubes implementations by significantly reducing spatial complexity for a grid of size $N$, making it suitable for large datasets or constrained memory environments.

### [Marching Cubes Edge Intersection and Topology Test](2251-marching-cubes-edge-intersection-and-topology-test.md)
This method resolves topological ambiguities for edge indices 0 through 10 (cases 1–11) in Marching Cubes by computing a bitmask from scalar field values at intersection points on an adjacent face. It replaces a static lookup table entry with a runtime bitwise test and arithmetic evaluation.

### Two-Body-Saddle Block Subdivision
A topological analysis technique that partitions a cube containing two body saddles into simple sub-blocks using axis-aligned planes through the saddle points, enabling exact case enumeration via finite state machines [S13]. It identifies two body saddles $s_u$ and $s_l$ with distinct isovalues $u > l$, defines displacement vectors, and subdivides the cube into blocks defined by planes $x=0, x_u, x_l, 1$, etc. It exploits the trilinear property that the scalar field is constant on lines co-axial with a saddle to deduce vertex isovalue orderings and resolve ambiguities in the central subblock [S13].

### Load-Time Marching Cubes Table Generation
A technique to generate the Marching Cubes triangle table at load time rather than hardcoding it, ensuring consistency between the generator and the runtime kernel [S14, unofficial]. It generates marching squares faces for each cube face, forms closed loops, triangulates them using fan triangulation, and applies an ambiguity rule that always separates the diagonal for ambiguous faces to ensure rotation invariance and crack-free results [S14, unofficial]. It validates the generated table against the cut-edge set of every case [S14, unofficial]. To prevent drift between the generator and the kernel, it interpolates literals from the generator's exports [S14, unofficial]. Note that full count symmetry does NOT hold for a face-consistent table (e.g., case 5 vs 250), but equal cut-edge sets and equal triangle counts for cases without an ambiguous face are enforced [S14, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   Time complexity per voxel is $O(1)$ for case resolution and triangle generation, as it requires only a direct array lookup and sequential reading of precomputed indices [S7, unofficial].
*   Memory complexity is $O(1)$ per voxel for the table, with a fixed size of 256 entries [S7, unofficial]. The table itself occupies $256 \times 16 \times 4$ bytes (approx. 16 KB) for standard integer types [S9, unofficial].

**Measured Performance:**
*   In a WebGPU implementation, the method achieved 4.12 ms/frame for a $128^3$ volume with $384^2$ output resolution on M-series hardware [S14, unofficial].

**Scaling:**
*   The method scales linearly with the number of voxels, as each voxel is processed independently [S7, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Extremely fast due to $O(1)$ lookup; no branching logic for topology; widely used in real-time volume rendering, medical imaging viewers, and game engines [S7, unofficial].
*   **Limitations:** Does not resolve topological ambiguities (e.g., cases 210, 213) dynamically, which may result in visual artifacts like holes or cracks if adjacent cubes resolve ambiguities differently [S8, unofficial]. The fixed triangulation for ambiguous cases can produce topological artifacts (holes or spikes) at the cube center when the scalar field has a saddle point [S7, unofficial].

**When to Use:**
*   Use when performance is critical and the scalar field does not have complex saddle points that require topological correctness, or when combined with ambiguity resolution techniques.

**When Not to Use:**
*   Do not use if topological correctness is paramount and the scalar field contains saddle points, unless combined with a method like [Marching Cubes Edge Intersection and Topology Test](2251-marching-cubes-edge-intersection-and-topology-test.md) or [Two-Body-Saddle Block Subdivision](#two-body-saddle-block-subdivision).

## Background
The Marching Cubes algorithm was introduced by William E. Lorensen and Harvey E. Cline in 1987 [S1, S8, unofficial]. The lookup table approach was developed to optimize the algorithm by precomputing the triangulation for all 256 possible vertex configurations [S3, S8, unofficial]. The standard table is widely cited from the original paper and subsequent implementations [S4, S12, unofficial]. The ambiguity problem in cases 46, 47, 98, 99, 210, 213, 214, 215 was identified early on, leading to various extensions and alternative table generation methods [S4, S7, S8, unofficial].

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
