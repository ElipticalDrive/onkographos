# Marching Cubes on GPU

Also called: Compute Shader Marching Cubes, MarchingCubesOnGPU

## Overview
Marching Cubes on GPU is a technique to extract an isosurface from a 3D scalar field (density texture) by executing the Marching Cubes algorithm within a GPU compute shader [S1, S2, unofficial]. It generates triangle geometry for each voxel where the isosurface passes, using a lookup table for topology and linear interpolation for vertex positions [S1, unofficial]. Unlike CPU-based implementations, this method leverages massive parallelism to process voxels simultaneously, often outputting geometry directly to structured buffers or append buffers for subsequent rendering [S1, S2, unofficial]. It builds on [GPU Marching Cubes with Lookup Tables](3690-gpu-marching-cubes-with-lookup-tables.md) by utilizing compute shaders for the core extraction logic rather than fragment shaders or texture-based lookups alone.

## How it works
The method processes a 3D voxel grid in parallel using compute shader thread groups. Each thread or group of threads is responsible for one or more voxels, determining edge intersections, and generating triangles.

### Inputs and Outputs
**Inputs:**
*   `_densityTexture` (Texture3D): 3D texture storing scalar field values [S1, unofficial].
*   `_isoLevel` (float): Scalar threshold for extraction [S1, unofficial].
*   `_gridSize` (int): Resolution $N \times N \times N$ of the volume [S1, unofficial].
*   `_Target` (float): Alternative isovalue threshold, typically $0.0f$ [S2, unofficial].
*   `_Width`, `_Height`, `_Depth` (int): Dimensions of the voxel grid, typically all equal to $N$ [S2, unofficial].
*   `_Border` (int): Number of voxels to skip at boundaries to avoid incorrect normals/neighbors, typically $1$ [S2, unofficial].
*   `_TriangleConnectionTable` (StructuredBuffer<int>): 256 entries, each containing 16 integers representing edge triples for triangulation, terminated by -1 [S2, unofficial].
*   `_CubeEdgeFlags` (StructuredBuffer<int>): 256 entries, bitmask where bit $n$ is set if edge $n$ is intersected [S2, unofficial].
*   `_Voxels` (StructuredBuffer<float>): Input volume data, size $N^3$ [S2, unofficial].
*   `_Normals` (StructuredBuffer<float3>): Input volume normals, size $N^3$ [S2, unofficial].

**Outputs:**
*   `triangleRW` (AppendStructuredBuffer<Triangle>): Output buffer for generated triangles [S1, unofficial].
*   `_Buffer` (RWStructuredBuffer<Vert>): Output buffer for vertices, max size $N^3 \times 3 \times 5$ vertices [S2, unofficial].
*   `StructuredBuffer<Triangle> triangles`: Final geometry buffer read by the vertex shader [S1, unofficial].

### Data Structures
*   **Vertex**: `{ float3 vPosition; float3 vNormal; }` [S1, unofficial].
*   **Triangle**: `{ Vertex v[3]; }` [S1, unofficial].
*   **Vert**: `{ float4 position; float3 normal; }` [S2, unofficial]. Position is stored as `float4(position - centre, 1.0)` where `centre = float3(_Width, 0, _Depth) / 2.0` [S2, unofficial].

### Algorithm Steps
1.  **Initialization**: Dispatch compute shader with thread group size `[numthreads(8, 8, 8)]` [S1, S2, unofficial]. Calculate number of groups as $N/8$ in each dimension [S2, unofficial].
2.  **Thread Identification**: Each thread receives `threadId` (or `id`) representing the voxel index [S1, S2, unofficial].
3.  **Boundary Check**: If `id.x >= _Width - 1 - _Border`, `id.y >= _Height - 1 - _Border`, or `id.z >= _Depth - 1 - _Border`, return immediately to avoid generating vertices at edges without neighbors [S2, unofficial].
4.  **Voxel Corner Sampling**:
    *   Calculate positions for the 8 corners of the current voxel. Coordinates are normalized to $[0,1]$ by dividing `(threadId + corner_offset)` by `(_gridSize - 1)`, then shifted by `offset = float3(-0.5, -0.5, -0.5)` to center the grid [S1, unofficial].
    *   Sample scalar field values at these 8 corners. In [S1, unofficial], `val[i]` is sampled via `_densityTexture.Load(int4(pos, 0)).x`. In [S2, unofficial], values are retrieved from `_Voxels` using linear indexing: `cube[i] = _Voxels[x + y * _Width + z * _Width * _Height]` with appropriate offsets for each corner [S2, unofficial].
