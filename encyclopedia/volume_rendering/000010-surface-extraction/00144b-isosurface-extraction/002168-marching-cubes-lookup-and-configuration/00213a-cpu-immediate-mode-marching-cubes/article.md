# CPU Immediate Mode Marching Cubes

Also called: CPU Immediate Sim, Immediate Mode Marching Cubes, Legacy Marching Cubes

## Overview
CPU Immediate Mode Marching Cubes is a classic CPU-based surface extraction method that generates mesh vertices and normals in immediate mode, rendering them directly via WebGL draw calls without storing a persistent mesh buffer [S1, unofficial]. It differs from modern deferred or GPU-accelerated approaches by performing all field evaluation, topology determination, and geometry batching on the CPU, sending only small uniform and texture updates per frame to the GPU [S1, unofficial]. This makes it a valuable reference implementation for comparison, though it is limited in performance at high resolutions or high blob counts due to CPU bottlenecks [S1, unofficial].

## How it works
The method operates on a discretized 3D grid (typically Cartesian) where each voxel stores a scalar value. It iterates over every cell in the grid, determines the isosurface intersection using precomputed lookup tables, and streams the resulting triangles to the GPU for immediate rendering.

### Inputs and Outputs
*   **Input**: A 3D scalar field $V$ of dimensions $N_x \times N_y \times N_z$, an isovalue $T$, and a set of metaball sources (if applicable) defined by position and radius.
*   **Output**: A stream of triangle primitives rendered to the screen. No persistent mesh data structure is retained in CPU memory between frames.

### Data Structures
The implementation uses fixed-size typed arrays to batch geometry before sending it to the GPU. The maximum batch size is defined by `maxCount`.

*   `maxCount`: Integer, typically 4096 [S1, unofficial].
*   `posArray`: `Float32Array` of size `maxCount * 3`. Stores vertex positions $(x, y, z)$ [S1, unofficial].
*   `normalArray`: `Float32Array` of size `maxCount * 3`. Stores vertex normals $(n_x, n_y, n_z)$ [S1, unofficial].
*   `uvArray`: `Float32Array` of size `maxCount * 2`. Stores UV coordinates (optional) [S1, unofficial].
*   `colorArray`: `Float32Array` of size `maxCount * 4`. Stores vertex colors (optional) [S1, unofficial].
*   `count`: Integer tracking the number of vertices currently in the batch [S1, unofficial].
*   `prim`: Primitive type identifier (e.g., `TRIANGLES`, `LINES`, `POINTS`) [S1, unofficial].
*   `primsize`: Integer, 3 for triangles, 2 for lines, 1 for points [S1, unofficial].
*   `hasPos`, `hasNormal`, `hasUV`, `hasColor`: Boolean flags indicating which attributes are populated for the current batch [S1, unofficial].

### Algorithm Steps

1.  **Field Generation**: For each voxel $(i, j, k)$ in the grid, compute the scalar value $V(i, j, k)$ by accumulating contributions from all metaballs and a floor term.
    $$ V(i, j, k) = \sum_{m} \frac{r_m^2}{\| \mathbf{p}_{ijk} - \mathbf{c}_m \|^2} + V_{floor} $$
    Where $\mathbf{p}_{ijk}$ is the center of the voxel, $\mathbf{c}_m$ is the center of metaball $m$, and $r_m$ is its radius. [S1, unofficial]

2.  **Batch Initialization**:
    *   Set `prim` to the desired primitive type.
    *   Set `primsize` to 3 (for triangles).
    *   Reset `count` to 0.
    *   Reset `hasPos`, `hasNormal`, `hasUV`, `hasColor` to `false`.

3.  **Grid Traversal**: Iterate over every cell $(i, j, k)$ in the grid, where $0 \le i < N_x-1$, $0 \le j < N_y-1$, $0 \le k < N_z-1$.

