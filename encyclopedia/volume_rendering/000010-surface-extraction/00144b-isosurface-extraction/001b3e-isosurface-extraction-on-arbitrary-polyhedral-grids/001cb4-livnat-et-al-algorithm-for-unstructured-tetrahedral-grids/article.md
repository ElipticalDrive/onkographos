# Livnat et al. algorithm for unstructured tetrahedral grids

Also called: Livnat's algorithm, GPU-based marching tetrahedra, hardware-accelerated isosurface extraction on tetrahedral grids

## Overview
This method extracts isosurfaces from unstructured 3D scalar fields defined on tetrahedral meshes by streaming cells directly into graphics hardware, computing triangles during rendering time without storing them explicitly [S1]. It sets itself apart from CPU-based extraction by leveraging vertex shaders for immediate feedback on isovalue changes, and from brute-force GPU methods by optionally using a hybrid CPU/GPU interval tree to pre-select intersecting cells, achieving up to 7x speedup over brute-force GPU rendering [S1]. Unlike structured grid methods (e.g., [Marching Cubes](52-marching-cubes.md)), it handles arbitrary tetrahedral connectivity by processing each cell's vertices through a vertex program that classifies them relative to the isovalue and outputs intersection points directly [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of tetrahedra. Each tetrahedron is defined by 4 vertices [S1]. For each vertex, the input data includes: 3D position, normal vector, scalar value, and a unique identifier $i \in \{1, 2, 3, 4\}$ indicating the vertex's role in the current tetrahedron processing batch [S1]. The global isovalue ($isoValue$) is a uniform parameter [S1].
*   **Output:** A stream of vertices forming triangle strips (length 4) that approximate the isosurface [S1]. The rasterizer discards degenerate triangles (zero area) generated for non-intersecting or single-triangle cases [S1].

**Data Structures:**
1.  **Vertex Input Structure:** Contains the entire tetrahedron's data for every input vertex to satisfy GPU pipeline constraints [S1].
    *   Fields: 4 corner positions ($\mathbf{coord}_{1..4}$, vec3), 4 corner normals ($\mathbf{nrml}_{1..4}$, vec3), 4 scalar values ($isoV$, float), 4 swizzled scalar values (for optimization), and the unique identifier $i$ [S1].
    *   Size: 144 bytes per vertex [S1].
    *   Total per tetrahedron: $4 \times 144 = 576$ bytes [S1].
2.  **Lookup Tables (Alternative Implementations):**
    *   *Table 2 (Edge Lookup):* Maps the 16 classification patterns and input vertex ID $i$ to an edge index (12, 13, 14, 23, 24, 34) or 0. Size: $16 \times 4 = 64$ entries. Each entry specifies which edge to intersect for the $i$-th output vertex [S1].
    *   *Table 3 (Edge Traverse Order):* An alternative to Table 2 to fit within older hardware limits (96 constants). Maps input vertex ID $i$ to a sequence of 3 edge tests [S1].
        *   $i=1$: Test edges 12, 13, 24.
        *   $i=2$: Test edges 14, 13, 24.
        *   $i=3$: Test edges 23, 24, 13.
        *   $i=4$: Test edges 34, 24, 13 [S1].
    *   *Edge Table & Jump Table:* Used in the Table 2 approach to determine edge masks and offsets based on the intersection bitmask [S1].

**Algorithm Steps:**
The algorithm operates in two main modes: Brute Force (GPU only) and Hybrid (CPU + GPU) [S1].

**1. Pre-processing (Hybrid Mode Only):**
*   Build an interval tree on the CPU to store the min/max scalar values of each tetrahedron [S1].
*   For a given isovalue, query the interval tree to identify tetrahedra where $min \le isoValue \le max$. These are the candidate cells [S1].

**2. Vertex Shader Processing (Per Tetrahedron):**
Four input vertices are sent to the vertex pipeline for each tetrahedron, each carrying the full tetrahedron data and a unique ID $i$ [S1].

