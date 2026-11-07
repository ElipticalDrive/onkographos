# CPU Immediate Mode Marching Cubes

Also called: CPU Immediate Sim, Immediate Mode Marching Cubes, Legacy Marching Cubes

## Overview
CPU Immediate Mode Marching Cubes is a classic CPU-based surface extraction method that generates mesh vertices and normals in immediate mode, rendering them directly via WebGL draw calls without storing a persistent mesh buffer [S1, unofficial]. It differs from its siblings by performing the entire field generation and geometry computation on the CPU and streaming the resulting primitives to the GPU frame-by-frame, serving primarily as a reference implementation for comparison against GPU-accelerated variants [S1, unofficial].

## How it works
The method operates on a discretized 3D grid (volume) where each voxel contains a scalar value. It computes the scalar field, determines surface intersections using the standard Marching Cubes algorithm, and streams the resulting triangles to the graphics pipeline.

### Inputs and Outputs
*   **Input**: A 3D grid of scalar values $V_{i,j,k}$ for $0 \le i,j,k < N$, where $N$ is the resolution. The scalar field is typically generated from metaballs or a floor term.
*   **Output**: A stream of triangle primitives (vertices and normals) rendered to the screen.

### Data Structures
The implementation uses fixed-size typed arrays to store vertex attributes for batching. The maximum number of vertices per batch is `maxCount` = 4096 [S1, unofficial].

*   `posArray`: `Float32Array` of size `maxCount * 3`. Stores vertex positions $(x, y, z)$.
*   `normalArray`: `Float32Array` of size `maxCount * 3`. Stores vertex normals $(n_x, n_y, n_z)$.
*   `uvArray`: `Float32Array` of size `maxCount * 2`. (Optional, not always populated in basic metaball demos).
*   `colorArray`: `Float32Array` of size `maxCount * 4`. (Optional).
*   `count`: Integer tracking the number of vertices currently in the buffer.
*   `hasPos`, `hasNormal`, `hasUV`, `hasColor`: Boolean flags indicating which attributes are valid for the current batch.

### Algorithm Steps

1.  **Field Generation**: For each voxel $(i, j, k)$ in the grid, compute the scalar value $S_{i,j,k}$ by accumulating contributions from all metaballs and a floor term [S1, unofficial].
    $$ S_{i,j,k} = \sum_{m} \frac{r_m^2}{d_m^2} + S_{floor} $$
    Where $d_m$ is the distance from the voxel center to the center of metaball $m$, and $r_m$ is the radius of metaball $m$. $S_{floor}$ is a constant background value.

2.  **Primitive Batching Initialization**:
    *   Set primitive type `prim` to `TRIANGLES`.
    *   Set `primsize` to 3.
    *   Reset `count` to 0.
    *   Reset flags `hasPos`, `hasNormal`, `hasUV`, `hasColor` to `false`.

