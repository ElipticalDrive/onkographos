# CPU Immediate Mode Marching Cubes

Also called: CPU Immediate Sim, Immediate Mode Marching Cubes, Legacy Marching Cubes

## Overview
CPU Immediate Mode Marching Cubes is a classic surface extraction method that generates mesh geometry on the CPU and renders it directly via immediate-mode draw calls without storing a persistent mesh buffer in GPU memory [S1, unofficial]. It solves the problem of visualizing scalar fields (such as metaballs) by computing the isosurface per frame, making it suitable for dynamic scenes where the underlying field changes frequently [S1, unofficial]. It differs from [Preprocessed Marching Cubes](3765-preprocessed-marching-cubes.md) by avoiding mesh caching and from [GPU Marching Cubes Terrain Pipeline with Indirect Rendering](3803-gpu-marching-cubes-terrain-pipeline-with-indirect-rendering.md) by performing all geometry generation on the CPU rather than the GPU [S1, unofficial].

## How it works
The method operates on a 3D regular grid (voxel grid) representing a scalar field. It computes the field values, determines surface intersections using the standard Marching Cubes algorithm, and streams the resulting triangles to the graphics API in batches.

### Inputs and Outputs
*   **Input**: A 3D scalar field defined on a grid of size $N_x \times N_y \times N_z$. The field is typically composed of contributions from multiple sources (e.g., metaballs) and a background term.
*   **Output**: A stream of triangle primitives rendered to the screen.

### Data Structures
The implementation maintains a fixed-size circular buffer or static array to hold vertex attributes for batching. The maximum number of vertices per batch is `maxCount` (typically 4096) [S1, unofficial].

**Buffers and Arrays**:
*   `posArray`: `Float32Array` of size `maxCount * 3`. Stores vertex positions $(x, y, z)$.
*   `normalArray`: `Float32Array` of size `maxCount * 3`. Stores vertex normals $(n_x, n_y, n_z)$.
*   `uvArray`: `Float32Array` of size `maxCount * 2`. Stores UV coordinates (optional).
*   `colorArray`: `Float32Array` of size `maxCount * 4`. Stores vertex colors (optional).
*   `count`: Integer tracking the number of vertices currently in the buffer.

**Marching Cubes Lookup**:
*   Standard 15-case lookup table mapping the 8-bit cube configuration (based on vertex signs relative to the isovalue) to a list of triangle edge indices [S1, unofficial].

### Steps

1.  **Field Generation**:
    For each voxel $(i, j, k)$ in the grid, compute the scalar value $V(i, j, k)$ by summing contributions from all active sources (e.g., metaballs) and a floor/background term.
    $$ V(i, j, k) = \sum_{m} \text{contribution}_m(i, j, k) + \text{floor\_term} $$
    [S1, unofficial]

2.  **Primitive Batching Initialization**:
    Initialize the batch state:
    *   Set primitive type `prim` (e.g., `TRIANGLES`).
    *   Set `primsize` (3 for triangles).
    *   Reset `count` to 0.
    *   Reset flags `hasPos`, `hasNormal`, `hasUV`, `hasColor` to false.
    [S1, unofficial]

3.  **Grid Traversal and Surface Extraction**:
    Iterate over every cell $(i, j, k)$ in the grid:
    a.  Retrieve the 8 scalar values at the cell's vertices.
    b.  Determine the cube configuration index by comparing each vertex value to the isovalue (typically 0.5 for normalized metaballs).
    c.  Use the lookup table to identify which edges of the cube are intersected by the isosurface.
    d.  For each intersected edge, interpolate the vertex position and normal.
        *   **Position Interpolation**: Linearly interpolate between the two vertices of the edge based on their scalar values.
        *   **Normal Calculation**: Compute the gradient of the scalar field at the intersection point (often approximated by the gradient of the metaball function) and normalize it.
    e.  Form triangles from the interpolated vertices according to the lookup table configuration.

4.  **Vertex Addition**:
    For each generated vertex:
    *   If position is available, store $x, y, z$ into `posArray` at indices `[count*3 + 0]`, `[count*3 + 1]`, `[count*3 + 2]`. Set `hasPos = true`.
    *   If normal is available, store $n_x, n_y, n_z$ into `normalArray` at indices `[count*3]`, `[count*3 + 1]`, `[count*3 + 2]`. Set `hasNormal = true`.
    *   Increment `count`.
    [S1, unofficial]

