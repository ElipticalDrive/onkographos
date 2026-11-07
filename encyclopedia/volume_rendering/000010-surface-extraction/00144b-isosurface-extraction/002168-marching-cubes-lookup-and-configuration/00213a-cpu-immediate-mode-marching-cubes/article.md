# CPU Immediate Mode Marching Cubes

Also called: CPU Immediate Sim, Immediate Mode Marching Cubes, Legacy Marching Cubes

## Overview
CPU Immediate Mode Marching Cubes is a classic surface extraction method that computes a 3D scalar field on the CPU, determines isosurface intersections using lookup tables, and renders the resulting geometry directly to the GPU via immediate mode draw calls without storing a persistent mesh buffer [S1, unofficial]. It differs from standard Marching Cubes implementations by bypassing intermediate mesh storage, generating vertices and normals frame-by-frame for direct WebGL rendering, which serves as a lightweight reference implementation but suffers from CPU bottlenecks at high resolutions [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D grid of resolution $N \times N \times N$ (e.g., $16^3$ to $128^3$), a set of metaball sources (position, radius, strength), and a floor term [S1, unofficial].
- **Output:** A stream of triangle primitives rendered to the screen via WebGL [S1, unofficial].

**Data Structures:**
The implementation uses fixed-size typed arrays for batching geometry before sending it to the GPU [S1, unofficial].
- `posArray`: `Float32Array` of size `maxCount * 3`. Stores vertex positions $(x, y, z)$.
- `normalArray`: `Float32Array` of size `maxCount * 3`. Stores vertex normals $(nx, ny, nz)$.
- `uvArray`: `Float32Array` of size `maxCount * 2`. Stores UV coordinates.
- `colorArray`: `Float32Array` of size `maxCount * 4`. Stores vertex colors.
- `maxCount`: Integer constant, typically 4096, defining the maximum number of vertices per batch [S1, unofficial].
- `count`: Integer tracking the number of vertices currently in the batch [S1, unofficial].
- `prim`: Primitive type (`TRIANGLES`, `LINES`, or `POINTS`) [S1, unofficial].
- `primsize`: Integer (3, 2, or 1) corresponding to the number of vertices per primitive [S1, unofficial].

**Steps:**

1.  **Field Generation:**
    For each voxel $(i, j, k)$ in the $N \times N \times N$ grid, compute the scalar field value $V(i, j, k)$ by accumulating contributions from all metaballs and a floor term [S1, unofficial].
    $$ V(i, j, k) = \sum_{b \in Blobs} \frac{strength_b}{distance((i,j,k), pos_b)^2} + floor\_term $$
    Where $distance$ is the Euclidean distance between the voxel center and the metaball center [S1, unofficial].

2.  **Batch Initialization:**
    Set `prim` to the desired primitive type. Calculate `primsize` (3 for triangles, 2 for lines, 1 for points). Store the shader program reference. Reset `count` to 0. Reset boolean flags `hasPos`, `hasNormal`, `hasUV`, `hasColor` to `false` [S1, unofficial].

3.  **Grid Traversal and Extraction:**
    Iterate over every cell $(x, y, z)$ in the grid from $0$ to $N-2$ [S1, unofficial].
    a.  **Case Determination:** Determine the 8-bit case index by comparing the scalar values at the 8 vertices of the cell against the isovalue (typically 0 or 1 depending on normalization) [S1, unofficial].
    b.  **Edge Interpolation:** Use precomputed Marching Cubes lookup tables (e.g., Paul Bourke's tables) to identify which edges of the cube are intersected by the isosurface [S1, unofficial]. For each intersected edge, interpolate the vertex position and normal using linear interpolation between the vertex scalar values [S1, unofficial].
    c.  **Vertex Addition:** For each generated vertex, add its position and normal to the batch arrays:
        -   Store $x, y, z$ into `posArray` at indices `[count*3 + 0]`, `[count*3 + 1]`, `[count*3 + 2]`. Set `hasPos = true` [S1, unofficial].
        -   Store $nx, ny, nz$ into `normalArray` at indices `[count*3]`, `[count*3 + 1]`, `[count*3 + 2]`. Set `hasNormal = true` [S1, unofficial].
        -   Increment `count` [S1, unofficial].
    d.  **Triangle Generation:** For each triangle defined by the case index:
        -   Copy positions from `posArray` at offsets $o_1, o_2, o_3$ and normals from `normalArray` at offsets $o_1, o_2, o_3$ into `posArray` and `normalArray` starting at index `count*3` [S1, unofficial].
        -   Increment `count` by 3 [S1, unofficial].
        -   **Overflow Check:** If `count >= maxCount - 3`, flush the buffer (see Step 5) [S1, unofficial].

4.  **Flush and Render:**
    After traversing the entire grid:
    a.  If `count == 0`, return [S1, unofficial].
    b.  Zero out remaining entries in `posArray` and `normalArray` from index `count*3` to the end of the array to prevent garbage data [S1, unofficial].
    c.  Bind the WebGL buffers (`posBuf`, `normalBuf`, etc.) using `gl.bufferData` with `DYNAMIC_DRAW` hint, uploading the current state of the typed arrays [S1, unofficial].
    d.  Enable vertex attribute arrays for any enabled attributes (position, normal, UV, color) and set pointers using `gl.vertexAttribPointer` [S1, unofficial].
    e.  Call `gl.drawArrays` with `prim`, offset 0, and count [S1, unofficial].
    f.  Reset `count` to 0 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_immediate_marching_cubes(grid_resolution, metaballs, isovalue, max_count=4096):
    # 1. Field Generation
    # Compute scalar field for each voxel
    field = compute_scalar_field(grid_resolution, metaballs)
    
    # 2. Batch Initialization
    prim = "TRIANGLES"
    primsize = 3
    count = 0
    has_pos = False
    has_normal = False
    
    # Allocate arrays
    pos_array = [0.0] * (max_count * 3)
    normal_array = [0.0] * (max_count * 3)
    
    # 3. Grid Traversal
    for x in range(grid_resolution - 1):
        for y in range(grid_resolution - 1):
            for z in range(grid_resolution - 1):
                # Get vertex values for the current cell
                v = [field[x][y][z], field[x+1][y][z], field[x+1][y+1][z], field[x][y+1][z],
                     field[x][y][z+1], field[x+1][y][z+1], field[x+1][y+1][z+1], field[x][y+1][z+1]]
                
                # Determine case index
                case_index = 0
                for i in range(8):
                    if v[i] >= isovalue:
                        case_index |= (1 << i)
                
                # Get triangles for this case
                triangles = get_triangles_from_table(case_index)
                
                for tri in triangles:
                    # Generate 3 vertices for the triangle
                    for vertex_idx in tri:
                        # Interpolate position and normal for the edge intersection
                        pos, norm = interpolate_vertex(v, vertex_idx, grid_resolution)
                        
                        # Add to batch
                        idx = count * 3
                        pos_array[idx] = pos[0]
                        pos_array[idx+1] = pos[1]
                        pos_array[idx+2] = pos[2]
                        has_pos = True
                        
                        normal_array[idx] = norm[0]
                        normal_array[idx+1] = norm[1]
                        normal_array[idx+2] = norm[2]
                        has_normal = True
                        
                        count += 1
                        
                        # Check for buffer overflow
                        if count >= max_count - 3:
                            flush_and_render(pos_array, normal_array, count, prim, has_pos, has_normal)
                            count = 0
                            has_pos = False
                            has_normal = False

    # 4. Flush and Render
    if count > 0:
        flush_and_render(pos_array, normal_array, count, prim, has_pos, has_normal)

def flush_and_render(pos_array, normal_array, count, prim, has_pos, has_normal):
    # Zero out remaining entries
    for i in range(count * 3, len(pos_array)):
        pos_array[i] = 0.0
        normal_array[i] = 0.0
        
    # Upload to GPU
    gl.buffer_data(gl.ARRAY_BUFFER, pos_array, gl.DYNAMIC_DRAW)
    gl.buffer_data(gl.ARRAY_BUFFER, normal_array, gl.DYNAMIC_DRAW)
    
    # Enable attributes
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

**Parameters:**
- `max_count`: 4096 [S1, unofficial].
- `grid_resolution`: $16^3, 32^3, 64^3, 128^3$ [S1, unofficial].
- `isovalue`: Not explicitly given, but implied by standard Marching Cubes usage [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Buffer Overflow:** If `count >= maxCount - 3`, the buffer must be flushed immediately to ensure space for a full triangle (3 vertices) [S1, unofficial].
- **Precision:** Uses `Float32Array` for storage, which may introduce precision issues at very high resolutions or with small features [S1, unofficial].

## Subtopics

### [Streaming Marching Cubes](6190-streaming-marching-cubes.md)
Processes the grid in chunks to reduce memory usage, unlike the immediate mode which batches in fixed-size arrays.

### [GPU and Shader-Based Extraction](9036-gpu-and-shader-based-extraction.md)
Moves field generation and geometry processing to the GPU, whereas this method performs them on the CPU.

### [Preprocessed Marching Cubes](3765-preprocessed-marching-cubes.md)
Stores the resulting mesh in a persistent buffer for reuse, unlike the immediate mode which regenerates it every frame.

## Operating characteristics
- **Time Complexity:** $O(N^3)$ for field generation and mesh extraction, where $N$ is the grid resolution [S1, unofficial].
- **Memory Complexity:** $O(maxCount)$ for the batch buffers, plus $O(N^3)$ for the scalar field storage [S1, unofficial].
- **Measured Performance:** Limited by CPU bottlenecks at high resolutions or high blob counts [S1, unofficial].
- **Strengths:** Simple implementation, low memory overhead for the mesh buffer, useful as a reference [S1, unofficial].
- **Limitations:** Poor performance at scale due to CPU-GPU data transfer and lack of parallelism on the CPU [S1, unofficial].
- **When to use:** For small-scale demonstrations, educational purposes, or as a baseline for comparison [S1, unofficial].
- **When not to use:** For real-time applications with large volumes or high resolutions [S1, unofficial].

## Background
The CPU Immediate Mode Marching Cubes implementation is inspired by Gregg Tavares' (Greggman) "Metaballs in JavaScript" demo [S1, unofficial]. It utilizes Paul Bourke's Marching Cubes tables for geometry generation [S1, unofficial]. The method is part of the WebGL2-GPU-Marching-Cubes project by 3merillon, which provides both CPU and GPU implementations for comparison [S1, unofficial].

## Sources
[S1] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
