# CPU Immediate Mode Marching Cubes

Also called: CPU Immediate Sim, Immediate Mode Marching Cubes, Legacy Marching Cubes

## Overview
CPU Immediate Mode Marching Cubes is a classic CPU-based surface extraction method that generates mesh vertices and normals in immediate mode, rendering them directly via WebGL draw calls without storing a persistent mesh buffer [S1, unofficial]. It differs from buffered or GPU-accelerated siblings by performing all field evaluation and geometry generation on the CPU every frame, batching small chunks of geometry to minimize API overhead [S1, unofficial]. This approach serves as a reference implementation for correctness and simplicity but suffers from CPU bottlenecks at high resolutions or large blob counts [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D grid of scalar values (field) representing metaballs or implicit surfaces; an isovalue threshold; WebGL context.
- **Output:** A stream of triangle primitives rendered to the screen.

**Data Structures:**
The implementation uses fixed-size typed arrays to hold vertex attributes for immediate batching [S1, unofficial].
- `posArray`: `Float32Array` of size `maxCount * 3`. Stores vertex positions $(x, y, z)$.
- `normalArray`: `Float32Array` of size `maxCount * 3`. Stores vertex normals $(nx, ny, nz)$.
- `uvArray`: `Float32Array` of size `maxCount * 2`. Stores UV coordinates (optional).
- `colorArray`: `Float32Array` of size `maxCount * 4`. Stores vertex colors (optional).
- `maxCount`: Integer, typically 4096 [S1, unofficial]. This defines the batch size.
- `count`: Integer, tracks the number of vertices currently in the buffer.
- `prim`: Primitive type (e.g., `TRIANGLES`).
- `primsize`: Integer, 3 for triangles, 2 for lines, 1 for points.
- `hasPos`, `hasNormal`, `hasUV`, `hasColor`: Boolean flags indicating which attributes are populated for the current batch.

**Algorithm Steps:**

1.  **Field Generation:**
    For each voxel in the 3D grid, compute the scalar field value $f(x,y,z)$ by accumulating contributions from all metaballs (blobs) and a floor term [S1, unofficial].
    $$ f(x,y,z) = \sum_{i} \text{blob}_i(x,y,z) + \text{floor}(x,y,z) $$
    [S1, unofficial]

2.  **Marching Cubes Extraction:**
    Iterate over each cube in the grid. Use precomputed lookup tables (e.g., Paul Bourke's tables) to determine which edges of the cube are intersected by the isosurface [S1, unofficial].
    - Determine the cube index based on the sign of the scalar values at the 8 vertices relative to the isovalue.
    - For each intersected edge, interpolate the vertex position and normal using linear interpolation between the two vertices of the edge [S1, unofficial].
    - Generate triangles based on the configuration defined by the lookup table.

3.  **Immediate Mode Rendering Loop:**
    The rendering is done in batches to reduce WebGL state changes and draw calls.

    **Pseudocode:**

    ```python
    # Initialization
    maxCount = 4096
    posArray = Float32Array(maxCount * 3)
    normalArray = Float32Array(maxCount * 3)
    uvArray = Float32Array(maxCount * 2)
    colorArray = Float32Array(maxCount * 4)
    count = 0
    prim = TRIANGLES
    primsize = 3
    hasPos = false
    hasNormal = false
    hasUV = false
    hasColor = false

    # Function to add a vertex position
    def add_vertex_position(x, y, z):
        nonlocal count, hasPos
        posArray[count * 3 + 0] = x
        posArray[count * 3 + 1] = y
        posArray[count * 3 + 2] = z
        hasPos = true
        count += 1
        # Check for buffer overflow before adding more vertices
        if count >= maxCount - primsize:
            flush_and_render()

    # Function to add a vertex normal
    def add_vertex_normal(nx, ny, nz):
        nonlocal count, hasNormal
        normalArray[count * 3 + 0] = nx
        normalArray[count * 3 + 1] = ny
        normalArray[count * 3 + 2] = nz
        hasNormal = true
        # Note: count is incremented by add_vertex_position, 
        # normals are stored at the same index as the corresponding position

    # Function to add a triangle
    def add_triangle(o1, o2, o3):
        nonlocal count
        # Copy positions from pos array at offsets o1, o2, o3
        # Offsets are in terms of vertex indices, so multiply by 3 for array indices
        for i in range(3):
            src_idx = o1 * 3 + i
            dst_idx = count * 3 + i
            posArray[dst_idx] = posArray[src_idx]
        
        for i in range(3):
            src_idx = o2 * 3 + i
            dst_idx = count * 3 + i
            posArray[dst_idx] = posArray[src_idx]

        for i in range(3):
            src_idx = o3 * 3 + i
            dst_idx = count * 3 + i
            posArray[dst_idx] = posArray[src_idx]

        # Copy normals
        for i in range(3):
            src_idx = o1 * 3 + i
            dst_idx = count * 3 + i
            normalArray[dst_idx] = normalArray[src_idx]

        for i in range(3):
            src_idx = o2 * 3 + i
            dst_idx = count * 3 + i
            normalArray[dst_idx] = normalArray[src_idx]

        for i in range(3):
            src_idx = o3 * 3 + i
            dst_idx = count * 3 + i
            normalArray[dst_idx] = normalArray[src_idx]

        count += 3
        # Check for buffer overflow
        if count >= maxCount - 3:
            flush_and_render()

    # Function to flush and render
    def flush_and_render():
        nonlocal count, hasPos, hasNormal, hasUV, hasColor
        if count == 0:
            return
        
        # Zero out remaining entries to prevent garbage data
        for i in range(count * 3, maxCount * 3):
            posArray[i] = 0.0
        for i in range(count * 3, maxCount * 3):
            normalArray[i] = 0.0
        # UV and Color arrays zeroing omitted for brevity if not used
        
        # Bind buffers
        gl.bufferData(gl.ARRAY_BUFFER, posArray, gl.DYNAMIC_DRAW)
        gl.bufferData(gl.ARRAY_BUFFER, normalArray, gl.DYNAMIC_DRAW)
        
        # Enable vertex attribute arrays
        if hasPos:
            gl.enableVertexAttribArray(0)
            gl.vertexAttribPointer(0, 3, gl.FLOAT, false, 0, 0)
        if hasNormal:
            gl.enableVertexAttribArray(1)
            gl.vertexAttribPointer(1, 3, gl.FLOAT, false, 0, 0)
        
        # Draw
        gl.drawArrays(prim, 0, count)
        
        # Reset state
        count = 0
        hasPos = false
        hasNormal = false
        hasUV = false
        hasColor = false
    ```

**Parameters:**
- `maxCount`: 4096 [S1, unofficial].
- `prim`: TRIANGLES [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Buffer Overflow:** The code checks if `count >= maxCount - 3` before drawing to ensure space for a triangle (3 vertices). If exceeded, it flushes immediately [S1, unofficial].
- **Precision:** Linear interpolation for vertex positions and normals assumes linear variation within the cube. This is standard for Marching Cubes but may introduce artifacts in highly non-linear fields.
- **Boundary Conditions:** The field generation must handle the "floor term" and edge voxels correctly to avoid artifacts at the volume boundaries [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the grid. Each voxel is processed once for field evaluation and once for marching cubes.
- **Memory Complexity:** $O(maxCount)$ for the immediate mode buffers, plus $O(V)$ for the field data where $V$ is the number of voxels.
- **Performance:** Limited by CPU performance. At high resolutions (e.g., $128^3$) or with many metaballs, the CPU becomes a bottleneck due to the per-voxel calculations and JavaScript overhead [S1, unofficial].
- **Strengths:** Simple implementation, no persistent mesh storage required, good for reference and low-resolution demos.
- **Limitations:** Poor scalability, high CPU usage, not suitable for real-time high-resolution rendering on CPU.

## Background
The CPU Immediate Mode Marching Cubes implementation is inspired by Gregg Tavares' (Greggman) "Metaballs in JavaScript" demo [S1, unofficial]. It uses classic Marching Cubes tables from Paul Bourke [S1, unofficial]. The project is part of the `WebGL2-GPU-Marching-Cubes` repository by 3merillon, which also includes a GPU-accelerated version for comparison [S1, unofficial].

## Sources
[S1] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