5.  **Cube Index Calculation**:
    *   Compute `cubeIndex` (0-255) by checking if each corner value is below `_isoLevel` (or `_Target`) [S1, S2, unofficial].
    *   Bit `i` is set if `val[i] < _isoLevel` (or `cube[i] <= _Target`) [S1, S2, unofficial].
6.  **Edge Intersection Check**:
    *   Check if `edgeTable[cubeIndex] != 0` (or if bits are set in `_CubeEdgeFlags`) [S1, S2, unofficial]. If zero, no surface passes through this voxel; skip processing [S1, unofficial].
    *   For each of the 12 edges, if the corresponding bit is set, calculate the intersection vertex [S1, S2, unofficial].
    *   **Vertex Interpolation**: Calculate the intersection position using linear interpolation:
        $$ \text{VertexInterp}(p_1, p_2, val_{p1}, val_{p2}) = \text{lerp}(p_1, p_2, \frac{\text{isoLevel} - val_{p1}}{val_{p2} - val_{p1}}) $$
        [S1, S2, unofficial].
    *   Alternatively, calculate offset: `GetOffset(v1, v2) = (delta == 0.0f) ? 0.5f : (_Target - v1) / delta`, where `delta = v2 - v1` [S2, unofficial].
7.  **Triangle Generation**:
    *   Iterate through `triTable[cubeIndex]` (or `_TriangleConnectionTable`) [S1, S2, unofficial].
    *   For every triplet of indices $(i, i+1, i+2)$ until -1 is encountered, create a triangle [S1, unofficial].
    *   In [S2, unofficial], iterate $i$ from 0 to 4 (up to 5 triangles per cube). Check if `_TriangleConnectionTable[flagIndex * 16 + 3*i] >= 0`. If so, get three vertex indices $v$ [S2, unofficial].
8.  **Normal Calculation**:
    *   For each triangle vertex, calculate the normal by calling `CalculateGradient` at the vertex position and normalizing it [S1, unofficial].
    *   **Gradient Calculation**: Central difference vector:
        $$ \text{CalculateGradient}(p) = \text{float3}( \dots ) $$
        where components are differences of `SampleLinear` at $p \pm \text{ratio}$ along each axis, and `ratio = 1.0 / (_gridSize - 1)` [S1, unofficial].
    *   Note: When sampling for gradient, an offset of `float3(0.5, 0.5, 0.5)` is added to coordinate $p$ to align with texture space $[0,1]$ [S1, unofficial].
    *   In [S2, unofficial], normals are sampled from `_Normals` buffer using bilinear/trilinear interpolation (`SampleBilinear`) [S2, unofficial].
