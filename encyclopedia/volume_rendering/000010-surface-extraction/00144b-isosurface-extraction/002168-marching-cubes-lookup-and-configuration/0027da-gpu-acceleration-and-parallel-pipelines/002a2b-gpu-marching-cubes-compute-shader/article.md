# GPU Marching Cubes (Compute Shader)
Also called: Marching Cubes GPU, Unity Marching Cubes Compute Shader

## Overview
This method extracts an isosurface from a 3D scalar field (density texture) by executing the Marching Cubes algorithm on the GPU using a compute shader [S1, unofficial]. It solves the performance bottleneck of CPU-based surface extraction by parallelizing voxel processing across thread groups [S1, unofficial]. Unlike CPU immediate mode implementations, it utilizes an append buffer for dynamic triangle generation and indirect draw calls to handle variable output sizes per frame [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field (`Texture3D`) of resolution $R \times R \times R$, an isosurface threshold value $I$ (default 0.5), and the precomputed Marching Cubes lookup tables (`edgeTable`, `triangleTable`) [S1, unofficial].
*   **Output:** A list of triangles (vertices and normals) stored in an append buffer, rendered via `Graphics.DrawProceduralIndirect` [S1, unofficial].

**Data Structures:**
1.  **`appendVertexBuffer`**: A `ComputeBuffer` of type `Append`.
    *   Capacity: $(R-1)^3 \times 5$ elements [S1, unofficial].
    *   Element size: 18 bytes (3 vertices $\times$ 6 floats per vertex) [S1, unofficial].
    *   Layout: Each element is a `Triangle` struct containing 3 `Vertex` structs. Each `Vertex` contains `float3 vPosition` and `float3 vNormal` [S1, unofficial].
2.  **`argBuffer`**: A `ComputeBuffer` of type `IndirectArguments`.
    *   Size: 4 integers (16 bytes) [S1, unofficial].
    *   Layout: `[vertexCount, instanceCount, startVertex, startInstance]` [S1, unofficial].
3.  **Lookup Tables**: Static `const` arrays in the shader. `edgeTable` is 256 `uint`s; `triangleTable` is $256 \times 16$ `int`s [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization**:
    *   Find the compute shader kernel: `kernelID = shader.FindKernel("MarchingCubes")` [S1, unofficial].
    *   Allocate `appendVertexBuffer` with capacity $(R-1)^3 \times 5$ elements of 18 bytes each [S1, unofficial].
    *   Allocate `argBuffer` with 4 integers [S1, unofficial].
    *   Set shader parameters: `_gridSize` = $R$, `_isoLevel` = $I$ [S1, unofficial].
    *   Bind `appendVertexBuffer` to the shader's `triangleRW` buffer [S1, unofficial].

2.  **Reset Output**:
    *   Call `appendVertexBuffer.SetCounterValue(0)` to clear the append buffer counter before dispatch [S1, unofficial].

3.  **Dispatch Compute Shader**:
    *   Calculate thread group dimensions: $(R/8, R/8, R/8)$ [S1, unofficial].
    *   Dispatch the kernel: `shader.Dispatch(kernelID, R/8, R/8, R/8)` [S1, unofficial].
    *   *Note*: This assumes $R$ is divisible by 8. Each thread group processes a block of voxels.

4.  **Compute Shader Kernel Execution (Per Voxel)**:
    *   **Thread ID to Voxel Mapping**: Each thread corresponds to a voxel $(x, y, z)$ in the $(R-1) \times (R-1) \times (R-1)$ grid of cubes [S1, unofficial].
    *   **Sample Vertices**: Read scalar values $V_i$ for the 8 vertices of the current cube from the `Texture3D` [S1, unofficial].
    *   **Case Index**: Compute the 8-bit case index $C$ by comparing $V_i$ against $I$. If $V_i < I$, set bit $i$ to 1 [S1, unofficial].
    *   **Lookup**:
        *   Retrieve edge intersection mask from `edgeTable[C]` [S1, unofficial].
        *   Retrieve triangle indices from `triangleTable[C]` [S1, unofficial].
    *   **Edge Interpolation**: For each intersected edge, calculate the intersection point $P$ and normal $N$.
        *   Linear interpolation for position: $P = V_a + \frac{I - V_a}{V_b - V_a}(V_b - V_a)$, where $V_a, V_b$ are vertex values [S1, unofficial].
        *   Normal calculation: Use gradient-based normals (e.g., central differences) or vertex normals interpolated to $P$ [S1, unofficial].
    *   **Append Triangles**: For each triangle triplet $(e_a, e_b, e_c)$ in the case configuration:
        *   Create 3 vertices from the interpolated edge points.
        *   Append the `Triangle` struct to `triangleRW` (the append buffer) [S1, unofficial].

5.  **Retrieve Vertex Count**:
    *   Initialize `argBuffer` with `{0, 1, 0, 0}` [S1, unofficial].
    *   Copy the append buffer count to `argBuffer`: `ComputeBuffer.CopyCount(appendVertexBuffer, argBuffer, 0)` [S1, unofficial].
    *   Read back the count from `argBuffer` to the CPU [S1, unofficial].

6.  **Adjust Draw Arguments**:
    *   Let $N$ be the vertex count read from `argBuffer`.
    *   Calculate total indices: $N_{indices} = N \times 3$ [S1, unofficial].
    *   Update `argBuffer` with `{N_{indices}, 1, 0, 0}` [S1, unofficial].

7.  **Render**:
    *   Bind `appendVertexBuffer` to the material's `triangles` buffer [S1, unofficial].
    *   Set material properties (model matrix, etc.) [S1, unofficial].
    *   Call `Graphics.DrawProceduralIndirect(MeshTopology.Triangles, argBuffer)` [S1, unofficial].

**Pseudocode (Compute Shader Kernel Logic):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the compute shader kernel "MarchingCubes"
# Assumes standard Marching Cubes lookup tables are available as static constants

[numthreads(8, 8, 8)]
void MarchingCubes (uint3 id : SV_DispatchThreadID)
{
    // 1. Determine voxel coordinates
    uint x = id.x;
    uint y = id.y;
    uint z = id.z;

    // 2. Check bounds (grid is R-1 cubes for R vertices)
    if (x >= gridSize - 1 || y >= gridSize - 1 || z >= gridSize - 1)
        return;

    // 3. Sample the 8 vertices of the current cube
    // Vertex ordering must match the standard MC convention
    float v0 = tex3D(_densityTexture, float3(x, y, z)).r;
    float v1 = tex3D(_densityTexture, float3(x + 1, y, z)).r;
    float v2 = tex3D(_densityTexture, float3(x + 1, y + 1, z)).r;
    float v3 = tex3D(_densityTexture, float3(x, y + 1, z)).r;
    float v4 = tex3D(_densityTexture, float3(x, y, z + 1)).r;
    float v5 = tex3D(_densityTexture, float3(x + 1, y, z + 1)).r;
    float v6 = tex3D(_densityTexture, float3(x + 1, y + 1, z + 1)).r;
    float v7 = tex3D(_densityTexture, float3(x, y + 1, z + 1)).r;

    // 4. Calculate case index C
    uint cubeIndex = 0;
    if (v0 < isoLevel) cubeIndex |= 1;
    if (v1 < isoLevel) cubeIndex |= 2;
    if (v2 < isoLevel) cubeIndex |= 4;
    if (v3 < isoLevel) cubeIndex |= 8;
    if (v4 < isoLevel) cubeIndex |= 16;
    if (v5 < isoLevel) cubeIndex |= 32;
    if (v6 < isoLevel) cubeIndex |= 64;
    if (v7 < isoLevel) cubeIndex |= 128;

    // 5. Handle ambiguous cases (46, 47, 98, 99, 210, 213, 214, 215)
    // Standard LUT uses fixed triangulation; implementation may need specific handling
    // [G] For simplicity, we assume the standard LUT handles this or the user accepts potential cracks.

    // 6. Get edge intersections
    // edgeTable[cubeIndex] gives the bitmask of intersected edges
    uint edgeMask = edgeTable[cubeIndex];
    
    // If no edges intersect, skip
    if (edgeMask == 0)
        return;

    // 7. Intersect edges and store points
    // Edge indices 0-11 correspond to the standard MC edge numbering
    float3 points[12];
    for (int i = 0; i < 12; i++) {
        if ((edgeMask & (1 << i)) != 0) {
            points[i] = IntersectEdge(i, v0, v1, v2, v3, v4, v5, v6, v7, x, y, z);
        }
    }

    // 8. Generate triangles from triangleTable
    // triangleTable[cubeIndex] contains triplets of edge indices, terminated by -1
    int* triIndices = triangleTable[cubeIndex];
    int idx = 0;
    while (triIndices[idx] != -1) {
        int e1 = triIndices[idx];
        int e2 = triIndices[idx + 1];
        int e3 = triIndices[idx + 2];
        
        // Create triangle from the 3 intersection points
        Triangle tri;
        tri.v0 = points[e1];
        tri.v1 = points[e2];
        tri.v2 = points[e3];
        
        // Calculate normal (e.g., cross product of two edges)
        float3 edgeA = tri.v1 - tri.v0;
        float3 edgeB = tri.v2 - tri.v0;
        tri.n = normalize(cross(edgeA, edgeB));
        
        // Append to buffer
        triangleRW.Append(tri);
        
        idx += 3;
    }
}

// Helper to interpolate intersection point on an edge
float3 IntersectEdge(int edgeIndex, float v0, float v1, float v2, float v3, float v4, float v5, float v6, float v7, uint x, uint y, uint z) {
    float3 p1, p2;
    float val1, val2;
    
    // Map edge index to vertices
    switch (edgeIndex) {
        case 0: p1 = float3(x, y, z); p2 = float3(x+1, y, z); val1 = v0; val2 = v1; break;
        case 1: p1 = float3(x+1, y, z); p2 = float3(x+1, y+1, z); val1 = v1; val2 = v2; break;
        case 2: p1 = float3(x+1, y+1, z); p2 = float3(x, y+1, z); val1 = v2; val2 = v3; break;
        case 3: p1 = float3(x, y+1, z); p2 = float3(x, y, z); val1 = v3; val2 = v0; break;
        case 4: p1 = float3(x, y, z+1); p2 = float3(x+1, y, z+1); val1 = v4; val2 = v5; break;
        case 5: p1 = float3(x+1, y, z+1); p2 = float3(x+1, y+1, z+1); val1 = v5; val2 = v6; break;
        case 6: p1 = float3(x+1, y+1, z+1); p2 = float3(x, y+1, z+1); val1 = v6; val2 = v7; break;
        case 7: p1 = float3(x, y+1, z+1); p2 = float3(x, y, z+1); val1 = v7; val2 = v4; break;
        case 8: p1 = float3(x, y, z); p2 = float3(x, y, z+1); val1 = v0; val2 = v4; break;
        case 9: p1 = float3(x+1, y, z); p2 = float3(x+1, y, z+1); val1 = v1; val2 = v5; break;
        case 10: p1 = float3(x+1, y+1, z); p2 = float3(x+1, y+1, z+1); val1 = v2; val2 = v6; break;
        case 11: p1 = float3(x, y+1, z); p2 = float3(x, y+1, z+1); val1 = v3; val2 = v7; break;
        default: return float3(0,0,0);
    }
    
    // Linear interpolation
    float t = (isoLevel - val1) / (val2 - val1);
    return lerp(p1, p2, t);
}
```

**Parameters:**
*   `Resolution` ($R$): Grid resolution. Must be divisible by 8 for the dispatch dimensions $(R/8, R/8, R/8)$ to be integer-aligned [S1, unofficial].
*   `isoLevel`: Scalar threshold. Default 0.5 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Hardware Dependency**: Requires DX11 hardware. Mobile support is uncertain [S1, unofficial].
*   **Density Field Generation**: The example generates the density field on the CPU, which is a bottleneck. For real applications, use a `RenderTexture` instead of `Texture3D` [S1, unofficial].
*   **Ambiguity**: Cases 46, 47, 98, 99, 210, 213, 214, 215 contain saddle points. The standard LUT uses fixed triangulation, which can cause cracks if adjacent voxels resolve ambiguities differently [S1, unofficial].
*   **Lighting**: The default shader uses a barebone diffuse lighting model. Custom shaders are recommended for better visuals [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels $(R-1)^3$. Each voxel is processed in parallel by a thread group.
*   **Memory Complexity**: $O(N)$ for the append buffer, with capacity $(R-1)^3 \times 5$ triangles [S1, unofficial].
*   **Performance**: Parallelizes the extraction process, significantly faster than CPU-based methods for high resolutions.
*   **Strengths**: Dynamic output size handled via append buffers and indirect draw calls; leverages GPU parallelism.
*   **Limitations**: Requires DX11; CPU density generation is a bottleneck in the reference implementation; potential topological artifacts in ambiguous cases.

## Background
This implementation is an unofficial GPU-based Marching Cubes algorithm for Unity, developed by Pavel Kouril [S1, unofficial]. It builds on the algorithm and lookup tables by Paul Bourke and uses gradient normals calculation methods from angelfire.com [S1, unofficial]. It was developed with Unity 5.6 and requires DX11 hardware [S1, unofficial].

## Sources
[S1] pavelkouril/unity-marching-cubes-gpu, https://github.com/pavelkouril/unity-marching-cubes-gpu, fetched 2026-10-26 (unofficial)
