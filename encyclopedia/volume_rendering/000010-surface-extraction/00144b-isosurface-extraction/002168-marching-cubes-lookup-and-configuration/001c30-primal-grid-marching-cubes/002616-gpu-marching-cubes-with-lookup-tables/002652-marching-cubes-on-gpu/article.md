# Marching Cubes on GPU

Also called: MarchingCubesOnGPU

## Overview
A GPU-based surface extraction technique that constructs an isosurface mesh from a 3D scalar volume (typically generated via 4D Perlin noise) using compute shaders to parallelize the Marching Cubes algorithm [S1, unofficial]. It differs from CPU implementations by offloading the voxel processing and triangle generation to the GPU, and from lookup-table-based GPU variants by using structured buffers for data storage rather than render textures, which is noted as slower but functional [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid of scalar values (`_Voxels`), a 3D grid of normals (`_Normals`), and two lookup tables (`_CubeEdgeFlags`, `_TriangleConnectionTable`).
*   **Output:** A structured buffer of vertices (`_Buffer`) containing position and normal data, which is subsequently rendered by a surface shader.

**Data Structures:**
*   `_Voxels`: `StructuredBuffer<float>`, size $N \times N \times N$.
*   `_Normals`: `StructuredBuffer<float3>`, size $N \times N \times N$.
*   `_CubeEdgeFlags`: `StructuredBuffer<int>`, 256 entries. Each integer is a bitmask where bit $n$ is set if edge $n$ is intersected.
*   `_TriangleConnectionTable`: `StructuredBuffer<int>`, 256 entries $\times$ 16 integers. Defines the triangulation for each of the 256 cube cases. Entries are terminated by -1.
*   `_Buffer`: `RWStructuredBuffer<Vert>`, max size $N \times N \times N \times 15$ (5 triangles $\times$ 3 vertices per cube).
    *   `Vert` struct:
        *   `position`: `float4` (xyz: position, w: 1.0).
        *   `normal`: `float3`.

**Parameters:**
*   `_Target`: Isovalue threshold. Typical value: `0.0f` [S1, unofficial].
*   `_Width`, `_Height`, `_Depth`: Grid dimensions. In the reference implementation, all equal $N$ (default 40) [S1, unofficial].
*   `_Border`: Boundary voxels to skip. Typical value: `1` [S1, unofficial].
*   Thread group size: `[numthreads(8,8,8)]` [S1, unofficial].

**Algorithm Steps:**
1.  **Dispatch:** Launch compute shader with $N/8$ groups in each dimension.
2.  **Boundary Check:** If the current thread's coordinates $(id.x, id.y, id.z)$ are within `_Border` distance of the grid edges (i.e., $id.x \ge Width - 1 - Border$, etc.), return immediately to avoid accessing out-of-bounds neighbors [S1, unofficial].
3.  **Sample Voxels:** Load the 8 corner values of the current cube into `cube[0..7]` using linear indexing:
    $$ cube[i] = \text{_Voxels}[x + (i\&1) + (y + ((i>>1)\&1)) \cdot Width + (z + ((i>>2)\&1)) \cdot Width \cdot Height] $$
4.  **Compute Cube Flag:** Initialize `flagIndex = 0`. For $i$ from 0 to 7: if `cube[i] <= _Target`, set bit $i$: `flagIndex |= (1 << i)` [S1, unofficial].
5.  **Calculate Edge Intersections:** Initialize `edgeVertex[12]`. For each edge $i$ from 0 to 11:
    *   If bit $i$ is set in `_CubeEdgeFlags[flagIndex]`:
        *   Let $e_x = \text{edgeConnection}[i].x$ and $e_y = \text{edgeConnection}[i].y$ be the indices of the two vertices defining edge $i$.
        *   Calculate offset: `offset = GetOffset(cube[e_x], cube[e_y])`.
        *   Calculate position: `edgeVertex[i] = pos + vertexOffset[e_x] + offset * edgeDirection[i]`, where `pos` is the cube's base position, `vertexOffset` maps vertex indices to local coordinates, and `edgeDirection` maps edge indices to direction vectors [S1, unofficial].
6.  **Generate Triangles:** Calculate output buffer index `idx = id.x + id.y * Width + id.z * Width * Height`.
    *   For each triangle $i$ from 0 to 4:
        *   Check if `_TriangleConnectionTable[flagIndex * 16 + 3*i] >= 0`.
        *   If valid, for each vertex $j$ from 0 to 2:
            *   Get vertex index $v = \text{_TriangleConnectionTable}[flagIndex * 16 + (3*i+j)]$.
            *   Get position `p = edgeVertex[v]`.
            *   Compute normal `n = SampleBilinear(_Normals, p)`.
            *   Store `Vert` with `position = float4(p - centre, 1.0)` and `normal = n` in `_Buffer[idx * 15 + (3*i+j)]`, where `centre = float3(Width, 0, Depth) / 2.0` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
NUM_THREADS_X = 8
NUM_THREADS_Y = 8
NUM_THREADS_Z = 8

# Helper: Linear interpolation offset
def GetOffset(v1, v2):
    delta = v2 - v1
    if delta == 0.0:
        return 0.5
    return (_Target - v1) / delta

# Helper: Bilinear/Trilinear Normal Sampling
def SampleBilinear(normals_buffer, pos):
    # Assumes normals_buffer is a 3D grid of float3
    # Perform trilinear interpolation based on pos fractional parts
    fx = pos.x - floor(pos.x)
    fy = pos.y - floor(pos.y)
    fz = pos.z - floor(pos.z)
    
    # Get 8 surrounding normal vectors
    # Note: Exact implementation details for trilinear interpolation of normals
    # are described as "calculate fractional parts... compute x0, x1... interpolate"
    # This is a placeholder for the specific bilinear/trilinear math provided in [S1]
    n000 = normals_buffer[...]; n100 = normals_buffer[...] # ...
    n001 = normals_buffer[...]; n101 = normals_buffer[...] # ...
    n010 = normals_buffer[...]; n110 = normals_buffer[...] # ...
    n011 = normals_buffer[...]; n111 = normals_buffer[...] # ...
    
    # Interpolate in X
    n00 = lerp(n000, n100, fx)
    n10 = lerp(n001, n101, fx)
    n01 = lerp(n010, n110, fx)
    n11 = lerp(n011, n111, fx)
    
    # Interpolate in Y
    n0 = lerp(n00, n01, fy)
    n1 = lerp(n10, n11, fy)
    
    # Interpolate in Z
    return lerp(n0, n1, fz)

# Main Compute Shader Kernel
kernel void MarchingCubesKernel(uint3 id : SV_DispatchThreadID) {
    int Width = _Width;
    int Height = _Height;
    int Depth = _Depth;
    int Border = _Border;
    float Target = _Target;
    
    // Boundary check
    if (id.x >= Width - 1 - Border || 
        id.y >= Height - 1 - Border || 
        id.z >= Depth - 1 - Border) {
        return;
    }
    
    int x = id.x;
    int y = id.y;
    int z = id.z;
    
    // 1. Sample 8 vertices of the cube
    float cube[8];
    cube[0] = _Voxels[x + y * Width + z * Width * Height];
    cube[1] = _Voxels[(x+1) + y * Width + z * Width * Height];
    cube[2] = _Voxels[x + (y+1) * Width + z * Width * Height];
    cube[3] = _Voxels[(x+1) + (y+1) * Width + z * Width * Height];
    cube[4] = _Voxels[x + y * Width + (z+1) * Width * Height];
    cube[5] = _Voxels[(x+1) + y * Width + (z+1) * Width * Height];
    cube[6] = _Voxels[x + (y+1) * Width + (z+1) * Width * Height];
    cube[7] = _Voxels[(x+1) + (y+1) * Width + (z+1) * Width * Height];
    
    // 2. Compute cube flag index
    int flagIndex = 0;
    for (int i = 0; i < 8; i++) {
        if (cube[i] <= Target) {
            flagIndex |= (1 << i);
        }
    }
    
    // 3. Calculate edge intersections
    float3 edgeVertex[12];
    int edgeFlags = _CubeEdgeFlags[flagIndex];
    
    // Pre-defined edge connections and directions would be needed here.
    // Assuming edgeConnection[i] returns {vertex_index_1, vertex_index_2}
    // and edgeDirection[i] returns the direction vector for edge i.
    // vertexOffset maps vertex index 0-7 to local cube coordinates (0/1).
    
    for (int i = 0; i < 12; i++) {
        if ((edgeFlags & (1 << i)) != 0) {
            int v1_idx = edgeConnection[i].x;
            int v2_idx = edgeConnection[i].y;
            
            float offset = GetOffset(cube[v1_idx], cube[v2_idx]);
            
            // pos is the base position of the cube (x, y, z)
            // vertexOffset[v] gives the local coordinate (0 or 1) for vertex v
            // edgeDirection[i] gives the vector from v1 to v2
            float3 pos = float3(x, y, z);
            float3 v1_local = vertexOffset[v1_idx];
            float3 v2_local = vertexOffset[v2_idx];
            float3 dir = v2_local - v1_local;
            
            edgeVertex[i] = pos + v1_local + offset * dir;
        }
    }
    
    // 4. Generate triangles
    int idx = x + y * Width + z * Width * Height;
    float3 centre = float3(Width, 0, Depth) / 2.0;
    
    for (int i = 0; i < 5; i++) {
        int tri_offset = flagIndex * 16 + 3 * i;
        if (_TriangleConnectionTable[tri_offset] >= 0) {
            for (int j = 0; j < 3; j++) {
                int v = _TriangleConnectionTable[tri_offset + j];
                float3 pos = edgeVertex[v];
                float3 normal = SampleBilinear(_Normals, pos);
                
                int out_idx = idx * 15 + (3 * i + j);
                _Buffer[out_idx].position = float4(pos - centre, 1.0);
                _Buffer[out_idx].normal = normal;
            }
        }
    }
}
```

## Operating characteristics
*   **Time Complexity:** $O(N^3)$, where $N$ is the grid dimension. Each voxel is processed by one thread group.
*   **Memory Complexity:** $O(N^3)$ for input volumes and output buffers. The output buffer size is fixed at $15 \times N^3$ vertices to accommodate the maximum possible triangles per cube.
*   **Performance:** The use of structured buffers for voxel and normal data is noted as slower than using render textures, but functional [S1, unofficial].
*   **Strengths:** Parallel processing of all voxels; integration with GPU-based noise generation (e.g., 4D Perlin noise) [S1, unofficial].
*   **Limitations:** Boundary handling requires skipping voxels, potentially leaving gaps or requiring post-processing; fixed-size output buffer may waste memory if the isosurface is sparse; structured buffers are less efficient than textures for random access in some GPU architectures [S1, unofficial].

## Background
The method is an implementation of the Marching Cubes algorithm adapted for GPU compute shaders, specifically demonstrated in the context of generating 4D Perlin noise volumes [S1, unofficial]. The compute shader code was borrowed from an example by Scrawk [S1, unofficial]. It is part of the "MarchingCubesOnGPU" project by Keijiro [S1, unofficial].

## Sources
[S1] keijiro/MarchingCubesOnGPU, https://github.com/keijiro/MarchingCubesOnGPU, fetched 2026-10-26 (unofficial)
