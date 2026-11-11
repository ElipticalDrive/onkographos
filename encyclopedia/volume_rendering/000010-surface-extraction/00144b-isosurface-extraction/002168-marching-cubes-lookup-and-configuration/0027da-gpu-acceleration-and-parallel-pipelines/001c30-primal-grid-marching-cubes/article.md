# Primal Grid Marching Cubes

Also called: Compute Shader Marching Cubes, GPU Marching Cubes (Compute Shader), Marching Cubes (CUDA), Marching Lines

## Overview
Primal Grid Marching Cubes is a GPU-accelerated isosurface extraction technique that processes a 3D scalar volume grid using compute shaders to output a triangle mesh [S1, S2, unofficial]. It solves the problem of real-time or high-performance surface reconstruction from discrete 3D array data by mapping each voxel to a parallel thread, utilizing precalculated lookup tables for topology and atomic counters for dynamic buffer management [S1, S2, unofficial]. It sets itself apart from CPU-based immediate mode implementations by leveraging massive parallelism and fixed-size buffers with cleanup passes, and from slab-based variants by processing the entire volume grid directly without explicit spatial partitioning for memory locality [S1, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field (voxel grid) of size $N_x \times N_y \times N_z$, an isovalue threshold $T$, and a transformation scale $S$ [S1, S2, S4, unofficial]. The scalar field may be a precomputed tensor or an implicit function evaluated on-the-fly [S2, S3, unofficial].
*   **Output:** A triangle mesh consisting of vertex positions, normals, and indices, stored in structured buffers (Vertex Buffer, Index Buffer) [S1, S3, unofficial].

**Data Structures:**
1.  **Triangle Table:** A lookup table of size 256, where each entry corresponds to one of the 256 possible binary configurations of a cube's 8 vertices [S1, S2, S4, unofficial].
    *   *Layout A (Unity/ComputeMarchingCubes):* `StructuredBuffer<uint2>` or `ComputeBuffer` of 256 `ulong` values. Each `ulong` (64 bits) packs edge indices: the lower 32 bits store edges 0–7, and the upper 32 bits store edges 8–15 [S1, unofficial].
    *   *Layout B (CuMCubes):* `int8_t triangle_table[256][16]`. Each row contains up to 16 signed 8-bit integers representing vertex edge indices (0–11) or -1 for empty slots [S2, unofficial].
2.  **Edge Table:** A lookup table of size 256, where each entry is a 16-bit bitmask indicating which of the 12 cube edges are intersected by the isosurface [S2, S3, unofficial].
3.  **Output Buffers:**
    *   **Vertex Buffer:** A `RWByteAddressBuffer` storing vertex positions (float3) and normals (float3) sequentially. Each vertex occupies 24 bytes (12 bytes position, 12 bytes normal) [S1, unofficial].
    *   **Index Buffer:** A `RWByteAddressBuffer` storing triangle indices. Each triangle consists of 3 `uint32` indices, occupying 12 bytes [S1, unofficial].
    *   **Counter:** A `ComputeBuffer` of type Counter, size 1 `uint`, used to atomically allocate triangle slots [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Allocate the Triangle Table, Vertex Buffer, Index Buffer, and Counter [S1, unofficial].
    *   The Vertex Buffer capacity is determined by a `MaxTriangle` budget, typically sized to $(N_x-1)(N_y-1)(N_z-1) \times 15$ to accommodate the maximum 5 triangles (15 vertices) per voxel [S1, S4, unofficial].
    *   Set the isovalue $T$ and scale $S$ [S1, S3, unofficial].

2.  **Dispatch:**
    *   Launch a compute shader kernel with a thread group size of $[4, 4, 4]$ (or similar), mapping the 3D grid of threads to the 3D volume grid [S1, S2, unofficial].
    *   Each thread $(i, j, k)$ is responsible for processing the voxel cube defined by grid coordinates $(i, j, k)$ [S1, S2, unofficial].

3.  **Boundary Check:**
    *   If any component of $(i+1, j+1, k+1)$ is greater than or equal to the corresponding dimension $(N_x, N_y, N_z)$, the thread returns immediately to avoid out-of-bounds memory access when sampling corner vertices [S1, S2, unofficial].

4.  **Voxel Sampling:**
    *   For the current voxel, sample the scalar values and gradients at its 8 corner vertices [S1, unofficial].
    *   The corner vertices are indexed $0$ to $7$. The grid coordinate for vertex $v$ is calculated as:
        $$ \text{Corner}(v) = (i + (v \& 1), j + ((v >> 1) \& 1), k + ((v >> 2) \& 1)) $$
        [S1, S6, unofficial].
    *   Store results in `samples[8]`, where `samples[v]` contains the gradient $(g_x, g_y, g_z)$ and scalar value $w$ [S1, unofficial].

5.  **Configuration Selector:**
    *   Calculate the cube configuration index $C$ (0–255) by comparing each vertex's scalar value $w_v$ to the isovalue $T$ [S1, S4, unofficial]:
        $$ C = \sum_{v=0}^{7} \left( w_v < T \right) \ll v $$
        [S1, S4, unofficial].
    *   If $C = 0$ or $C = 255$, the cube does not intersect the isosurface; return immediately [S1, S2, unofficial].

6.  **Edge Intersection and Normal Calculation:**
    *   Retrieve the intersected edges from the Edge Table for configuration $C$ [S2, S3, unofficial].
    *   For each intersected edge $e$ (0–11):
        *   Identify the two vertex indices $v_1, v_2$ connected by edge $e$ using the Edge Vertex Pair mapping [S1, unofficial].
        *   Calculate the interpolation parameter $p$:
            $$ p = \frac{T - w_{v_1}}{w_{v_2} - w_{v_1}} $$
            [S1, unofficial].
        *   Calculate the intersection point position $P_{edge}$ and normal $N_{edge}$:
            $$ P_{edge} = \text{TransformPoint}(\text{Lerp}(\text{Pos}(v_1), \text{Pos}(v_2), p)) $$
            $$ N_{edge} = -\text{Normalize}(\text{Lerp}(\text{Grad}(v_1), \text{Grad}(v_2), p)) $$
            [S1, unofficial].
        *   *Note:* Some implementations use simple midpoint interpolation $(P_{v_1} + P_{v_2})/2$ if the field is binary or for performance, but linear interpolation is standard for smooth fields [S4, S1, unofficial].

7.  **Triangle Generation:**
    *   Retrieve the triangle configuration data for $C$ from the Triangle Table [S1, S2, unofficial].
    *   Iterate through the edge indices in the table. For each triplet of edge indices $(e_1, e_2, e_3)$:
        *   If the index is -1 (or 15 in some layouts), the configuration is exhausted [S1, S2, unofficial].
        *   Retrieve the pre-calculated intersection points for $e_1, e_2, e_3$.
        *   Form a triangle from these three points [S1, S4, unofficial].

8.  **Buffer Writing:**
    *   Atomically increment the global counter to claim a slot in the output buffers:
        $$ \text{count} = \text{Counter.IncrementCounter}() $$
        [S1, unofficial].
    *   If $\text{count} \ge \text{MaxTriangle}$, return to prevent buffer overflow [S1, unofficial].
    *   Write the 3 vertices of the triangle to the Vertex Buffer at offset $\text{count} \times 3 \times 24$ bytes [S1, unofficial].
    *   Write the 3 indices $(\text{count} \times 3, \text{count} \times 3 + 1, \text{count} \times 3 + 2)$ to the Index Buffer at offset $\text{count} \times 3 \times 4$ bytes [S1, unofficial].

9.  **Cleanup (Optional but Recommended):**
    *   Run a second kernel to clear unused portions of the buffers. This kernel atomically increments the counter until it reaches `MaxTriangle`, writing zeroed vertices and indices to the remaining space to ensure clean data for the mesh [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Kernel: MeshReconstruction
# Inputs:
#   Voxels: float[] (scalar field)
#   Dims: uint3 (Nx, Ny, Nz)
#   Isovalue: float
#   Scale: float
#   TriangleTable: uint2[] (size 256)
#   EdgeTable: uint16[] (size 256)
#   VertexBuffer: RWByteAddressBuffer
#   IndexBuffer: RWByteAddressBuffer
#   Counter: Counter

[numthreads(4, 4, 4)]
void MeshReconstruction(uint3 id : SV_DispatchThreadID) {
    // 1. Boundary Check
    if (id.x + 1 >= Dims.x || id.y + 1 >= Dims.y || id.z + 1 >= Dims.z) {
        return;
    }

    // 2. Sample 8 Vertices
    float4 samples[8];
    for (uint i = 0; i < 8; i++) {
        uint3 corner = id + CubeVertex(i);
        samples[i] = VoxelValueWithGradient(corner);
    }

    // 3. Calculate Configuration Selector
    uint selector = 0;
    for (uint i = 0; i < 8; i++) {
        if (samples[i].w < Isovalue) {
            selector |= (1 << i);
        }
    }

    // 4. Early Exit for Non-Intersecting Cubes
    if (selector == 0 || selector == 255) {
        return;
    }

    // 5. Calculate Edge Intersections
    float3 edge_points[12];
    float3 edge_normals[12];
    
    uint edges = EdgeTable[selector];
    for (uint e = 0; e < 12; e++) {
        if ((edges >> e) & 1) {
            uint2 v_pair = EdgeVertexPair(e);
            uint v1 = v_pair.x;
            uint v2 = v_pair.y;
            
            float w1 = samples[v1].w;
            float w2 = samples[v2].w;
            float3 p1 = TransformPoint(samples[v1].xyz); // Assuming xyz stored in sample or looked up
            float3 p2 = TransformPoint(samples[v2].xyz);
            float3 g1 = samples[v1].xyz; // Gradient
            float3 g2 = samples[v2].xyz; // Gradient
            
            float t = (Isovalue - w1) / (w2 - w1);
            edge_points[e] = lerp(p1, p2, t);
            edge_normals[e] = -normalize(lerp(g1, g2, t));
        }
    }

    // 6. Generate Triangles
    uint2 tri_data = TriangleTable[selector];
    // Layout: Lower 32 bits of tri_data.x are edges 0-7, Upper 32 bits are 8-15?
    // Or as per S1: uint2 stores packed edge indices.
    // Simplified iteration assuming tri_data contains edge indices for triangles
    
    // Note: Exact parsing of TriangleTable depends on implementation (S1 vs S2).
    // Using S1 style: uint2 packed.
    uint edges_packed = tri_data.x | (tri_data.y << 16); // Example packing logic
    
    // Iterate through triangle groups of 3 edges
    // This part varies significantly by table format. 
    // Using S2 style (int8_t table) for clarity in pseudocode structure:
    // int8_t config = TriangleTable[selector];
    // for (int i = 0; i < 16; i += 3) {
    //    if (config[i] == -1) break;
    //    ...
    // }
    
    // Generic placeholder for table iteration based on S1/S2 merge:
    // Retrieve edge indices e1, e2, e3 from TriangleTable[selector]
    // If valid, write to buffers.
    
    uint count = Counter.IncrementCounter();
    if (count >= MaxTriangle) return;
    
    // Write vertices and indices
    // ...
}

uint3 CubeVertex(uint index) {
    bool x = index & 1;
    bool y = index & 2;
    bool z = index & 4;
    return uint3(x ^ y, y, z); // Bit decomposition
}

uint2 EdgeVertexPair(uint index) {
    // Standard MC edge-to-vertex mapping
    uint v1 = index & 7;
    uint v2 = index < 8 ? ((index + 1) & 3) | (index & 4) : v1 + 4;
    return uint2(v1, v2);
}

float3 TransformPoint(float3 p) {
    // Map grid index to world space
    return (p + 0.5 - Dims / 2.0) * Scale;
}
```

## Subtopics

### CPU Marching Cubes Fallback
A CPU-based fallback mechanism used in hybrid implementations (e.g., CuMCubes) when GPU processing is not available or for preprocessing [S2, unofficial]. It samples an implicit function at grid points on the CPU to create a density grid, transfers this grid to the GPU, and then invokes the GPU Marching Cubes kernel [S2, unofficial]. It iterates over the sample points using nested loops, calling the user-provided function to compute density values, and stores them in a CPU tensor before transferring to CUDA memory [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_x N_y N_z)$, as each voxel is processed by one thread [S4, unofficial].
*   **Memory Complexity:** $O(N_x N_y N_z)$ for the input volume and output buffers [S4, unofficial]. The output buffer size is fixed at $(N_x-1)(N_y-1)(N_z-1) \times 15$ vertices to handle the worst case [S4, unofficial].
*   **Performance:** Highly parallel and efficient on modern GPUs due to the independence of voxel processing [S1, S2, unofficial]. Performance is limited by memory bandwidth when sampling the scalar field and writing to the output buffers [S1, unofficial].
*   **Strengths:** Deterministic memory usage (fixed buffer size), GPU-friendly parallelism, supports dynamic scenes and implicit functions [S1, S2, unofficial].
*   **Limitations:** Fixed buffer size may lead to unused space (cleared by cleanup pass) or overflow if `MaxTriangle` is underestimated [S1, unofficial]. Topological ambiguities (e.g., Type A holes) can cause discontinuities or holes in the mesh [S7]. Vertex interpolation using fixed midpoints (in some variants) assumes binary fields or exact midpoint intersections [S4, unofficial].
*   **When to Use:** Real-time isosurface extraction, large volumetric datasets, implicit function visualization [S1, S2, S4, unofficial].
*   **When Not to Use:** Scenes requiring exact manifold connectivity without post-processing (due to ambiguities) or when memory is extremely constrained and the volume is sparse [S7][S4, unofficial].

## Background
The Marching Cubes algorithm was published in 1987 by Lorensen and Cline for extracting a polygonal mesh of an isosurface from a three-dimensional discrete scalar field [S6, unofficial]. It solves the problem of visualizing constant density surfaces by dividing the volume into cubic cells and approximating the surface within each cell using a lookup table based on the 256 possible binary configurations of a cube's 8 vertices [S6, S115, unofficial]. The GPU acceleration variants, such as Compute Shader Marching Cubes, leverage the parallel architecture of GPUs to process these cells simultaneously, significantly speeding up the reconstruction process compared to CPU-based implementations [S1, S2, unofficial]. The lookup tables used are often sourced from Paul Bourke's 'Polygonise' geometry reference [S2, S1, unofficial].

## Sources
[S1] keijiro/ComputeMarchingCubes, https://github.com/keijiro/ComputeMarchingCubes, fetched 2026-10-26 (unofficial)
[S2] lzhnb/CuMCubes, https://github.com/lzhnb/CuMCubes, fetched 2026-10-26 (unofficial)
[S3] mwalczyk/marching-cubes, https://github.com/mwalczyk/marching-cubes, fetched 2026-10-26 (unofficial)
[S4] ForeverDavid/marching-cubes-cuda, https://github.com/ForeverDavid/marching-cubes-cuda, fetched 2026-10-26 (unofficial)
[S5] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S6] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S7] Local hull-based surface construction of volumetric data from silhouettes, https://doi.org/10.1109/tip.2008.926149, fetched 2026-10-30