*   **Step 1: Classification.**
    Calculate the signed distance of each vertex to the isovalue:
    $$ F_k = s_k - isoValue \quad \text{for } k \in \{1, 2, 3, 4\} $$
    Classify each vertex as positive ($+$) if $F_k > 0$ or negative ($-$) if $F_k \le 0$. This results in a 4-bit classification pattern (16 cases) [S1].

*   **Step 2: Edge Intersection Determination.**
    *   *Method A (Table 2 / Bitmask):*
        Determine intersecting edges by checking if the product of signed distances at edge endpoints is negative: $E = (F_u \cdot F_v) < 0.0$ for each edge [S1]. Create a 6-bit bitmask $E$ for the 6 edges (12, 13, 14, 23, 24, 34). Compute index: $idx = \text{dot}(E, [1, 2, 4, 8, 16, 32])$ [S1]. Lookup edge configuration from Jump Table and Edge Table to get edge weights/masks [S1].
    *   *Method B (Table 3 / Sequential Test):*
        Based on input vertex ID $i$, sequentially test the three edges specified in Table 3 [S1]. If an edge intersects (endpoints have different signs), compute the intersection point and output it. If no intersection is found after 3 tests, output a degenerate vertex $(0,0,0,0)$ [S1].

*   **Step 3: Intersection Point Calculation.**
    Identify the two active edges (or one, if only one triangle is formed) that intersect the isosurface [S1]. Calculate scalar values at the intersection points on these edges using linear interpolation. For an edge between vertices $A$ and $B$ with scalars $s_A, s_B$:
    $$ frac = \frac{isoValue - s_A}{s_B - s_A} $$
    The intersection point position $\mathbf{p}_{int}$ is:
    $$ \mathbf{p}_{int} = (1 - frac) \mathbf{p}_A + frac \mathbf{p}_B $$
    The normal $\mathbf{n}_{int}$ is interpolated similarly:
    $$ \mathbf{n}_{int} = (1 - frac) \mathbf{n}_A + frac \mathbf{n}_B $$
    *Note: In the Table 2 approach, barycentric coordinates $E$ are interpolated from edge weights, and position/normal are computed as weighted sums of all 4 vertices using these coordinates [S1].*

*   **Step 4: Output Generation.**
    *   If the tetrahedron contains 0 triangles (no intersection), output 4 degenerate vertices (e.g., $(0,0,0,0)$) [S1].
    *   If 1 triangle, output 1 valid triangle and 2 degenerate vertices [S1].
    *   If 2 triangles, output 2 valid triangles sharing an edge [S1].
    The output vertices are rendered as a triangle strip of length 4. The indices are chosen such that the stream forms the correct triangulation or degenerate patches [S1].

*   **Step 5: Transformation.**
    Transform the computed 3D position to clip space:
    $$ \mathbf{hpos} = M_{wvp} \cdot \mathbf{p}_{int} $$
    Output the homogeneous position and interpolated normal [S1].

