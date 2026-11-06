# GPU Marching Cubes with Lookup Tables

Also called: Compute Shader Marching Cubes, GPU Marching Cubes with Fixed-Size Buffer, MC GPU, Marching Cubes GPU Implementation, Marching Cubes GPU Mesh Reconstruction

## Overview
A GPU-based implementation of the Marching Cubes algorithm that uses pre-computed lookup tables stored in 2D textures or structured buffers to determine edge intersections and triangle configurations for a voxel grid [S1, S2, S3, unofficial]. It generates a triangle mesh from a 3D scalar field (voxels) using compute shaders, processing each voxel independently to determine surface intersections and outputting vertex data (position and normal) into a structured buffer [S2, S3, unofficial]. Unlike CPU-based implementations, this method leverages massive parallelism to process voxels simultaneously, often outputting geometry directly to structured buffers or append buffers for subsequent rendering [S1, S2, unofficial]. It builds on [Primal Grid Marching Cubes](1211-primal-grid-marching-cubes.md) by utilizing compute shaders for the core extraction logic rather than fragment shaders or texture-based lookups alone.

## How it works
**Inputs and Outputs:**
- **Input:** A 3D scalar field (voxel grid) of dimensions $N_x \times N_y \times N_z$, typically generated via procedural noise or loaded from a texture. A target isovalue $\tau$ (default 0.0) [S2, unofficial].
- **Output:** A structured buffer containing vertex data (position and normal) for the extracted mesh. The buffer size is fixed to the maximum possible vertex count: $N_x N_y N_z \times 15$ vertices (5 triangles per voxel $\times$ 3 vertices per triangle) [S2, unofficial].

