# Primal Grid Marching Cubes

Also called: Compute Shader Marching Cubes, Marching Cube

## Overview
Primal Grid Marching Cubes is a GPU-accelerated isosurface extraction technique that processes a 3D scalar volume grid using compute shaders to output a triangle mesh [S1, S2, unofficial][S3, unofficial]. It solves the problem of converting discrete volumetric data into a continuous geometric representation suitable for rasterization or ray tracing [S3, unofficial]. Unlike direct volume rendering, which visualizes the volume directly, this method creates an intermediate polygonal mesh [S3, unofficial]. It differs from its parent [Isosurface extraction](30-isosurface-extraction.md) by explicitly constructing a triangle mesh via lookup tables and atomic counters rather than just traversing the grid [S1, unofficial], and from [Dual Marching Cubes](906-dual-marching-cubes.md) by operating on primal grid vertices rather than dual grid cell centers .

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field (voxel grid) stored in a texture or structured buffer, an isovalue threshold, grid dimensions, and a triangle budget [S1, S2, unofficial][S3, unofficial].
*   **Output:** A triangle mesh consisting of vertex positions, vertex normals, and an index buffer defining the triangles [S1, S3, unofficial].

**Data Structures:**
1.  **Triangle Table (`TriangleTable`):** A structured buffer of `uint2` (or `int8_t[256][16]`), size 256. Each entry stores the triangle configuration for one of the 256 possible cube states. The data packs edge indices such that the first 8 edges are in the lower 32 bits of the first `uint`, and edges 8-15 are in the lower 32 bits of the second `uint` [S1, S2, unofficial]. Entries use `-1` (or `0xff` in packed form) to indicate no triangle for a given edge configuration [S2, unofficial].
2.  **Counter Buffer:** A compute buffer of type `Counter`, size 1 `uint`, used to atomically allocate triangle slots in the vertex and index buffers [S1, unofficial].
3.  **Vertex Buffer:** A `RWByteAddressBuffer` storing vertex positions (`float3`) and normals (`float3`) sequentially. Each vertex occupies 24 bytes (12 bytes for position, 12 bytes for normal) [S1, unofficial].
4.  **Index Buffer:** A `RWByteAddressBuffer` storing triangle indices. Each triangle consists of 3 `uint32` indices, occupying 12 bytes [S1, unofficial].
5.  **Voxel Data:** A `StructuredBuffer<float>` or 3D texture containing the scalar field values [S1, S3, unofficial].

**Numbered Steps:**

1.  **Initialization:** The host allocates the `TriangleTable`, `Counter`, `VertexBuffer`, and `IndexBuffer`. The `Counter` is reset to 0. The `VertexBuffer` and `IndexBuffer` are sized based on a `MaxTriangle` budget [S1, unofficial].
2.  **Dispatch:** A compute shader kernel (`MeshReconstruction`) is launched with a thread group size of `[4, 4, 4]` (or similar), mapping global thread IDs to voxel coordinates $(i, j, k)$ [S1, S2, unofficial][S3, unofficial].
3.  **Boundary Check:** Each thread checks if its voxel coordinate is within bounds. If `id` exceeds grid dimensions in any component, the thread returns immediately [S1, unofficial].
4.  **Sample Vertices:** For the current voxel cube, sample the scalar values and gradients at the 8 vertices. For vertex index $i \in [0, 7]$, calculate the grid coordinate `id + CubeVertex(i)`. `CubeVertex(i)` returns `uint3(x, y, z)` where $x, y, z$ are the bits of $i$. Store results in `float4 samples[8]`, where `xyz` is the gradient and `w` is the scalar value [S1, unofficial].
5.  **Calculate Case Index:** Compute the cube configuration selector (bitmask) $C$. For each vertex $i$, if `samples[i].w < Isovalue`, set bit $i$ in the selector:
    $$ C = \sum_{i=0}^{7} \mathbb{I}(\text{samples}[i].w < \text{Isovalue}) \cdot 2^i $$
    If $C = 0$ or $C = 0xFF$, the cube does not intersect the isosurface; return immediately [S1, S2, unofficial][S3, unofficial].
