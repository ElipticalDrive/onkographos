# GPU Marching Cubes (Compute Shader)

Also called: Marching Cubes GPU, Unity Marching Cubes Compute Shader

## Overview
This method implements the Marching Cubes isosurface extraction algorithm using a GPU compute shader, specifically designed for integration within the Unity engine environment [S1, unofficial]. It solves the problem of real-time surface generation from a 3D scalar density field by offloading the voxel classification, topology lookup, and vertex interpolation to the GPU [S1, unofficial]. Unlike CPU-based implementations, it leverages parallel thread execution for grid processing and uses an append buffer with atomic counters to handle the variable number of output triangles, avoiding the need for pre-allocation of exact mesh sizes [S1, unofficial]. It differs from histogram-pyramid approaches by relying on atomic operations for buffer management, which simplifies the pipeline but may introduce contention under high triangle density [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   **Density Texture:** A 3D texture (`Texture3D`) of resolution $R \times R \times R$ containing scalar density values. In the reference implementation, this is generated on the CPU, but the method supports binding a `RenderTexture` for GPU-generated fields [S1, unofficial].
*   **Grid Size ($R$):** An integer defining the resolution of the volume grid.
*   **Isolevel:** A float scalar threshold (default 0.5) used to determine the surface boundary [S1, unofficial].

**Outputs:**
*   **Triangle Buffer:** An append buffer storing the generated mesh geometry.
*   **Vertex Count:** The total number of vertices generated, used for indirect rendering arguments.

### Data Structures

**1. Triangle Struct (Shader Side)**
The output buffer stores triangles as a structured buffer. Each triangle consists of 3 vertices.
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```csharp
struct Vertex {
    float3 vPosition;
    float3 vNormal;
};

struct Triangle {
    Vertex v[3];
};
```
Each vertex contains 6 floats (12 bytes). A triangle contains 3 vertices (36 bytes).

**2. Append Vertex Buffer (CPU/GPU Side)**
*   **Type:** `ComputeBufferType.Append`
*   **Capacity:** $(R-1) \times (R-1) \times (R-1) \times 5$ elements.
    *   The factor 5 represents the maximum number of triangles per voxel in the standard Marching Cubes algorithm.
    *   Element size: $sizeof(float) \times 18$ bytes? *Correction based on code:* The code allocates `(Resolution - 1)^3 * 5` elements of size `sizeof(float) * 18`. However, the struct `Triangle` has 3 vertices, each with `float3` pos and `float3` normal. That is $3 \times (3+3) = 18$ floats. So the buffer is an array of `Triangle` structs, but allocated as a raw byte buffer or structured buffer of size 18 floats. The code uses `ComputeBuffer` with stride 18 floats.
*   **Purpose:** Stores the output triangles.

**3. Argument Buffer**
*   **Type:** `ComputeBufferType.IndirectArguments`
*   **Size:** 4 integers (16 bytes).
*   **Purpose:** Stores draw call arguments: `[vertexCount, instanceCount, startVertex, startInstance]`.

**4. Lookup Tables (Shader Side)**
*   **`edgeTable`:** 256-entry array of integers (hex values). Maps the 8-bit vertex configuration to the set of intersected edges.
*   **`triTable`:** 256x16 array of integers. Maps the 8-bit configuration to up to 5 triangles (15 indices). Entries are padded with -1 if fewer than 5 triangles are generated.

### Numbered Steps

1.  **Initialization (CPU):**
    *   Find the compute shader kernel named "MarchingCubes" using `FindKernel("MarchingCubes")` [S1, unofficial].
    *   Allocate `appendVertexBuffer` with capacity $(R-1)^3 \times 5$ elements of size 18 floats [S1, unofficial].
    *   Allocate `argBuffer` with 4 integers [S1, unofficial].
    *   Set shader parameters: `_gridSize` to $R$, `_isoLevel` to 0.5 [S1, unofficial].
    *   Bind `DensityTexture` to the shader [S1, unofficial].

2.  **Buffer Reset (CPU):**
    *   Call `appendVertexBuffer.SetCounterValue(0)` to reset the append buffer counter to zero before each frame [S1, unofficial].

3.  **Dispatch Compute Shader (CPU):**
    *   Dispatch the "MarchingCubes" kernel.
    *   Thread group dimensions: $(R/8, R/8, R/8)$. This implies 8x8x8 threads per group, processing a sub-volume of the grid [S1, unofficial].

4.  **Compute Shader Execution (GPU):**
    *   Each thread group processes a block of voxels.
    *   For each voxel in the block:
        *   Sample the 8 corner vertices from the `DensityTexture`.
        *   Compute the 8-bit configuration index by comparing vertex values to `_isoLevel`.
        *   Look up the intersected edges in `edgeTable`.
        *   Interpolate vertex positions and normals on the intersected edges.
        *   Look up the triangle indices in `triTable`.
        *   Use an atomic counter to reserve space in `appendVertexBuffer` and write the triangle data [S1, unofficial].

5.  **Retrieve Vertex Count (CPU):**
    *   Initialize `argBuffer` with `{0, 1, 0, 0}` [S1, unofficial].
    *   Use `ComputeBuffer.CopyCount` to copy the current count of `appendVertexBuffer` into `argBuffer[0]` [S1, unofficial].
    *   Read back the count from `argBuffer` to the CPU [S1, unofficial].

6.  **Adjust Draw Arguments (CPU):**
    *   Multiply the vertex count (`args[0]`) by 3 to get the total index count for triangles (assuming 3 indices per triangle) [S1, unofficial].
    *   Update `argBuffer` with the new arguments: `[vertexCount * 3, 1, 0, 0]` [S1, unofficial].

7.  **Render (CPU):**
    *   Set the material pass.
    *   Bind `appendVertexBuffer` to the "triangles" buffer in the shader.
    *   Set the model matrix.
    *   Call `Graphics.DrawProceduralIndirect` with `MeshTopology.Triangles` and the `argBuffer` [S1, unofficial].

### Formulas

**Vertex Interpolation:**
For an edge between vertices $v_1$ and $v_2$ with scalar values $s_1$ and $s_2$:
$$
\text{param} = \frac{\text{Isolevel} - s_1}{s_2 - s_1}
$$
$$
\text{pos} = \text{lerp}(v_1, v_2, \text{param})
$$
$$
\text{normal} = \text{lerp}(\nabla s_1, \nabla s_2, \text{param})
$$
Where $\nabla s$ is the gradient at the vertex, approximated by finite differences or provided by the density field [S1, unofficial].

**Buffer Capacity:**
$$
\text{Capacity} = (R-1)^3 \times 5
$$
Where $R$ is the grid resolution [S1, unofficial].

### Parameters

*   **Resolution ($R$):** The dimension of the cubic volume grid. Typical values depend on desired detail and GPU memory. The reference implementation uses a variable $R$ [S1, unofficial].
*   **Isolevel:** The scalar threshold for surface extraction. Default: 0.5 [S1, unofficial].
*   **Thread Group Size:** 8x8x8 threads per group [S1, unofficial].

### Edge Cases and Pitfalls

*   **Hardware Dependency:** The implementation assumes DX11 hardware support. Mobile support is uncertain and likely unsupported due to compute shader limitations on older mobile GPUs [S1, unofficial].
*   **CPU Bottleneck:** The reference implementation generates the density field on the CPU. For real-time applications, this should be replaced with a GPU-generated `RenderTexture` to avoid CPU-GPU synchronization bottlenecks [S1, unofficial].
*   **Atomic Contention:** Using atomic counters for the append buffer can lead to contention if many threads attempt to write simultaneously. This may reduce performance in high-density scenarios compared to histogram-pyramid approaches [S1, unofficial].
*   **Buffer Overflow:** The append buffer is pre-allocated with a maximum capacity. If the number of triangles exceeds $(R-1)^3 \times 5$, data will be lost. This is unlikely for standard Marching Cubes but possible if the isosurface is extremely complex or if the algorithm is modified [S1, unofficial].
*   **Gradient Calculation:** The reference implementation uses a specific method for gradient normals. Incorrect gradient calculation can lead to visual artifacts [S1, unofficial].

## Subtopics

### GPU Marching Cubes with Histogram Pyramid
Uses hierarchical reduction to determine exact triangle counts and offsets, avoiding atomic contention. Choose this when performance is critical and atomic bottlenecks are observed. [Link](2317-gpu-marching-cubes-with-histogram-pyramid.md)

### GPU Marching Cubes with Atomic Counter Emission
A more generic implementation of the atomic counter approach, potentially with different buffer management strategies. Choose this for simpler implementations where histogram overhead is not justified. [Link](4082-gpu-marching-cubes-with-atomic-counter-emission.md)

### GPU Marching Cubes Terrain Pipeline with Indirect Rendering
Focuses on terrain-specific optimizations and indirect rendering techniques. Choose this for large-scale terrain generation. [Link](3803-gpu-marching-cubes-terrain-pipeline-with-indirect-rendering.md)

### GPU-accelerated Marching Cubes with Distance Extraction
Incorporates distance field extraction for smoother surfaces. Choose this when surface smoothness is a priority. [Link](3818-gpu-accelerated-marching-cubes-with-distance-extraction.md)

## Operating characteristics

*   **Time Complexity:** $O(N)$ where $N = R^3$ is the number of voxels. Each voxel is processed in constant time by a thread.
*   **Memory Complexity:** $O(N)$ for the input density texture and $O(N)$ for the output triangle buffer (worst-case).
*   **Measured Performance:** Not given in the sources. Performance depends heavily on GPU architecture, resolution, and triangle density.
*   **Scaling:** Scales linearly with resolution. Higher resolutions require larger buffers and more compute time.
*   **Strengths:**
    *   Simple implementation using standard compute shaders.
    *   No need for complex histogram pyramid structures.
    *   Direct integration with Unity's rendering pipeline.
*   **Limitations:**
    *   Atomic counter contention can limit performance.
    *   CPU-based density generation is a bottleneck in the reference implementation.
    *   DX11 hardware dependency limits mobile support.
*   **When to Use:** Real-time applications in Unity where moderate resolution and simplicity are prioritized over maximum performance.
*   **When Not to Use:** High-resolution terrain generation, mobile platforms without DX11 support, or scenarios requiring maximum throughput where histogram pyramid methods are preferred.

## Background

This implementation was developed for Unity 5.6 by Pavel Kouril [S1, unofficial]. It builds upon the classic Marching Cubes algorithm by Paul Bourke and the gradient normals calculation method described in "MCImproved" [S1, unofficial]. The code was contributed to the GitHub repository `pavelkouril/unity-marching-cubes-gpu` [S1, unofficial]. It is an unofficial implementation, not peer-reviewed, and serves as a demonstration of GPU-based Marching Cubes in Unity [S1, unofficial].

## Sources

[S1] pavelkouril/unity-marching-cubes-gpu, https://github.com/pavelkouril/unity-marching-cubes-gpu, fetched 2026-10-26 (unofficial)