3.  **Grid Traversal and Surface Extraction**: Iterate over each cell $(i, j, k)$ in the grid from $(0,0,0)$ to $(N-2, N-2, N-2)$:
    *   Retrieve the 8 corner scalar values $v_0 \dots v_7$ of the cell.
    *   Determine the **cube index** (case number) by comparing each $v_n$ against the isovalue $I$. If $v_n \ge I$, set bit $n$ to 1.
    *   Use the precomputed **Marching Cubes Lookup Table** (e.g., Paul Bourke's tables) to determine:
        *   Which edges are intersected.
        *   Which triangles to generate (up to 5 triangles per case).
    *   For each intersected edge, calculate the intersection point $P$ and normal $N$ via linear interpolation:
        $$ P = V_a + \frac{I - v_a}{v_b - v_a} (V_b - V_a) $$
        $$ N = \text{normalize}(\nabla S(P)) $$
        Where $V_a, V_b$ are the voxel centers of the edge endpoints, and $v_a, v_b$ are their scalar values. The gradient $\nabla S$ is approximated using central differences or analytical derivatives of the metaball function.

4.  **Vertex Addition**:
    *   **Add Vertex Position**: Store $P.x, P.y, P.z$ into `posArray` at indices `[count*3 + 0]`, `[count*3 + 1]`, `[count*3 + 2]`. Set `hasPos = true`.
    *   **Add Vertex Normal**: Store $N.x, N.y, N.z$ into `normalArray` at indices `[count*3]`, `[count*3 + 1]`, `[count*3 + 2]`. Set `hasNormal = true`.
    *   **Advance Vertex Count**: Increment `count`.

5.  **Triangle Generation and Buffer Management**:
    *   For each triangle defined by the lookup table (vertices $o_1, o_2, o_3$):
        *   Check if `count >= maxCount - 3`. If true, call `draw()` to flush the buffer before adding new vertices.
        *   Copy positions from `posArray` at offsets $o_1, o_2, o_3$ into `posArray` starting at `count*3`.
        *   Copy normals from `normalArray` at offsets $o_1, o_2, o_3$ into `normalArray` starting at `count*3`.
        *   Increment `count` by 3.

6.  **Flush and Render**:
    *   If `count == 0`, return.
    *   Zero out remaining entries in `posArray` and `normalArray` from `count*3` to `maxCount*3` to prevent garbage data.
    *   Bind `posBuf` and `normalBuf` using `gl.bufferData` with `DYNAMIC_DRAW` hint, copying data from `posArray` and `normalArray`.
    *   Enable vertex attribute arrays for position and normal.
    *   Set pointers: `gl.vertexAttribPointer` for position (3 components) and normal (3 components).
    *   Call `gl.drawArrays(gl.TRIANGLES, 0, count)`.
    *   Reset `count` to 0.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_immediate_marching_cubes(grid_resolution, metaballs, isovalue, gl_context):
    max_count = 4096
    pos_array = Float32Array(max_count * 3)
    normal_array = Float32Array(max_count * 3)
    count = 0
    has_pos = False
    has_normal = False

    # 1. Field Generation
    scalar_field = compute_scalar_field(grid_resolution, metaballs)

    # 2. Initialize Batching
    prim_type = gl_context.TRIANGLES
    prim_size = 3
    
    # 3. Traverse Grid
    for i in range(grid_resolution - 1):
        for j in range(grid_resolution - 1):
            for k in range(grid_resolution - 1):
                # Get 8 corner values
                corners = [
                    scalar_field[i, j, k], scalar_field[i+1, j, k],
                    scalar_field[i+1, j, k+1], scalar_field[i, j, k+1],
                    scalar_field[i, j+1, k], scalar_field[i+1, j+1, k],
                    scalar_field[i+1, j+1, k+1], scalar_field[i, j+1, k+1]
                ]
                
                # Determine cube index
                cube_index = 0
                for bit in range(8):
                    if corners[bit] >= isovalue:
                        cube_index |= (1 << bit)
                
                # Get triangles for this case
                triangles = get_triangles_from_lookup_table(cube_index)
                
                for tri in triangles:
                    # Ensure space for 3 vertices
                    if count >= max_count - 3:
                        flush_and_render(gl_context, pos_array, normal_array, count, has_pos, has_normal)
                        count = 0
                        has_pos = False
                        has_normal = False

                    # Process 3 vertices of the triangle
                    for vertex_idx in tri:
                        # Interpolate vertex position and normal
                        pos, normal = interpolate_vertex(corners, cube_index, vertex_idx, grid_resolution)
                        
                        # Add position
                        idx = count * 3
                        pos_array[idx] = pos[0]
                        pos_array[idx+1] = pos[1]
                        pos_array[idx+2] = pos[2]
                        has_pos = True
                        
                        # Add normal
                        normal_array[idx] = normal[0]
                        normal_array[idx+1] = normal[1]
                        normal_array[idx+2] = normal[2]
                        has_normal = True
                        
                        count += 1

    # Final flush
    if count > 0:
        flush_and_render(gl_context, pos_array, normal_array, count, has_pos, has_normal)

def flush_and_render(gl, pos_arr, norm_arr, count, has_pos, has_normal):
    if count == 0:
        return
    
    # Zero out unused buffer space
    for i in range(count * 3, len(pos_arr)):
        pos_arr[i] = 0
        norm_arr[i] = 0
        
    # Upload to GPU
    gl.buffer_data(gl.ARRAY_BUFFER, pos_arr, gl.DYNAMIC_DRAW)
    gl.buffer_data(gl.ARRAY_BUFFER, norm_arr, gl.DYNAMIC_DRAW)
    
    # Set up attributes
    if has_pos:
        gl.enable_vertex_attrib_array(0)
        gl.vertex_attrib_pointer(0, 3, gl.FLOAT, False, 0, 0)
    if has_normal:
        gl.enable_vertex_attrib_array(1)
        gl.vertex_attrib_pointer(1, 3, gl.FLOAT, False, 0, 0)
        
    # Draw
    gl.draw_arrays(gl.TRIANGLES, 0, count)
    
    # Reset
    count = 0
```

### Parameters and Edge Cases
*   **maxCount**: Fixed at 4096 in the reference implementation [S1, unofficial].
*   **Isovale**: The threshold for surface extraction. Not given in the sources.
*   **Grid Resolution**: Typical values are $16^3, 32^3, 64^3, 128^3$ [S1, unofficial].
*   **Pitfall - Buffer Overflow**: The code explicitly checks `if count >= maxCount - 3` before adding a triangle to ensure there is space for 3 vertices. If this condition is met, the buffer is flushed immediately [S1, unofficial].
*   **Pitfall - Precision**: Linear interpolation is used for vertex positions. For high-resolution grids or large metaballs, this may result in slight inaccuracies compared to higher-order interpolation methods [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N^3 \cdot M)$, where $N$ is the grid resolution and $M$ is the number of metaballs. Each voxel requires $M$ distance calculations for field generation, and each cell requires constant-time lookup and interpolation for surface extraction.
*   **Memory Complexity**: $O(N^3)$ for the scalar field storage, plus $O(1)$ for the immediate mode buffers (fixed size 4096 vertices) [S1, unofficial].
*   **Performance**: Limited by CPU bottlenecks at high resolutions or high blob counts. It is significantly slower than GPU-based implementations because field generation and geometry computation are done on the CPU, and data is transferred to the GPU every frame [S1, unofficial].
*   **Strengths**: Simple implementation, serves as a good reference for understanding the algorithm, minimal CPU-GPU transfer overhead compared to storing large persistent meshes (only small uniform and texture updates per frame) [S1, unofficial].
*   **Limitations**: Poor scalability. Not suitable for real-time applications with high-resolution volumes or complex scenes.
*   **When to use**: For educational purposes, reference implementations, or simple demos where GPU acceleration is not available or desired for comparison [S1, unofficial].
*   **When not to use**: For production applications requiring high performance, large volumes, or real-time interaction with complex geometry.

## Background
The CPU Immediate Mode Marching Cubes method is inspired by Gregg Tavares (Greggman)'s "Metaballs in JavaScript" demo [S1, unofficial]. It implements the classic Marching Cubes algorithm using precomputed tables (e.g., Paul Bourke's tables) to determine edge intersections and triangle configurations [S1, unofficial]. The method was popularized in the context of WebGL demos to demonstrate real-time rendering capabilities on the CPU before GPU-based optimizations became standard.

## Sources
[S1] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