9.  **Output Storage**:
    *   In [S1, unofficial], append the generated triangle to `triangleRW` append buffer [S1, unofficial].
    *   In [S2, unofficial], store vertex data in `_Buffer` at index `idx = id.x + id.y * _Width + id.z * _Width * _Height` [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Entry Point
[numthreads(8, 8, 8)]
void MarchingCubes(uint3 threadId : SV_DispatchThreadID)
{
    // 1. Boundary Check
    if (threadId.x >= _Width - 1 - _Border || 
        threadId.y >= _Height - 1 - _Border || 
        threadId.z >= _Depth - 1 - _Border)
    {
        return;
    }

    // 2. Define Voxel Corners
    float3 offset = float3(-0.5, -0.5, -0.5);
    float3 p[8];
    p[0] = (threadId + float3(0, 0, 1)) / (_gridSize - 1) + offset;
    p[1] = (threadId + float3(1, 0, 1)) / (_gridSize - 1) + offset;
    p[2] = (threadId + float3(1, 0, 0)) / (_gridSize - 1) + offset;
    p[3] = (threadId + float3(0, 0, 0)) / (_gridSize - 1) + offset;
    p[4] = (threadId + float3(0, 1, 1)) / (_gridSize - 1) + offset;
    p[5] = (threadId + float3(1, 1, 1)) / (_gridSize - 1) + offset;
    p[6] = (threadId + float3(1, 1, 0)) / (_gridSize - 1) + offset;
    p[7] = (threadId + float3(0, 1, 0)) / (_gridSize - 1) + offset;

    // 3. Sample Scalar Values
    float val[8];
    val[0] = SampleData(threadId + int3(0, 0, 1));
    val[1] = SampleData(threadId + int3(1, 0, 1));
    val[2] = SampleData(threadId + int3(1, 0, 0));
    val[3] = SampleData(threadId + int3(0, 0, 0));
    val[4] = SampleData(threadId + int3(0, 1, 1));
    val[5] = SampleData(threadId + int3(1, 1, 1));
    val[6] = SampleData(threadId + int3(1, 1, 0));
    val[7] = SampleData(threadId + int3(0, 1, 0));

    // 4. Compute Cube Index
    int cubeIndex = 0;
    if (val[0] < _isoLevel) cubeIndex |= 1;
    if (val[1] < _isoLevel) cubeIndex |= 2;
    if (val[2] < _isoLevel) cubeIndex |= 4;
    if (val[3] < _isoLevel) cubeIndex |= 8;
    if (val[4] < _isoLevel) cubeIndex |= 16;
    if (val[5] < _isoLevel) cubeIndex |= 32;
    if (val[6] < _isoLevel) cubeIndex |= 64;
    if (val[7] < _isoLevel) cubeIndex |= 128;

    // 5. Check if Surface Exists
    if (edgeTable[cubeIndex] == 0) return;

    // 6. Calculate Edge Vertices
    float3 vertlist[12];
    if (edgeTable[cubeIndex] & 1) vertlist[0] = VertexInterp(p[0], p[1], val[0], val[1]);
    if (edgeTable[cubeIndex] & 2) vertlist[1] = VertexInterp(p[1], p[2], val[1], val[2]);
    if (edgeTable[cubeIndex] & 4) vertlist[2] = VertexInterp(p[2], p[3], val[2], val[3]);
    if (edgeTable[cubeIndex] & 8) vertlist[3] = VertexInterp(p[3], p[0], val[3], val[0]);
    if (edgeTable[cubeIndex] & 16) vertlist[4] = VertexInterp(p[4], p[5], val[4], val[5]);
    if (edgeTable[cubeIndex] & 32) vertlist[5] = VertexInterp(p[5], p[6], val[5], val[6]);
    if (edgeTable[cubeIndex] & 64) vertlist[6] = VertexInterp(p[6], p[7], val[6], val[7]);
    if (edgeTable[cubeIndex] & 128) vertlist[7] = VertexInterp(p[7], p[4], val[7], val[4]);
    if (edgeTable[cubeIndex] & 256) vertlist[8] = VertexInterp(p[0], p[4], val[0], val[4]);
    if (edgeTable[cubeIndex] & 512) vertlist[9] = VertexInterp(p[1], p[5], val[1], val[5]);
    if (edgeTable[cubeIndex] & 1024) vertlist[10] = VertexInterp(p[2], p[6], val[2], val[6]);
    if (edgeTable[cubeIndex] & 2048) vertlist[11] = VertexInterp(p[3], p[7], val[3], val[7]);

    // 7. Generate Triangles
    int triIndex = 0;
    while (triTable[cubeIndex][triIndex] != -1)
    {
        int i = triTable[cubeIndex][triIndex];
        int j = triTable[cubeIndex][triIndex + 1];
        int k = triTable[cubeIndex][triIndex + 2];
        triIndex += 3;

        // Create Triangle
        Triangle tri;
        tri.v[0] = CreateVertex(vertlist[i]);
        tri.v[1] = CreateVertex(vertlist[j]);
        tri.v[2] = CreateVertex(vertlist[k]);

        // Append to buffer
        triangleRW.Append(tri);
    }
}

float3 VertexInterp(float3 p1, float3 p2, float valp1, float valp2)
{
    return lerp(p1, p2, (_isoLevel - valp1) / (valp2 - valp1));
}

float3 CalculateGradient(float3 p)
{
    float ratio = 1.0 / (_gridSize - 1);
    return float3(
        SampleLinear(p - float3(1.0, 0.0, 0.0) * ratio) - SampleLinear(p + float3(1.0, 0.0, 0.0) * ratio),
        SampleLinear(p - float3(0.0, 1.0, 0.0) * ratio) - SampleLinear(p + float3(0.0, 1.0, 0.0) * ratio),
        SampleLinear(p - float3(0.0, 0.0, 1.0) * ratio) - SampleLinear(p + float3(0.0, 0.0, 1.0) * ratio)
    );
}

float SampleLinear(float3 p)
{
    // Offset to align with texture coordinate space [0,1]
    return _densityTexture.SampleLevel(myLinearClampSampler, p + float3(0.5, 0.5, 0.5), 0).x;
}

float SampleData(int3 pos)
{
    return _densityTexture.Load(int4(pos, 0)).x;
}
```

### Edge Cases and Pitfalls
*   **Division by Zero**: If `delta` (difference between two corner values) is zero in `GetOffset`, return `0.5f` as the offset to avoid division by zero [S2, unofficial].
*   **Boundary Conditions**: Boundary voxels lack neighbors for correct cube construction and normal calculation. Handled by skipping voxels within `_Border` distance of the grid edges [S2, unofficial].
*   **Linear Sampling Offset**: When sampling the density texture for gradient calculation, an offset of `float3(0.5, 0.5, 0.5)` is added to the coordinate `p` before sampling to align with the texture coordinate space $[0,1]$ [S1, unofficial].
*   **Performance**: Using structured buffers for voxel and normal data is noted as slower than render textures but functional [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N^3)$, where $N$ is the grid resolution, as each voxel is processed in parallel [S1, S2, unofficial].
*   **Memory Complexity**: $O(N^3)$ for input volume and output geometry buffers [S1, S2, unofficial].
*   **Performance**: Performance depends on GPU compute shader efficiency and the density of the isosurface. Dispatching $N/8$ groups in each dimension allows for efficient parallel processing [S2, unofficial].
*   **Strengths**: High parallelism, suitable for real-time extraction of dynamic volumes (e.g., Perlin noise) [S2, unofficial].
*   **Limitations**: Structured buffer I/O can be slower than texture-based approaches [S2, unofficial]. Boundary handling requires explicit checks or padding [S2, unofficial].

## Background
Marching Cubes on GPU is an adaptation of the classic Marching Cubes algorithm [S2, unofficial] to run on GPU compute shaders. It was implemented in examples such as those by Pavel Kouril [S1, unofficial] and Keijiro [S2, unofficial], often in the context of Unity engine development. The technique allows for real-time isosurface extraction from 3D scalar fields, such as those generated by 4D Perlin noise [S2, unofficial].

## Sources
[S1] pavelkouril/unity-marching-cubes-gpu, https://github.com/pavelkouril/unity-marching-cubes-gpu, fetched 2026-10-26 (unofficial)
[S2] keijiro/MarchingCubesOnGPU, https://github.com/keijiro/MarchingCubesOnGPU, fetched 2026-10-26 (unofficial)