5.  **Triangle Addition and Buffer Flush Check**:
    For each generated triangle (3 vertices):
    *   Copy the positions and normals of the 3 vertices into `posArray` and `normalArray` starting at index `count * 3`.
    *   Increment `count` by 3.
    *   **Overflow Check**: If `count >= maxCount - 3`, call `draw()` to flush the current buffer before adding more vertices.
    [S1, unofficial]

6.  **Flush and Render**:
    After traversing the entire grid:
    *   If `count == 0`, return.
    *   Zero out remaining entries in `posArray` and `normalArray` from `count * 3` to the end to prevent garbage data.
    *   Bind the buffers to the WebGL context using `gl.bufferData` with `DYNAMIC_DRAW`.
    *   Enable vertex attribute arrays for any enabled attributes (position, normal, etc.) and set pointers using `gl.vertexAttribPointer`.
    *   Call `gl.drawArrays` with the primitive type, offset 0, and count.
    *   Reset `count` to 0.
    [S1, unofficial]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_COUNT = 4096
PRIM_SIZE = 3  # For triangles

# Data structures
pos_array = [0.0] * (MAX_COUNT * 3)
normal_array = [0.0] * (MAX_COUNT * 3)
count = 0
has_pos = False
has_normal = False

def add_vertex(x, y, z, nx, ny, nz):
    global count, has_pos, has_normal
    if count >= MAX_COUNT - 3:
        flush_and_draw()
    
    idx = count * 3
    pos_array[idx] = x
    pos_array[idx + 1] = y
    pos_array[idx + 2] = z
    has_pos = True
    
    normal_array[idx] = nx
    normal_array[idx + 1] = ny
    normal_array[idx + 2] = nz
    has_normal = True
    
    count += 1

def add_triangle(v1, v2, v3):
    # v1, v2, v3 are tuples of (x, y, z, nx, ny, nz)
    add_vertex(v1[0], v1[1], v1[2], v1[3], v1[4], v1[5])
    add_vertex(v2[0], v2[1], v2[2], v2[3], v2[4], v2[5])
    add_vertex(v3[0], v3[1], v3[2], v3[3], v3[4], v3[5])

def flush_and_draw():
    global count, has_pos, has_normal
    if count == 0:
        return
    
    # Zero out remaining buffer space
    for i in range(count * 3, MAX_COUNT * 3):
        pos_array[i] = 0.0
        normal_array[i] = 0.0
        
    # Bind buffers and set attributes
    # gl.bufferData(gl.ARRAY_BUFFER, pos_array, gl.DYNAMIC_DRAW)
    # gl.bufferData(gl.ARRAY_BUFFER, normal_array, gl.DYNAMIC_DRAW)
    # gl.enableVertexAttribArray(pos_attr)
    # gl.vertexAttribPointer(pos_attr, 3, gl.FLOAT, False, 0, 0)
    # if has_normal:
    #     gl.enableVertexAttribArray(normal_attr)
    #     gl.vertexAttribPointer(normal_attr, 3, gl.FLOAT, False, 0, 0)
    
    # gl.drawArrays(gl.TRIANGLES, 0, count)
    
    # Reset state
    count = 0
    has_pos = False
    has_normal = False

def compute_field(grid_size, blobs):
    # Compute scalar field values for all voxels
    # Returns 3D array of scalar values
    pass

def interpolate_edge(v1_val, v2_val, p1, p2, isovalue):
    # Linear interpolation of position and normal
    t = (isovalue - v1_val) / (v2_val - v1_val)
    pos = p1 + t * (p2 - p1)
    # Normal interpolation would require gradient calculation
    # For simplicity, assume normal is passed or computed separately
    return pos

def marching_cubes_cell(cell_vertices, cell_positions, isovalue):
    # Determine cube configuration
    config = 0
    for i in range(8):
        if cell_vertices[i] < isovalue:
            config |= (1 << i)
    
    # Lookup triangle indices
    triangles = lookup_table[config]
    
    generated_tris = []
    for tri in triangles:
        verts = []
        for edge_idx in tri:
            v1_idx, v2_idx = edge_map[edge_idx]
            v1_val = cell_vertices[v1_idx]
            v2_val = cell_vertices[v2_idx]
            p1 = cell_positions[v1_idx]
            p2 = cell_positions[v2_idx]
            
            # Interpolate position
            pos = interpolate_edge(v1_val, v2_val, p1, p2, isovalue)
            
            # Compute normal (gradient of field)
            # Normal = normalize(grad_field(pos))
            # This requires access to the field function or precomputed gradients
            normal = compute_normal(pos) 
            
            verts.append((pos[0], pos[1], pos[2], normal[0], normal[1], normal[2]))
        
        generated_tris.append(verts)
    
    return generated_tris

