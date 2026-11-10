# GPU Marching Cubes (Compute Shader)

Also called: Marching Cubes GPU, Unity Marching Cubes Compute Shader

## Overview
This method implements the Marching Cubes isosurface extraction algorithm using a GPU compute shader, specifically designed for integration within the Unity engine environment [S1, unofficial]. It addresses the performance bottleneck of CPU-based surface extraction by parallelizing the voxel processing across the GPU, while handling the variable number of output triangles per voxel through an append buffer mechanism [S1, unofficial]. Unlike [GPU Marching Cubes with Histogram Pyramid](2317-gpu-marching-cubes-with-histogram-pyramid.md), which uses a two-pass allocation for precise memory placement, this approach relies on atomic appends, accepting potential buffer over-allocation or fragmentation in exchange for implementation simplicity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field (density texture) of resolution $N \times N \times N$, typically stored as a `Texture3D` or `RenderTexture` with single-channel float format [S1, unofficial].
*   **Input:** The isosurface threshold value (`isoLevel`), defaulting to 0.5 [S1, unofficial].
*   **Output:** A structured buffer containing the extracted mesh vertices and normals, written via an append buffer.
*   **Output:** Indirect draw arguments for rendering the resulting geometry.

**Data Structures:**
1.  **Lookup Tables:** Static constant arrays in the compute shader.
    *   `edgeTable`: 256-entry array of hex values (uint32), defining edge intersections for each of the 256 cube configurations [S1, unofficial].
    *   `triTable`: 256x16 array of integers (int32), defining up to 5 triangles (15 indices) per configuration, padded with -1 [S1, unofficial].
2.  **Append Buffer (`appendVertexBuffer`):**
    *   Type: `ComputeBufferType.Append` [S1, unofficial].
    *   Capacity: $(N-1)^3 \times 5$ elements, where each element is 18 bytes (3 vertices $\times$ 6 floats per vertex) [S1, unofficial].
    *   Layout: Structured buffer of `Triangle` structs. Each `Triangle` contains 3 `Vertex` structs. Each `Vertex` contains `float3 vPosition` and `float3 vNormal` [S1, unofficial].