6.  **Retrieve Configuration:** Look up the triangle configuration data for selector $C$ from the `TriangleTable`: `uint2 tri_data = TriangleTable[C]` [S1, unofficial].
7.  **Iterate Edges:** Iterate through the triangle data in steps of 3 ($i = 0, 3, 6, 9, 12$). Extract three edge indices $e_1, e_2, e_3$ using `EdgeIndexFromTriangleTable(tri_data, i)`. If $e_1 == 15$ (or -1), the configuration is exhausted; return [S1, S2, unofficial].
8.  **Compute Intersections:** For each edge index $e \in \{e_1, e_2, e_3\}$:
    *   Get the vertex pair using `EdgeVertexPair(e)` [S1, unofficial].
    *   Calculate the interpolation parameter:
        $$ p = \frac{\text{Isovalue} - \text{samples}[v_1].w}{\text{samples}[v_2].w - \text{samples}[v_1].w} $$
    *   Compute the vertex position:
        $$ \mathbf{pos} = \text{TransformPoint}(\text{lerp}(\mathbf{v}_1, \mathbf{v}_2, p)) $$
    *   Compute the normal:
        $$ \mathbf{normal} = -\text{normalize}(\text{lerp}(\text{samples}[v_1].xyz, \text{samples}[v_2].xyz, p)) $$
    *   `TransformPoint(p)` maps grid coordinates to object space: $\mathbf{p} + 0.5 - \text{Dims}/2 \times \text{Scale}$ [S1, unofficial].