**Data Structures:**
1.  **Voxel Grid (`_Voxels`):** A 1D structured buffer or 3D texture storing scalar values. Accessed via linear index $idx = x + y \cdot N_x + z \cdot N_x \cdot N_y$ [S2, unofficial].
2.  **Normal Field (`_Normals`):** A 3D texture (e.g., `Tex3D<float3>`) storing pre-computed gradients/normals at voxel centers or vertices [S2, unofficial].
3.  **Edge Table (`_CubeEdgeFlags`):** A lookup table mapping the 8-bit vertex configuration index (0–255) to a 12-bit edge intersection mask. Each bit corresponds to one of the 12 edges of the cube [S2, unofficial]. Stored in a `ComputeBuffer<int>` of size 256 or a 2D texture `edge_table_tex` of size $256 \times 1$ with format `GL_R16I` [S1, S2, unofficial].
4.  **Triangle Connection Table (`_TriangleConnectionTable`):** A lookup table defining the triangles for each of the 256 cube configurations. Each configuration stores up to 16 vertex indices (defining up to 5 triangles). Stored in a `ComputeBuffer<int>` of size $256 \times 16$ or a 2D texture `tri_table_tex` of size $16 \times 256$ with format `GL_R16I` [S1, S2, unofficial].
5.  **Output Buffer (`_Buffer`):** A `RWStructuredBuffer<Vert>` where `Vert` contains `float4 position` (w=1.0) and `float3 normal` [S2, unofficial]. Indexed by $idx \cdot 15 + (3 \cdot i + j)$, where $idx$ is the voxel linear index, and $3 \cdot i + j$ is the vertex index within the triangle list for that voxel [S2, unofficial].
6.  **Cube Geometry Constants:**
    -   `vertexOffset[8]`: Offsets of the 8 cube corners relative to the cube origin.
    -   `edgeDirection[12]`: Direction vectors for the 12 edges.
    -   `edgeConnection[12]`: Pairs of vertex indices for each edge (e.g., `int2(0,1)` for the edge between vertex 0 and 1) [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    -   Initialize the output `_Buffer` with a marker value (e.g., position.w = -1.0) to indicate unused vertices [S2, unofficial].
    -   Set up lookup tables (`_CubeEdgeFlags`, `_TriangleConnectionTable`) and ensure texture filtering is `NEAREST` and wrapping is `CLAMP_TO_EDGE` to prevent interpolation artifacts in integer lookups [S1, S2, unofficial].

2.  **Dispatch Compute Shader:**
    -   Dispatch a compute shader with thread groups of size $4 \times 4 \times 4$ or $8 \times 8 \times 8$ [S2, S3, unofficial].
    -   Each thread processes one voxel cube identified by `SV_DispatchThreadID` $(id_x, id_y, id_z)$ [S3, unofficial].
    -   The grid dimension $N$ must be divisible by the thread group size [S2, unofficial].

3.  **Boundary Check:**
    -   If any coordinate of $(id + 1)$ is greater than or equal to the grid dimensions $(N_x, N_y, N_z)$, or if the voxel is within the `_Border` distance of the grid edge, return immediately to avoid out-of-bounds access [S2, S3, unofficial].

4.  **Sample Voxel Data:**
    -   Sample the scalar values and gradients at the 8 vertices of the current cube.
    -   Store them in an array `samples[8]`, where `samples[k]` contains the gradient (xyz) and scalar value (w) [S3, unofficial].
    -   Clamp indices to $[1, N-1]$ if necessary to handle boundaries [S3, unofficial].

5.  **Compute Cube Configuration Selector:**
    -   Initialize an integer `selector` to 0.
    -   For each vertex $k \in [0, 7]$, if `samples[k].w < Isovalue`, set the $k$-th bit: `selector |= (1 << k)` [S3, unofficial].

6.  **Handle Empty/Fully-Filled Cubes:**
    -   If `selector == 0` or `selector == 0xff` (255), return immediately as there is no intersection with the isosurface [S3, unofficial].

7.  **Compute Edge Intersections:**
    -   Retrieve the edge mask `edgeFlags` from `_CubeEdgeFlags[selector]` [S2, unofficial].
    -   For each edge index $i \in [0, 11]$:
        -   If the $i$-th bit is set in `edgeFlags` (`(edgeFlags & (1 << i)) != 0`):
            -   Get the two vertex indices $v_1, v_2$ from `edgeConnection[i]` [S2, unofficial].
            -   Get the scalar values $val_1 = samples[v_1].w$ and $val_2 = samples[v_2].w$ [S2, unofficial].
            -   Compute the interpolation parameter:
                $$
                offset = \begin{cases} 
                \frac{\tau - val_1}{val_2 - val_1} & \text{if } val_2 \neq val_1 \\
                0.5 & \text{otherwise}
                \end{cases}
                $$
                [S2, unofficial]
            -   Calculate the intersection position in local cube space:
                $$
                pos_{intersect} = pos_{v_1} + offset \cdot (pos_{v_2} - pos_{v_1})
                $$
                [S2, unofficial]
            -   Transform to world space and compute the normal by interpolating the gradients or sampling from `_Normals` [S3, unofficial].
            -   Store the result in `edgeVertex[i]` [S2, unofficial].

8.  **Emit Triangles:**
    -   Retrieve the triangle configuration data from `_TriangleConnectionTable[selector * 16 + 3 * i]` for $i = 0, 1, \dots, 4$ [S2, unofficial].
    -   For each triangle $i$:
        -   If the first edge index is 15 (or -1, depending on table encoding), stop processing this cube [S3, unofficial].
        -   Extract the three vertex indices $idx_1, idx_2, idx_3$ from the table [S2, unofficial].
        -   For each vertex index, retrieve the corresponding `edgeVertex` position and normal [S2, unofficial].
        -   Create a `Vert` struct with the world-space position and normal [S2, unofficial].
        -   Write the vertex to `_Buffer[idx * 15 + (3 * i + j)]` where $j \in \{0, 1, 2\}$ [S2, unofficial].

9.  **Rendering:**
    -   Render the mesh using `Graphics.DrawProceduralNow` with `MeshTopology.Triangles` and the total buffer size [S2, unofficial].
    -   The vertex shader runs for each vertex in the buffer. Triangles with zero area (unused vertices) generate no fragments [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Main Function
[numthreads(8, 8, 8)]
void CSMain(uint3 id : SV_DispatchThreadID) {
    int x = id.x;
    int y = id.y;
    int z = id.z;
    
    // Boundary check
    if (x >= Width - 1 - Border || y >= Height - 1 - Border || z >= Depth - 1 - Border) {
        return;
    }

    // 1. Sample 8 vertices of the cube
    float4 samples[8];
    for (int k = 0; k < 8; k++) {
        int3 vPos = GetVertexPosition(x, y, z, k); // Helper to get grid coords of corner k
        samples[k] = VoxelValueWithGradient(vPos); // Returns float4(gradient, scalar)
    }

    // 2. Compute selector
    int selector = 0;
    for (int k = 0; k < 8; k++) {
        if (samples[k].w < Isovalue) {
            selector |= (1 << k);
        }
    }

    // 3. Early exit for empty/full cubes
    if (selector == 0 || selector == 255) {
        return;
    }

    // 4. Get edge flags
    int edgeFlags = CubeEdgeFlags[selector];

    // 5. Compute edge intersections
    float3 edgeVertex[12];
    float3 edgeNormal[12];
    
    for (int i = 0; i < 12; i++) {
        if ((edgeFlags & (1 << i)) != 0) {
            int2 vPair = EdgeConnection[i];
            float v1 = samples[vPair.x].w;
            float v2 = samples[vPair.y].w;
            
            float offset = 0.5f;
            float delta = v2 - v1;
            if (delta != 0.0f) {
                offset = (Isovalue - v1) / delta;
            }
            
            // Linear interpolation of position
            float3 p1 = GetWorldPosition(x, y, z, vPair.x);
            float3 p2 = GetWorldPosition(x, y, z, vPair.y);
            edgeVertex[i] = p1 + offset * (p2 - p1);
            
            // Interpolate normal or sample from normal texture
            edgeNormal[i] = Normalize(samples[vPair.x].xyz + offset * (samples[vPair.y].xyz - samples[vPair.x].xyz));
        }
    }

    // 6. Emit triangles
    int baseIdx = (x + y * Width + z * Width * Height) * 15;
    
    for (int i = 0; i < 5; i++) {
        int triOffset = selector * 16 + 3 * i;
        int e1 = TriangleConnectionTable[triOffset];
        if (e1 == 15) break; // End of triangle list
        
        int e2 = TriangleConnectionTable[triOffset + 1];
        int e3 = TriangleConnectionTable[triOffset + 2];
        
        // Emit 3 vertices
        for (int j = 0; j < 3; j++) {
            int edgeIdx = (j == 0) ? e1 : ((j == 1) ? e2 : e3);
            int vertIdx = baseIdx + 3 * i + j;
            
            Vert v;
            v.position = float4(edgeVertex[edgeIdx], 1.0f);
            v.normal = edgeNormal[edgeIdx];
            
            // Adjust position relative to center if required by rendering pipeline
            // v.position.xyz -= Centre;
            
            Buffer[vertIdx] = v;
        }
    }
}

// Helper to get grid coordinates of a vertex corner k (0-7)
int3 GetVertexPosition(int x, int y, int z, int k) {
    // Mapping based on standard MC vertex ordering:
    // 0:(0,0,0), 1:(1,0,0), 2:(1,1,0), 3:(0,1,0)
    // 4:(0,0,1), 5:(1,0,1), 6:(1,1,1), 7:(0,1,1)
    int dx = (k & 1);
    int dy = (k & 2) >> 1;
    int dz = (k & 4) >> 2;
    return int3(x + dx, y + dy, z + dz);
}
```

## Subtopics

### [Marching Cubes on GPU](3767-marching-cubes-on-gpu.md)
A technique to extract an isosurface from a 3D scalar field by executing the Marching Cubes algorithm within a GPU compute shader. It generates triangle geometry for each voxel where the isosurface passes, using a lookup table for topology and linear interpolation for vertex positions. Unlike CPU-based implementations, this method leverages massive parallelism to process voxels simultaneously, often outputting geometry directly to structured buffers or append buffers for subsequent rendering. It builds on [GPU Marching Cubes with Lookup Tables](3690-gpu-marching-cubes-with-lookup-tables.md) by utilizing compute shaders for the core extraction logic rather than fragment shaders or texture-based lookups alone.

## Operating characteristics
-   **Time Complexity:** $O(N^3)$, where $N$ is the grid dimension. Each voxel is processed independently in parallel.
-   **Memory Complexity:** $O(N^3)$ for the voxel grid and output buffer. The output buffer is fixed at $15 \cdot N^3$ vertices, which is an upper bound [S2, unofficial].
-   **Performance:** Rendering performance is comparable to converting the buffer to a normal mesh, as the vertex shader runs for all vertices but generates no fragments for unused ones [S2, unofficial]. For complex scenes, an append buffer or counter buffer might be more efficient to avoid processing zero-area triangles [S2, unofficial].
-   **Strengths:** Massive parallelism, no dynamic allocation during extraction, simple rendering pipeline integration via `DrawProcedural` [S2, unofficial].
-   **Limitations:** Fixed-size buffer leads to memory waste if the surface area is small. Boundary handling requires explicit checks or padding [S2, unofficial]. Precision issues may arise if `delta` is zero, requiring a fallback to 0.5 [S2, unofficial].

## Background
The method is a GPU adaptation of the classic Marching Cubes algorithm [S1, S2, S3, unofficial]. It was popularized in the context of real-time procedural generation, such as in the "Marching-Cubes-On-The-GPU" project by Scrawk, which demonstrated generating voxels and mesh entirely on the GPU using compute shaders and Unity's `DrawProcedural` [S2, unofficial]. Variations exist in academic and open-source repositories, often optimizing for specific hardware features like texture lookups for lookup tables [S1, unofficial] or structured buffers for direct memory access [S3, unofficial].

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] Scrawk/Marching-Cubes-On-The-GPU, https://github.com/Scrawk/Marching-Cubes-On-The-GPU, fetched 2026-10-26 (unofficial)
[S3] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