3.  **Argument Buffer (`argBuffer`):**
    *   Type: `ComputeBufferType.IndirectArguments` [S1, unofficial].
    *   Layout: 4 integers (16 bytes) storing draw call arguments: vertex count, instance count, start vertex/index, start instance [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Find the compute shader kernel named "MarchingCubes" using `FindKernel("MarchingCubes")` [S1, unofficial].
    *   Allocate `appendVertexBuffer` with capacity $(N-1)^3 \times 5$ elements of 18 bytes each [S1, unofficial].
    *   Allocate `argBuffer` with 4 integers [S1, unofficial].
    *   Set shader parameters: `_gridSize` to $N$, `_isoLevel` to the threshold value [S1, unofficial].
    *   Bind the `appendVertexBuffer` to the "triangleRW" buffer in the kernel [S1, unofficial].

2.  **Reset State:**
    *   Reset the append buffer counter to 0 using `SetCounterValue(0)` before each frame to clear previous output [S1, unofficial].

3.  **Dispatch Compute Shader:**
    *   Dispatch the kernel with thread groups sized $(N/8, N/8, N/8)$, implying an 8x8x8 thread group size [S1, unofficial].
    *   Each thread processes one voxel $(i, j, k)$.
    *   **Boundary Check:** Threads must verify that all 8 corner vertices of the voxel exist within the volume bounds $[0, N-1]$ to avoid out-of-bounds sampling [S1, S2, unofficial]. If any corner is out of bounds, the thread exits [S1, S2, unofficial].
    *   **Voxel Processing:**
        *   Sample the scalar field values $v_0 \dots v_7$ at the 8 corners of the voxel [S1, unofficial].
        *   Compute the case index $C = \sum_{k=0}^{7} (v_k < \text{isoLevel} ? 2^k : 0)$ [S1, unofficial].
        *   Look up `edgeTable[C]` to determine which edges are intersected [S1, unofficial].
        *   Look up `triTable[C]` to determine the triangle indices [S1, unofficial].
        *   For each triangle defined in `triTable[C]`:
            *   Calculate the intersection points on the intersected edges using linear interpolation [S1, unofficial].
            *   Calculate the vertex normal using the gradient of the scalar field at the intersection point [S1, unofficial].
            *   Atomically append the triangle (3 vertices) to `appendVertexBuffer` [S1, unofficial].

4.  **Retrieve Output Count:**
    *   Initialize `argBuffer` with `{0, 1, 0, 0}` [S1, unofficial].
    *   Use `ComputeBuffer.CopyCount` to copy the current count of `appendVertexBuffer` to `argBuffer[0]` [S1, unofficial].
    *   Read back the count from `argBuffer` to the CPU [S1, unofficial].

5.  **Adjust Draw Arguments:**
    *   Multiply the vertex count (stored in `argBuffer[0]`) by 3 to get the total index count for triangles (assuming 3 indices per triangle) [S1, unofficial].
    *   Update `argBuffer` with the new vertex count [S1, unofficial].

6.  **Rendering:**
    *   In the render pass, set the material pass [S1, unofficial].
    *   Bind `appendVertexBuffer` to the "triangles" buffer in the shader [S1, unofficial].
    *   Set the model matrix [S1, unofficial].
    *   Call `Graphics.DrawProceduralIndirect` with `MeshTopology.Triangles` and the `argBuffer` [S1, unofficial].

**Pseudocode (Compute Shader Kernel):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
[numthreads(8, 8, 8)]
void MarchingCubes (uint3 id : SV_DispatchThreadID)
{
    // 1. Boundary Check
    // The volume has resolution N. Voxels are indexed 0 to N-2.
    // Corners range from 0 to N-1.
    if (id.x >= _gridSize - 1 || id.y >= _gridSize - 1 || id.z >= _gridSize - 1)
        return;

    // 2. Sample Scalar Field
    // Assume _densityTexture is a Texture3D
    float3 p0 = float3(id.x, id.y, id.z);
    float3 p1 = float3(id.x + 1, id.y, id.z);
    float3 p2 = float3(id.x, id.y + 1, id.z);
    float3 p3 = float3(id.x + 1, id.y + 1, id.z);
    float3 p4 = float3(id.x, id.y, id.z + 1);
    float3 p5 = float3(id.x + 1, id.y, id.z + 1);
    float3 p6 = float3(id.x, id.y + 1, id.z + 1);
    float3 p7 = float3(id.x + 1, id.y + 1, id.z + 1);

    float v0 = _densityTexture[p0];
    float v1 = _densityTexture[p1];
    float v2 = _densityTexture[p2];
    float v3 = _densityTexture[p3];
    float v4 = _densityTexture[p4];
    float v5 = _densityTexture[p5];
    float v6 = _densityTexture[p6];
    float v7 = _densityTexture[p7];

    // 3. Compute Case Index
    uint cubeIndex = 0;
    if (v0 < _isoLevel) cubeIndex |= 1;
    if (v1 < _isoLevel) cubeIndex |= 2;
    if (v2 < _isoLevel) cubeIndex |= 4;
    if (v3 < _isoLevel) cubeIndex |= 8;
    if (v4 < _isoLevel) cubeIndex |= 16;
    if (v5 < _isoLevel) cubeIndex |= 32;
    if (v6 < _isoLevel) cubeIndex |= 64;
    if (v7 < _isoLevel) cubeIndex |= 128;

    // 4. Lookup Edges and Triangles
    uint edgeMask = edgeTable[cubeIndex];
    int* triIndices = triTable[cubeIndex];

    // 5. Generate Triangles
    // triIndices contains up to 15 integers (5 triangles * 3 indices)
    // Padded with -1
    for (int i = 0; i < 15; i += 3)
    {
        int idx0 = triIndices[i];
        int idx1 = triIndices[i + 1];
        int idx2 = triIndices[i + 2];

        if (idx0 == -1) break; // End of triangles for this case

        // Calculate intersection points for edges
        // Edge indices correspond to the 12 edges of the cube
        // This requires mapping edge indices to corner pairs and interpolating
        // For brevity, assume helper function GetIntersection(edgeIndex, vA, vB, pA, pB, isoLevel)
        
        float3 pos0 = GetIntersection(idx0, v0, v1, p0, p1, _isoLevel); // Example mapping
        float3 pos1 = GetIntersection(idx1, v0, v2, p0, p2, _isoLevel);
        float3 pos2 = GetIntersection(idx2, v0, v4, p0, p4, _isoLevel);
        
        // Calculate normals (gradient approximation)
        float3 norm0 = CalculateGradient(pos0, v0, ...);
        float3 norm1 = CalculateGradient(pos1, v1, ...);
        float3 norm2 = CalculateGradient(pos2, v2, ...);

        // Append to buffer
        Triangle tri;
        tri.v[0] = Vertex(pos0, norm0);
        tri.v[1] = Vertex(pos1, norm1);
        tri.v[2] = Vertex(pos2, norm2);
        
        triangleRW.Append(tri);
    }
}
```

**Parameters:**
*   `Resolution` ($N$): The grid resolution for the marching cubes volume. Used to calculate buffer sizes and dispatch threads [S1, unofficial].
*   `isoLevel`: The scalar threshold value for isosurface extraction. Default value is 0.5f [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Hardware Support:** The implementation assumes DX11 hardware. Mobile support is uncertain and likely unsupported [S1, unofficial].
*   **Density Field Generation:** The example generates the density field on the CPU, which may be a bottleneck. For real applications, `Texture3D` should be replaced with `RenderTexture` [S1, unofficial].
*   **Buffer Overflow:** The append buffer is pre-allocated with a maximum capacity. If the scene complexity exceeds this, triangles will be dropped. The capacity is $(N-1)^3 \times 5$ triangles [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N^3)$ for the compute shader dispatch, where $N$ is the resolution. Each voxel is processed in constant time relative to the grid size.
*   **Memory Complexity:** $O(N^3)$ for the input texture and $O(N^3)$ for the output buffer (worst-case allocation) [S1, unofficial].
*   **Performance:** Performance is dependent on GPU compute capabilities and DX11 support. The use of append buffers avoids the overhead of a histogram pyramid but may lead to buffer management issues if the maximum triangle count is not accurately estimated [S1, unofficial].
*   **Strengths:** Simple implementation, leverages standard Unity compute shaders, no need for complex two-pass allocation logic [S1, unofficial].
*   **Limitations:** Fixed maximum buffer size, potential for dropped triangles if capacity is exceeded, CPU-bound density generation in the example [S1, unofficial].
*   **When to use:** Suitable for prototyping or simple scenes within Unity where DX11 hardware is guaranteed and scene complexity is predictable [S1, unofficial].
*   **When not to use:** High-performance production environments requiring dynamic scene complexity, mobile platforms, or when precise memory control is required [S1, unofficial].

## Background
This implementation was developed for Unity 5.6, with potential compatibility with older versions [S1, unofficial]. It is based on the Marching Cubes algorithm and lookup tables by Paul Bourke [S1, unofficial]. The gradient normals calculation is based on work from "MCImproved" [S1, unofficial]. The specific Unity implementation is credited to Pavel Kouril and Vilém Otte [S1, unofficial].

## Sources
[S1] pavelkouril/unity-marching-cubes-gpu, https://github.com/pavelkouril/unity-marching-cubes-gpu, fetched 2026-10-26 (unofficial)