def render_frame(grid_size, blobs, isovalue):
    field = compute_field(grid_size, blobs)
    
    # Iterate over cells
    for i in range(grid_size[0] - 1):
        for j in range(grid_size[1] - 1):
            for k in range(grid_size[2] - 1):
                # Get 8 vertices of the cell
                cell_vertices = [
                    field[i][j][k], field[i+1][j][k], field[i+1][j+1][k], field[i][j+1][k],
                    field[i][j][k+1], field[i+1][j][k+1], field[i+1][j+1][k+1], field[i][j+1][k+1]
                ]
                cell_positions = [
                    (i, j, k), (i+1, j, k), (i+1, j+1, k), (i, j+1, k),
                    (i, j, k+1), (i+1, j, k+1), (i+1, j+1, k+1), (i, j+1, k+1)
                ]
                
                tris = marching_cubes_cell(cell_vertices, cell_positions, isovalue)
                for tri in tris:
                    add_triangle(tri[0], tri[1], tri[2])
    
    flush_and_draw()
```

### Parameters
*   `maxCount`: Buffer size in vertices. Default 4096 [S1, unofficial].
*   `isovalue`: Threshold for surface extraction. Typically 0.5 for normalized metaballs [S1, unofficial].
*   `grid_size`: Resolution of the voxel grid (e.g., $16^3, 32^3, 64^3, 128^3$) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Buffer Overflow**: If the number of vertices exceeds `maxCount`, the buffer must be flushed immediately. The check `count >= maxCount - 3` ensures there is space for at least one triangle [S1, unofficial].
*   **Precision**: Linear interpolation is used for vertex positions. For high-resolution grids or complex fields, this may result in visual artifacts if the field is not smooth [S1, unofficial].
*   **Performance**: The CPU bottleneck becomes significant at high resolutions or with many blobs, as all field computations and geometry generation occur on the CPU [S1, unofficial].

## Subtopics

### [Preprocessed Marching Cubes](3765-preprocessed-marching-cubes.md)
Caches the mesh to avoid recomputation, differing by storing geometry rather than generating it immediately.

### [GPU Marching Cubes Terrain Pipeline with Indirect Rendering](3803-gpu-marching-cubes-terrain-pipeline-with-indirect-rendering.md)
Offloads geometry generation to the GPU, differing by using GPU compute shaders instead of CPU loops.

### [Streaming Marching Cubes](6190-streaming-marching-cubes.md)
Processes the grid in chunks to reduce memory usage, differing by handling large grids that do not fit in memory.

## Operating characteristics
*   **Time Complexity**: $O(N^3)$ for a grid of size $N \times N \times N$, as every voxel is processed [S1, unofficial].
*   **Memory Complexity**: $O(N^3)$ for the scalar field storage, plus $O(1)$ for the immediate mode buffer (fixed size) [S1, unofficial].
*   **Performance**: Limited by CPU performance. At high resolutions (e.g., $128^3$) or with many blobs (e.g., 100), frame rates drop significantly due to CPU bottlenecks [S1, unofficial].
*   **Strengths**: Simple implementation, no persistent mesh memory overhead, suitable for dynamic fields where the mesh changes every frame [S1, unofficial].
*   **Limitations**: Poor performance at high resolutions, no mesh caching, limited by CPU throughput [S1, unofficial].
*   **When to Use**: Real-time visualization of dynamic scalar fields (e.g., metaballs) where simplicity and low memory footprint are prioritized over high resolution [S1, unofficial].
*   **When Not to Use**: Static scenes, high-resolution requirements, or when GPU acceleration is available and preferred [S1, unofficial].

## Background
The CPU Immediate Mode Marching Cubes method is inspired by Gregg Tavares' (Greggman) "Metaballs in JavaScript" demo, which demonstrated real-time metaball rendering using WebGL immediate mode [S1, unofficial]. It builds on the classic Marching Cubes algorithm by Paul Bourke, which provides the lookup tables for surface extraction [S1, unofficial]. The implementation is part of the `WebGL2-GPU-Marching-Cubes` project by 3merillon, which also includes a GPU-optimized version for comparison [S1, unofficial].

## Sources
[S1] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
