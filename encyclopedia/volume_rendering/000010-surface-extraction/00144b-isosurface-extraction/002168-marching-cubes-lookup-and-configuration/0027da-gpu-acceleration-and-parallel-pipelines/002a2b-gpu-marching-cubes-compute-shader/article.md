# GPU Marching Cubes (Compute Shader)

Also called: Marching Cubes GPU, Unity Marching Cubes Compute Shader

## Overview
This method implements the Marching Cubes isosurface extraction algorithm using a GPU compute shader within the Unity engine environment [S1, unofficial]. It offloads voxel processing and triangle generation to the GPU while generating the density field on the CPU in its basic form [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D texture (`Texture3D`) containing scalar density values, a grid resolution integer, and an isosurface threshold value [S1, unofficial].
*   **Output:** A structured append buffer (`AppendStructuredBuffer<Triangle>`) containing a list of triangles, where each triangle consists of three vertices with position (`float3`) and normal (`float3`) attributes [S1, unofficial].

**Data Structures:**
1.  **Lookup Tables:** Static constant arrays within the compute shader. `edgeTable` is a 256-entry array of hex values (bitmasks) [S1, unofficial]. `triTable` is a 256x16 array of integers, where each row defines up to 5 triangles (15 indices) for a given cube configuration, padded with -1 [S1, unofficial].
2.  **Append Buffer (`appendVertexBuffer`):** A `ComputeBuffer` of type `Append` with size calculated as $(Resolution - 1)^3 \times 5$ elements, where each element is 18 bytes ($3 \text{ vertices} \times 6 \text{ floats}$) [S1, unofficial].
3.  **Argument Buffer (`argBuffer`):** A `ComputeBuffer` of type `IndirectArguments` with size 4 integers (16 bytes) [S1, unofficial].

**Parameters:**
*   `_gridSize`: Integer, set to `Resolution` [S1, unofficial].
*   `_isoLevel`: Float, default 0.5f [S1, unofficial].

**Algorithm Steps:**
1.  **Kernel Initialization:** Find the compute shader kernel named "MarchingCubes" using `FindKernel("MarchingCubes")` [S1, unofficial].
2.  **Buffer Allocation:** Allocate `appendVertexBuffer` with capacity $(Resolution - 1)^3 \times 5$ elements of 18 bytes each, and allocate `argBuffer` with 4 integers [S1, unofficial].
3.  **Parameter Setup:** Set `_gridSize` to `Resolution` and `_isoLevel` to the desired threshold. Bind the density texture and `appendVertexBuffer` to the shader's "triangleRW" buffer [S1, unofficial].
4.  **Counter Reset:** Call `SetCounterValue(0)` on `appendVertexBuffer` to clear previous frame's output [S1, unofficial].
5.  **Dispatch:** Dispatch the compute shader kernel with thread groups of size $(Resolution/8, Resolution/8, Resolution/8)$, implying an 8x8x8 thread group size [S1, unofficial].
6.  **Count Retrieval:** Initialize `argBuffer` with `{0, 1, 0, 0}`. Use `ComputeBuffer.CopyCount` to copy the append buffer's current element count to `argBuffer` at offset 0. Read back the count from `argBuffer` [S1, unofficial].
7.  **Argument Adjustment:** Multiply the retrieved vertex count (`args[0]`) by 3 to get the total index count for triangles (assuming 3 indices per triangle). Update `argBuffer` with the new arguments [S1, unofficial].
8.  **Rendering:** In the rendering pass, set the material pass, bind `appendVertexBuffer` to the shader's "triangles" buffer, set the model matrix, and call `Graphics.DrawProceduralIndirect` with `MeshTopology.Triangles` and the `argBuffer` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Lookup tables defined as static constants in the compute shader [S1, unofficial]
edgeTable = [0x0, 0x1, ...]  # 256 hex values
triTable = [[-1, -1, ...]]   # 256x16 ints, padded with -1

class Vertex:
    def __init__(self, pos, normal):
        self.pos = pos
        self.normal = normal

class Triangle:
    def __init__(self, v0, v1, v2):
        self.v = [v0, v1, v2]

def interpolate_edge(vertices, edge_index, iso_level):
    # Standard MC edge-to-vertex mapping [G]
    edge_map = [
        (0, 1), (1, 2), (2, 3), (3, 0),  # bottom face
        (4, 5), (5, 6), (6, 7), (7, 4),  # top face
        (0, 4), (1, 5), (2, 6), (3, 7)   # vertical edges
    ]
    v0_idx, v1_idx = edge_map[edge_index]
    v0, v1 = vertices[v0_idx], vertices[v1_idx]
    t = (iso_level - v0) / (v1 - v0)
    # Linear interpolation for position [G]
    pos = [0.0, 0.0, 0.0]
    # Gradient-based normal calculation [G]
    normal = [0.0, 0.0, 0.0]
    return Vertex(pos, normal)

def marching_cubes_kernel(density_texture, grid_size, iso_level, triangle_rw):
    # Thread mapping: one thread group processes one voxel cube [S1, unofficial]
    # Valid cube indices range from 0 to N-2 [S1, unofficial]
    for z in range(grid_size - 1):
        for y in range(grid_size - 1):
            for x in range(grid_size - 1):
                # 1. Sample the 8 vertices of the current voxel cube
                v = [
                    density_texture[x, y, z],
                    density_texture[x+1, y, z],
                    density_texture[x+1, y+1, z],
                    density_texture[x, y+1, z],
                    density_texture[x, y, z+1],
                    density_texture[x+1, y, z+1],
                    density_texture[x+1, y+1, z+1],
                    density_texture[x, y+1, z+1]
                ]

                # 2. Compute Case Index C
                C = 0
                for i in range(8):
                    if v[i] < iso_level:
                        C |= (1 << i)

                # 3. Lookup Triangle Configuration
                edge_mask = edgeTable[C]
                tri_indices = triTable[C]

                # 4. Generate Triangles
                idx = 0
                while tri_indices[idx] != -1:
                    e0 = tri_indices[idx]
                    e1 = tri_indices[idx+1]
                    e2 = tri_indices[idx+2]
                    idx += 3

                    p0 = interpolate_edge(v, e0, iso_level)
                    p1 = interpolate_edge(v, e1, iso_level)
                    p2 = interpolate_edge(v, e2, iso_level)

                    # 5. Append to Output Buffer
                    # Atomic counter allocation handled by GPU append buffer [S1, unofficial]
                    triangle_rw.append(Triangle(p0, p1, p2))
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The algorithm operates on cubes defined by grid points. If the grid resolution is $N$, valid cube indices range from $0$ to $N-2$ [S1, unofficial].
*   **Precision:** Floating-point interpolation may introduce small artifacts. Gradient-based normal calculation is recommended for smooth shading [S1, unofficial].
*   **Hardware Dependency:** The implementation assumes DX11 hardware support. Mobile support is uncertain [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N^3)$ for a grid of resolution $N$, as each voxel is processed by one thread group [S1, unofficial].
*   **Memory Complexity:** $O(N^3)$ for the input density texture and $O(N^3)$ for the output buffer (worst-case 5 triangles per voxel) [S1, unofficial].
*   **Performance:** Depends heavily on GPU compute capabilities and the efficiency of the append buffer implementation. Atomic counters can cause contention; histogram-based approaches may offer better scalability [S1, unofficial].
*   **Strengths:** Real-time extraction on GPU; leverages Unity's compute shader infrastructure [S1, unofficial].
*   **Limitations:** CPU-bound density generation in the basic implementation; fixed maximum output size requires careful buffer sizing; DX11 dependency [S1, unofficial].

## Background
This implementation was developed for Unity 5.6, building upon the original Marching Cubes algorithm and lookup tables by Paul Bourke [S1, unofficial]. It incorporates gradient-based normal calculations from an advanced Marching Cubes resource [S1, unofficial].

## Sources
[S1] pavelkouril/unity-marching-cubes-gpu, https://github.com/pavelkouril/unity-marching-cubes-gpu, fetched 2026-10-26 (unofficial)