9.  **Atomic Allocation:** Atomically increment the `Counter` to get the current triangle index count: `count = Counter.IncrementCounter()`. If `count >= MaxTriangle`, return to prevent buffer overflow [S1, unofficial].
10. **Write Output:** Calculate vertex index `vidx = count * 3`. Write the three computed vertices to the `VertexBuffer` at offsets `vidx*24`, `(vidx+1)*24`, `(vidx+2)*24`. Write indices `vidx`, `vidx+1`, `vidx+2` to the `IndexBuffer` at offset `vidx*12` [S1, unofficial].
11. **Cleanup:** After the reconstruction dispatch, run a `ClearUnused` kernel. This kernel atomically increments the counter until it reaches `MaxTriangle`, writing zeroed vertices and indices to the remaining buffer space to ensure clean data for the mesh [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Kernel: MeshReconstruction
# Each thread processes one voxel cube at grid coordinate `id` (uint3)

def MeshReconstruction(id: uint3, Dims: uint3, Isovalue: float, Scale: float, MaxTriangle: uint):
    # 1. Boundary Check: Dims is the number of cells. If id >= Dims, return.
    if any(id >= Dims):
        return

    # 2. Sample 8 vertices of the cube
    samples = [float4(0.0, 0.0, 0.0, 0.0)] * 8
    for i in range(8):
        x = (i >> 0) & 1
        y = (i >> 1) & 1
        z = (i >> 2) & 1
        # Standard primal grid vertex mapping
        offset = uint3(x, y, z)
        coord = id + offset
        samples[i] = VoxelValueWithGradient(coord, Dims)

    # 3. Calculate Case Index (Selector)
    selector = 0
    for i in range(8):
        if samples[i].w < Isovalue:
            selector |= (1 << i)

    # 4. Early Exit for non-intersecting cubes
    if selector == 0 or selector == 0xFF:
        return

    # 5. Retrieve Triangle Configuration
    tri_data = TriangleTable[selector]

    # 6. Process Triangles
    # Each uint2 in tri_data holds edge indices. 
    # Lower 32 bits of tri_data.x: edges 0-7
    # Lower 32 bits of tri_data.y: edges 8-15
    # We iterate through 5 triangles (15 edge indices)
    
    current_tri = 0
    while current_tri < 5:
        # Extract 3 edge indices for the current triangle
        e1 = GetEdgeIndex(tri_data, current_tri * 3)
        e2 = GetEdgeIndex(tri_data, current_tri * 3 + 1)
        e3 = GetEdgeIndex(tri_data, current_tri * 3 + 2)

        # Check for end of configuration (15 or -1)
        if e1 == 15 or e1 == -1:
            break

        # 7. Compute Vertices for this triangle
        verts = [float3(0.0, 0.0, 0.0)] * 3
        normals = [float3(0.0, 0.0, 0.0)] * 3

        for k in range(3):
            edge_idx = [e1, e2, e3][k]
            
            # Get vertex pair for edge
            v1, v2 = EdgeVertexPair(edge_idx)
            
            # Interpolate position
            val1 = samples[v1].w
            val2 = samples[v2].w
            denom = val2 - val1
            
            if denom == 0:
                param = 0.5 # Guard against division by zero [G]
            else:
                param = (Isovalue - val1) / denom

            # Lerp positions in grid space
            coord1 = id + CubeVertex(v1)
            coord2 = id + CubeVertex(v2)
            grid_pos = lerp(coord1, coord2, param)
            
            # Transform to object space
            verts[k] = TransformPoint(grid_pos, Dims, Scale)
            
            # Lerp normals (gradients)
            grad1 = samples[v1].xyz
            grad2 = samples[v2].xyz
            normals[k] = -normalize(lerp(grad1, grad2, param))

        # 8. Atomic Counter
        count = Counter.IncrementCounter()
        if count >= MaxTriangle:
            return

        # 9. Write to Buffers
        vidx = count * 3
        WriteVertex(VertexBuffer, vidx * 24, verts[0], normals[0])
        WriteVertex(VertexBuffer, (vidx + 1) * 24, verts[1], normals[1])
        WriteVertex(VertexBuffer, (vidx + 2) * 24, verts[2], normals[2])
        
        WriteIndices(IndexBuffer, vidx * 12, uint3(vidx, vidx + 1, vidx + 2))

        current_tri += 1

# Helper: CubeVertex maps bit index i to grid offset (x, y, z)
def CubeVertex(i: uint) -> uint3:
    x = i & 1
    y = (i >> 1) & 1
    z = (i >> 2) & 1
    return uint3(x, y, z)

# Helper: EdgeVertexPair maps edge index to vertex indices
def EdgeVertexPair(edge_idx: uint) -> tuple:
    if edge_idx < 4:
        return edge_idx, (edge_idx + 1) & 3
    elif edge_idx < 8:
        return edge_idx, (edge_idx + 1) & 7 if edge_idx < 7 else 4
    else:
        v = edge_idx - 8
        return v, v + 4

# Helper: GetEdgeIndex extracts 4-bit edge index from packed uint2
def GetEdgeIndex(tri_data: uint2, index: uint) -> int:
    if index < 8:
        val = tri_data.x
        shift = (index % 4) * 4
        return (val >> shift) & 0xF
    else:
        val = tri_data.y
        shift = ((index - 8) % 4) * 4
        return (val >> shift) & 0xF

# Helper: VoxelValueWithGradient samples scalar and gradient
def VoxelValueWithGradient(coord: uint3, Dims: uint3) -> float4:
    # Clamp coordinates to grid bounds for gradient calculation
    c = uint3(max(coord.x, 1) - 1, max(coord.y, 1) - 1, max(coord.z, 1) - 1)
    p = uint3(min(coord.x + 1, Dims.x - 1), min(coord.y + 1, Dims.y - 1), min(coord.z + 1, Dims.z - 1))
    # Assume Voxels is a global 3D array
    v = Voxels[c.x + Dims.x * (c.y + Dims.y * c.z)]
    v_nx = Voxels[(c.x - 1) + Dims.x * (c.y + Dims.y * c.z)]
    v_px = Voxels[(p.x - 1) + Dims.x * (c.y + Dims.y * c.z)]
    v_ny = Voxels[c.x + Dims.x * ((c.y - 1) + Dims.y * c.z)]
    v_py = Voxels[c.x + Dims.x * ((p.y - 1) + Dims.y * c.z)]
    v_nz = Voxels[c.x + Dims.x * (c.y + Dims.y * (c.z - 1))]
    v_pz = Voxels[c.x + Dims.x * (c.y + Dims.y * (p.z - 1))]
    return float4(v_px - v_nx, v_py - v_ny, v_pz - v_nz, v)

# Helper: TransformPoint maps grid coordinates to object space
def TransformPoint(grid_coord: uint3, Dims: uint3, Scale: float) -> float3:
    p = float3(grid_coord)
    return (p + 0.5 - float3(Dims) / 2.0) * Scale

# Helper: Atomic Counter
class Counter:
    _value = 0
    @staticmethod
    def IncrementCounter() -> uint:
        Counter._value += 1
        return Counter._value - 1

# Helper: WriteVertex
def WriteVertex(buffer: RWByteAddressBuffer, offset: uint, pos: float3, norm: float3):
    buffer.Store(offset, pos.x)
    buffer.Store(offset + 4, pos.y)
    buffer.Store(offset + 8, pos.z)
    buffer.Store(offset + 12, norm.x)
    buffer.Store(offset + 16, norm.y)
    buffer.Store(offset + 20, norm.z)

# Helper: WriteIndices
def WriteIndices(buffer: RWByteAddressBuffer, offset: uint, indices: uint3):
    buffer.Store(offset, indices.x)
    buffer.Store(offset + 4, indices.y)
    buffer.Store(offset + 8, indices.z)
```

**Parameters:**
*   `Isovalue`: The threshold value for isosurface extraction; vertices with values below this are considered inside [S1, S3, unofficial].
*   `MaxTriangle`: The maximum number of triangles allowed in the output mesh, used to prevent buffer overflow [S1, unofficial].
*   `Scale`: A scalar factor used to transform grid coordinates to object space [S1, unofficial].
*   `Dims`: The dimensions of the voxel grid $(x, y, z)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero/Full Cube:** If the selector is 0 or 0xFF, the cube is skipped [S1, unofficial].
*   **Buffer Overflow:** If the counter exceeds `MaxTriangle`, further writes are dropped [S1, unofficial].
*   **Division by Zero:** If the scalar values at the two endpoints of an edge are equal, the interpolation parameter is undefined. A guard value (e.g., 0.5) should be used [G].
*   **Ambiguity:** Standard Marching Cubes suffers from topological ambiguity in certain configurations (e.g., Type A hole). This implementation uses the standard lookup table, which may produce discontinuities or holes in such cases [S7].

## Subtopics

### CPU Marching Cubes Fallback
A CPU-based fallback for Marching Cubes in CuMCubes, which samples a function at grid points to create a density grid, then calls the GPU Marching Cubes implementation after transferring the grid to the GPU [S2, unofficial]. It iterates over the sample points grid using nested loops, calls the user-provided Python function to compute density, stores the result in a CPU tensor, transfers it to the GPU, and then invokes the GPU wrapper [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N^3)$ for a grid of $N$ points per axis, as each voxel is processed in parallel [S4, unofficial].
*   **Memory Complexity:** $O(N^3)$ for the geometry buffer, as each grid cell outputs a fixed maximum of 15 vertices (5 triangles) [S4, unofficial]. The vertex and index buffers are pre-allocated to `MaxTriangle` [S1, unofficial].
*   **Performance:** Performance scales with the number of voxels and the fraction of intersecting cubes. The use of atomic counters introduces contention, which can reduce throughput at high triangle counts [S1, S2, unofficial].
*   **Strengths:** Deterministic memory usage due to fixed-size buffers; GPU-friendly parallelism; supports both precomputed tensors and on-the-fly implicit function evaluation [S1, S2, unofficial].
*   **Limitations:** Atomic counter contention and potential for buffer overflow if `MaxTriangle` is underestimated [S1, unofficial]; topological ambiguities in standard MC configurations [S7].
*   **When to Use:** When real-time or high-performance isosurface extraction from volumetric data is required on a GPU.
*   **When Not to Use:** When memory is extremely constrained and the triangle count is unpredictable (use hierarchical methods); when topological correctness in ambiguous cases is critical (use Dual Marching Cubes or Ambiguous Case Resolvers).

## Background
The method is based on the classic Marching Cubes algorithm by Lorensen and Cline , adapted for GPU compute shaders. The triangle table data is sourced from Paul Bourke's 'Polygonise' geometry reference, which is attributed to the original MC paper [S2, S1, unofficial]. The specific implementation details (atomic counters, buffer layout) are derived from Unity's ComputeMarchingCubes sample and CUDA extensions like CuMCubes [S1, S2, unofficial].

## Sources
[S1] keijiro/ComputeMarchingCubes, https://github.com/keijiro/ComputeMarchingCubes, fetched 2026-10-26 (unofficial)
[S2] lzhnb/CuMCubes, https://github.com/lzhnb/CuMCubes, fetched 2026-10-26 (unofficial)
[S3] mwalczyk/marching-cubes, https://github.com/mwalczyk/marching-cubes, fetched 2026-10-26 (unofficial)
[S4] ForeverDavid/marching-cubes-cuda, https://github.com/ForeverDavid/marching-cubes-cuda, fetched 2026-10-26 (unofficial)
[S5] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S6] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S7] Local hull-based surface construction of volumetric data from silhouettes, https://doi.org/10.1109/tip.2008.926149, fetched 2026-10-30