**Pseudocode (Table 3 Approach):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_shader(IN, isoValue, worldViewProj):
    # IN contains coord1..4 (vec3), nrml1..4 (vec3), isoV1..4 (float), i (int)
    coords = [IN.coord1, IN.coord2, IN.coord3, IN.coord4]
    normals = [IN.nrml1, IN.nrml2, IN.nrml3, IN.nrml4]
    scalars = [IN.isoV1, IN.isoV2, IN.isoV3, IN.isoV4]
    i = IN.i - 1  # Convert 1-based to 0-based

    # Step 1: Classification
    F = [s - isoValue for s in scalars]

    # Helper to check if edge (u, v) intersects
    def edge_intersects(u, v):
        return (F[u] > 0) != (F[v] > 0)

    # Helper to compute intersection point on edge (u, v)
    def compute_intersection(u, v):
        frac = (isoValue - scalars[u]) / (scalars[v] - scalars[u])
        pos = coords[u] + frac * (coords[v] - coords[u])
        normal = normals[u] + frac * (normals[v] - normals[u])
        return pos, normal

    # Step 2 & 3: Sequential Edge Tests based on i (Table 3)
    # Edges are 1-based in source, converted to 0-based here
    if i == 0:
        edges = [(0, 1), (0, 2), (1, 3)]
    elif i == 1:
        edges = [(0, 3), (0, 2), (1, 3)]
    elif i == 2:
        edges = [(1, 2), (1, 3), (0, 2)]
    elif i == 3:
        edges = [(2, 3), (1, 3), (0, 2)]
    else:
        edges = []

    for u, v in edges:
        if edge_intersects(u, v):
            pos, normal = compute_intersection(u, v)
            hpos = [sum(worldViewProj[r][c] * pos[c] for c in range(3)) + worldViewProj[r][3] for r in range(4)]
            return {"HPOS": hpos, "COL0": normal}

    # Step 4: No intersection found for this vertex role
    return {"HPOS": [0, 0, 0, 1], "COL0": [0, 0, 0]}
```

**Parameters:**
*   `isoValue`: The scalar threshold. Typically a float.
*   `worldViewProj`: The combined transformation matrix.
*   **Memory:** 144 bytes per vertex input. 576 bytes per tetrahedron streamed [S1].

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** GPU vertex programs cannot generate variable numbers of outputs. Cases with 0 or 1 triangle must output degenerate (zero-area) triangles, which are discarded by the rasterizer [S1].
*   **Memory Limits:** If the tetrahedral grid exceeds video memory, transferring data via AGP bus causes a 10-20x performance drop [S1].
*   **Isovalue Changes:** Changing the isovalue in Hybrid mode requires updating the interval tree nodes in video memory, causing a slight frame rate decrease [S1].
*   **Precision:** Linear interpolation assumes linear scalar variation within the tetrahedron.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of tetrahedra processed. In Brute Force mode, all tetrahedra are processed. In Hybrid mode, only intersecting tetrahedra are processed, reducing $N$ by ~70% for typical datasets [S1].
*   **Memory Complexity:** $O(N)$ for storing the tetrahedral data in video memory. 576 bytes per tetrahedron [S1].
*   **Performance:**
    *   On nVidia GeForce FX 5800 Ultra (128MB) with elec2 dataset (142,131 tetrahedra): 9.92 million tetrahedra/sec (diffuse lighting) [S1].
    *   On nVidia GeForce Quadro FX 3000 (256MB) with elec1 dataset (19,549 tetrahedra): 7.22 million tetrahedra/sec (diffuse lighting) [S1].
    *   Hybrid CPU/GPU version achieves ~7x speedup over GPU Brute Force [S1].
    *   Pure CPU implementation achieves about half the performance of the hardware accelerated method [S1].
*   **Strengths:** Immediate feedback for isovalue changes; high throughput when data fits in VRAM; handles unstructured grids natively [S1].
*   **Limitations:** High memory usage per tetrahedron (576 bytes); performance degrades significantly if data does not fit in VRAM; requires modern GPU with vertex shader support [S1].

## Background
The algorithm was introduced by Livnat et al. in 2001/2002 for realtime isosurface extraction using graphics hardware [S1]. It addresses the limitations of CPU-based extraction and the memory overhead of storing explicit triangle meshes. The method leverages the parallel processing power of GPUs to compute isosurfaces on the fly. The hybrid approach incorporates interval trees for spatial selection, a technique previously used in CPU-based volume rendering [S1].

## Sources
[S1] Realtime Isosurface Extraction with Graphics Hardware, https://doi.org/10.2312/egs.20041007, fetched 2026-10-26
[S2] marching cubes algorithm, https://www.sciencedirect.com/topics/computer-science/marching-cube-algorithm, fetched 2026-10-30 (book)