4.  **Cube Configuration**:
    *   Retrieve the scalar values at the 8 vertices of the current cube: $v_0, \dots, v_7$.
    *   Compute the cube index (case ID) by comparing each vertex value to the isovalue $T$. If $v_n \ge T$, set bit $n$ to 1; otherwise 0.
    *   Look up the edge intersections and triangle connectivity from precomputed Marching Cubes tables (e.g., Paul Bourke's tables) [S1, unofficial].

5.  **Vertex Interpolation**:
    *   For each intersected edge, calculate the intersection point $\mathbf{p}$ and normal $\mathbf{n}$.
    *   The intersection point is found by linear interpolation along the edge between vertices $\mathbf{a}$ and $\mathbf{b}$ with values $v_a$ and $v_b$:
        $$ \mathbf{p} = \mathbf{a} + \frac{T - v_a}{v_b - v_a} (\mathbf{b} - \mathbf{a}) $$
    *   The normal is computed as the gradient of the scalar field at $\mathbf{p}$, normalized:
        $$ \mathbf{n} = \frac{\nabla V(\mathbf{p})}{\| \nabla V(\mathbf{p}) \|} $$
    *   [G] The gradient $\nabla V$ can be approximated using central differences on the scalar field $V$.

6.  **Batching and Rendering**:
    *   For each triangle defined by the lookup table (vertices $o_1, o_2, o_3$):
        *   Check if `count >= maxCount - 3`. If true, call `flushAndRender()`.
        *   Add vertex positions: Store $\mathbf{p}_{o1}, \mathbf{p}_{o2}, \mathbf{p}_{o3}$ into `posArray` at indices `[count*3 + 0]`, `[count*3 + 1]`, `[count*3 + 2]`. Set `hasPos = true`.
        *   Add vertex normals: Store $\mathbf{n}_{o1}, \mathbf{n}_{o2}, \mathbf{n}_{o3}$ into `normalArray` at indices `[count*3]`, `[count*3 + 1]`, `[count*3 + 2]`. Set `hasNormal = true`.
        *   Increment `count` by 3.

7.  **Flush and Render**:
    *   If `count == 0`, return.
    *   Zero out remaining entries in `posArray` and `normalArray` from `count*3` to `length` to prevent garbage data.
    *   Bind buffers: Use `gl.bufferData` with `DYNAMIC_DRAW` for `posBuf` and `normalBuf` (and others if enabled).
    *   Enable vertex attribute arrays and set pointers using `gl.vertexAttribPointer`.
    *   Call `gl.drawArrays(prim, 0, count)`.
    *   Reset `count` to 0.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_COUNT = 4096
PRIM_SIZE = 3 # For triangles

# Data Structures
pos_array = Float32Array(MAX_COUNT * 3)
normal_array = Float32Array(MAX_COUNT * 3)
count = 0
has_pos = False
has_normal = False

def add_vertex_position(x, y, z):
    global count, has_pos
    pos_array[count * 3 + 0] = x
    pos_array[count * 3 + 1] = y
    pos_array[count * 3 + 2] = z
    has_pos = True

def add_vertex_normal(nx, ny, nz):
    global count, has_normal
    normal_array[count * 3 + 0] = nx
    normal_array[count * 3 + 1] = ny
    normal_array[count * 3 + 2] = nz
    has_normal = True

def add_triangle(o1, o2, o3):
    global count
    # Check buffer overflow before adding 3 vertices
    if count >= MAX_COUNT - 3:
        flush_and_render()
    
    # Add positions
    add_vertex_position(pos_array[o1*3], pos_array[o1*3+1], pos_array[o1*3+2])
    add_vertex_position(pos_array[o2*3], pos_array[o2*3+1], pos_array[o2*3+2])
    add_vertex_position(pos_array[o3*3], pos_array[o3*3+1], pos_array[o3*3+2])
    
    # Add normals
    add_vertex_normal(normal_array[o1*3], normal_array[o1*3+1], normal_array[o1*3+2])
    add_vertex_normal(normal_array[o2*3], normal_array[o2*3+1], normal_array[o2*3+2])
    add_vertex_normal(normal_array[o3*3], normal_array[o3*3+1], normal_array[o3*3+2])
    
    count += 3

def flush_and_render():
    global count, has_pos, has_normal
    if count == 0:
        return
    
    # Zero out unused buffer space
    for i in range(count * 3, MAX_COUNT * 3):
        pos_array[i] = 0
        normal_array[i] = 0
        
    # Bind buffers to WebGL
    # gl.bufferData(gl.ARRAY_BUFFER, pos_array, gl.DYNAMIC_DRAW)
    # gl.bufferData(gl.ARRAY_BUFFER, normal_array, gl.DYNAMIC_DRAW)
    
    # Enable attributes
    # if has_pos: gl.enableVertexAttribArray(0); gl.vertexAttribPointer(0, 3, gl.FLOAT, False, 0, 0)
    # if has_normal: gl.enableVertexAttribArray(1); gl.vertexAttribPointer(1, 3, gl.FLOAT, False, 0, 0)
    
    # Draw
    # gl.drawArrays(gl.TRIANGLES, 0, count)
    
    # Reset
    count = 0
    has_pos = False
    has_normal = False

def main_loop(grid, isovalue, blobs):
    # 1. Field Generation
    scalar_field = compute_scalar_field(grid, isovalue, blobs)
    
    # 2. Batch Init
    count = 0
    has_pos = False
    has_normal = False
    
    # 3. Grid Traversal
    for i in range(grid.width - 1):
        for j in range(grid.height - 1):
            for k in range(grid.depth - 1):
                # Get cube vertices
                v = [scalar_field[i][j][k], scalar_field[i+1][j][k], ...] # 8 vertices
                
                # Compute cube index
                cube_index = 0
                for n in range(8):
                    if v[n] >= isovalue:
                        cube_index |= (1 << n)
                
                # Lookup triangles
                triangles = lookup_marching_cubes_table(cube_index)
                
                for tri in triangles:
                    # Interpolate vertices for this triangle
                    verts = []
                    for edge_idx in tri:
                        edge = get_cube_edge(edge_idx)
                        p1, p2 = get_cube_vertex(edge[0]), get_cube_vertex(edge[1])
                        v1, v2 = scalar_field[p1], scalar_field[p2]
                        
                        # Interpolate position
                        t = (isovalue - v1) / (v2 - v1)
                        pos = lerp(p1, p2, t)
                        
                        # Interpolate normal (approximate gradient)
                        grad = compute_gradient(scalar_field, pos)
                        norm = normalize(grad)
                        
                        verts.append((pos, norm))
                    
                    # Add to batch
                    # Note: In the source, positions/normals are stored in arrays first, 
                    # then referenced by offset. Here we simulate the storage.
                    # We need to store them in pos_array/normal_array at 'count'
                    
                    # Store pos 1
                    pos_array[count*3] = verts[0].pos.x
                    pos_array[count*3+1] = verts[0].pos.y
                    pos_array[count*3+2] = verts[0].pos.z
                    normal_array[count*3] = verts[0].norm.x
                    normal_array[count*3+1] = verts[0].norm.y
                    normal_array[count*3+2] = verts[0].norm.z
                    count += 1
                    
                    # Store pos 2
                    pos_array[count*3] = verts[1].pos.x
                    pos_array[count*3+1] = verts[1].pos.y
                    pos_array[count*3+2] = verts[1].pos.z
                    normal_array[count*3] = verts[1].norm.x
                    normal_array[count*3+1] = verts[1].norm.y
                    normal_array[count*3+2] = verts[1].norm.z
                    count += 1
                    
                    # Store pos 3
                    pos_array[count*3] = verts[2].pos.x
                    pos_array[count*3+1] = verts[2].pos.y
                    pos_array[count*3+2] = verts[2].pos.z
                    normal_array[count*3] = verts[2].norm.x
                    normal_array[count*3+1] = verts[2].norm.y
                    normal_array[count*3+2] = verts[2].norm.z
                    count += 1
                    
                    # Check for flush
                    if count >= MAX_COUNT - 3:
                        flush_and_render()
    
    # Final flush
    flush_and_render()
```

### Parameters and Typical Values
*   `maxCount`: 4096 [S1, unofficial].
*   `prim`: `TRIANGLES` [S1, unofficial].
*   `primsize`: 3 [S1, unofficial].
*   Grid Resolution: 16³, 32³, 64³, 128³ [S1, unofficial].
*   Blob Counts: 1, 10, 50, 100 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Buffer Overflow**: The code explicitly checks `if count >= maxCount - 3` before adding a triangle. If this condition is met, `flushAndRender()` is called immediately to ensure space for the 3 vertices of the triangle [S1, unofficial].
*   **Precision**: Linear interpolation assumes the scalar field is smooth enough within the cell. Discontinuities or very steep gradients may cause artifacts.
*   **Boundary Conditions**: The grid traversal iterates from $0$ to $N-1$, assuming the scalar field is defined on the vertices of the grid cells. Boundary voxels must be handled correctly to avoid out-of-bounds access.

## Operating characteristics
*   **Time Complexity**: $O(N_x N_y N_z)$ for field generation and mesh extraction, where $N_x, N_y, N_z$ are the grid dimensions. The constant factor is high due to CPU-bound scalar field computation and JavaScript/TypeScript overhead.
*   **Memory Complexity**: $O(N_x N_y N_z)$ for the scalar field storage, plus $O(1)$ for the batch buffers (fixed size `maxCount`).
*   **Performance**: Limited by CPU performance. At high resolutions (e.g., 128³) or with many metaballs, the CPU becomes a bottleneck, leading to low frame rates [S1, unofficial].
*   **Strengths**: Simple implementation, serves as a good reference for understanding the algorithm, minimal GPU memory usage for geometry (streamed immediately).
*   **Limitations**: Poor scalability with resolution and object count. Not suitable for real-time applications with complex scenes or high-resolution volumes.
*   **When to Use**: Educational purposes, reference implementations, or simple demos where GPU acceleration is not available or desired for comparison.
*   **When Not to Use**: Production applications requiring high performance, large volumes, or real-time interaction with complex geometry.

## Background
The CPU Immediate Mode Marching Cubes method is inspired by Gregg Tavares' (Greggman) "Metaballs in JavaScript" demo [S1, unofficial]. It implements the classic Marching Cubes algorithm using precomputed lookup tables (e.g., Paul Bourke's tables) to determine edge intersections and triangle connectivity [S1, unofficial]. The "immediate mode" aspect refers to the direct rendering of geometry via WebGL draw calls without storing a persistent mesh buffer, a technique common in older graphics APIs and simple demos. This approach is part of the broader WebGL2-GPU-Marching-Cubes project by 3merillon, which also includes a GPU-accelerated version for comparison [S1, unofficial].

## Sources
[S1] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
