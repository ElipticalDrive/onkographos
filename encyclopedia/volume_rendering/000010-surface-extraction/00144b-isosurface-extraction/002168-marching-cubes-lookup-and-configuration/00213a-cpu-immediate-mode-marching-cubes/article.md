# CPU Immediate Mode Marching Cubes

Also called: CPU Immediate Sim, Immediate Mode Marching Cubes, Legacy Marching Cubes

## Overview
CPU Immediate Mode Marching Cubes is a classic CPU-based surface extraction method that generates mesh vertices and normals in immediate mode, rendering them directly via WebGL draw calls without storing a persistent mesh buffer [S1, unofficial]. It differs from its siblings by performing all field computation and geometry generation on the CPU every frame, bypassing persistent buffer management in favor of dynamic, batched uploads. This approach serves as a reference implementation for comparison but is limited by CPU bottlenecks at high resolutions or large blob counts [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D scalar field defined on a grid of voxels. The field is computed by accumulating contributions from metaballs (blobs) and a floor term for each voxel [S1, unofficial].
- **Output:** A stream of triangles (positions and normals) rendered immediately to the GPU.

**Data Structures:**
The implementation uses a fixed-size buffer system to manage vertex data before upload.
- `maxCount`: The maximum number of vertices per batch. Set to 4096 [S1, unofficial].
- `posArray`: A `Float32Array` of size `maxCount * 3` storing vertex positions (x, y, z) [S1, unofficial].
- `normalArray`: A `Float32Array` of size `maxCount * 3` storing vertex normals (x, y, z) [S1, unofficial].
- `uvArray`: A `Float32Array` of size `maxCount * 2` (optional) [S1, unofficial].
- `colorArray`: A `Float32Array` of size `maxCount * 4` (optional) [S1, unofficial].
- `posBuf`, `normalBuf`, `uvBuf`, `colorBuf`: WebGLBuffer objects corresponding to the data arrays [S1, unofficial].
- `count`: An integer tracking the current number of vertices in the batch [S1, unofficial].
- `hasPos`, `hasNormal`, `hasUV`, `hasColor`: Boolean flags indicating which attributes are populated [S1, unofficial].

**Steps:**
1. **Field Generation:** For each voxel in the 3D grid, compute the scalar value by summing contributions from all metaballs and a floor term [S1, unofficial].
2. **Marching Cubes Extraction:** For each cube in the grid, use precomputed lookup tables (e.g., Paul Bourke's tables) to determine which edges are intersected by the isosurface [S1, unofficial]. Interpolate vertex positions at these intersections and compute normals [S1, unofficial].
3. **Immediate Rendering Loop:**
   - Initialize primitive batching: Set primitive type (`TRIANGLES`, `LINES`, or `POINTS`), calculate `primsize` (3, 2, or 1), store program reference, reset `count` to 0, and reset attribute flags (`hasPos`, `hasNormal`, etc.) to false [S1, unofficial].
   - Add vertices: For each generated vertex, store coordinates into `posArray` at indices `[count*3 + 0]`, `[count*3 + 1]`, `[count*3 + 2]` and set `hasPos = true`. Similarly, store normals into `normalArray` and set `hasNormal = true` [S1, unofficial].
   - Add triangles: Copy positions and normals for three vertices (offsets `o1`, `o2`, `o3`) into the arrays starting at `count*3`. Increment `count` by 3. If `count >= maxCount - 3`, flush the buffer [S1, unofficial].
   - Advance vertex count: Increment `count`. If `count >= maxCount - 3` and `count` is divisible by `primsize`, flush the buffer [S1, unofficial].
   - Flush and Render: If `count` is 0, return. Otherwise, zero out remaining entries in `posArray` from `count*3` to length. Bind buffers for enabled attributes using `gl.bufferData` with `DYNAMIC_DRAW`. Enable vertex attribute arrays and set pointers. Call `gl.drawArrays` with the primitive type, offset 0, and `count`. Reset `count` to 0 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_COUNT = 4096
PRIM_SIZE = 3  # For TRIANGLES

# Data structures
pos_array = [0.0] * (MAX_COUNT * 3)
normal_array = [0.0] * (MAX_COUNT * 3)
uv_array = [0.0] * (MAX_COUNT * 2)
color_array = [0.0] * (MAX_COUNT * 4)

# State
count = 0
has_pos = False
has_normal = False
has_uv = False
has_color = False

def add_vertex(x, y, z, nx, ny, nz):
    global count, has_pos, has_normal
    # Store position
    pos_array[count * 3 + 0] = x
    pos_array[count * 3 + 1] = y
    pos_array[count * 3 + 2] = z
    has_pos = True
    
    # Store normal
    normal_array[count * 3 + 0] = nx
    normal_array[count * 3 + 1] = ny
    normal_array[count * 3 + 2] = nz
    has_normal = True
    
    count += 1
    if count >= MAX_COUNT - 3:
        flush_and_render()

def add_triangle(o1, o2, o3):
    global count
    # Copy positions
    for i in range(3):
        idx = (count + i) * 3
        pos_array[idx + 0] = pos_array[o1 * 3 + i]
        pos_array[idx + 1] = pos_array[o1 * 3 + i + 1] # Note: Source implies copying from existing pos array offsets
        pos_array[idx + 2] = pos_array[o1 * 3 + i + 2]
        # Repeat for o2, o3...
        # Actually, source says: "Copy positions from pos array at offsets o1, o2, o3 ... into posArray ... starting at index count*3"
        # This implies o1, o2, o3 are indices into the already populated pos_array for the triangle vertices.
        pass
    
    # Copy normals
    for i in range(3):
        idx = (count + i) * 3
        normal_array[idx + 0] = normal_array[o1 * 3 + i]
        normal_array[idx + 1] = normal_array[o1 * 3 + i + 1]
        normal_array[idx + 2] = normal_array[o1 * 3 + i + 2]
        # Repeat for o2, o3...
        pass
        
    count += 3
    if count >= MAX_COUNT - 3:
        flush_and_render()

def flush_and_render():
    global count, has_pos, has_normal, has_uv, has_color
    if count == 0:
        return
    
    # Zero out remaining entries
    for i in range(count * 3, MAX_COUNT * 3):
        pos_array[i] = 0.0
    # Zero out normal array similarly if needed, though not explicitly stated for normals, 
    # it's good practice. Source only mentions posArray.
    
    # Bind buffers
    # gl.bufferData(gl.ARRAY_BUFFER, pos_array, gl.DYNAMIC_DRAW)
    # gl.bufferData(gl.ARRAY_BUFFER, normal_array, gl.DYNAMIC_DRAW)
    # ... for uv and color if enabled
    
    # Enable attributes
    # if has_pos: gl.enableVertexAttribArray(pos_loc); gl.vertexAttribPointer(pos_loc, 3, gl.FLOAT, False, 0, 0)
    # if has_normal: gl.enableVertexAttribArray(norm_loc); gl.vertexAttribPointer(norm_loc, 3, gl.FLOAT, False, 0, 0)
    
    # Draw
    # gl.drawArrays(gl.TRIANGLES, 0, count)
    
    # Reset
    count = 0
    has_pos = False
    has_normal = False
    has_uv = False
    has_color = False

def main_loop():
    # 1. Field Generation
    # Compute scalar field for each voxel based on metaballs and floor
    
    # 2. Marching Cubes Extraction
    # For each cube:
    #   Determine edge intersections using lookup tables
    #   Interpolate vertices
    #   Generate triangles
    #   For each triangle, call add_triangle with vertex indices
    
    # 3. Flush remaining
    flush_and_render()
```

**Parameters:**
- `maxCount`: 4096 [S1, unofficial].
- `primsize`: 3 for triangles [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Buffer Overflow:** The code checks if `count >= maxCount - 3` before drawing to ensure space for a triangle (3 vertices). If exceeded, it flushes immediately [S1, unofficial].
- **Precision:** Floating-point precision in `Float32Array` may affect vertex accuracy at high resolutions [G].
- **Boundary Conditions:** Field generation must handle grid boundaries correctly, especially for the floor term [S1, unofficial].

## Subtopics

### [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md)
Converts signed distance fields to PLY meshes. Choose when persistent mesh storage is required.

### [Marching Cubes Triangulation](3682-marching-cubes-triangulation.md)
General triangulation logic. Choose for understanding the core algorithm.

### [Simplified Marching Cubes](3759-simplified-marching-cubes.md)
Reduces lookup table complexity. Choose for performance on simple fields.

### [Preprocessed Marching Cubes](3765-preprocessed-marching-cubes.md)
Precomputes data for faster extraction. Choose for static fields.

### [Sparse Marching Cubes](3825-sparse-marching-cubes.md)
Uses sparse data structures. Choose for large, sparse fields.

### [Marching Cubes with Narrow-Band Filtering](3826-marching-cubes-with-narrow-band-filtering.md)
Filters field data. Choose for noisy fields.

### [Modified Marching Cubes with Triangle Decimation](3903-modified-marching-cubes-with-triangle-decimation.md)
Reduces triangle count. Choose for low-poly outputs.

### [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md)
Moves computation to GPU. Choose for high performance.

### [Marching Cubes Vertex List Generation](4447-marching-cubes-vertex-list-generation.md)
Generates vertex lists. Choose for post-processing.

### [Lookup Table and Configuration](4978-lookup-table-and-configuration.md)
Configures MC tables. Choose for custom topology.

### [Marching Cubes with Disjoint Patch Decomposition](5004-marching-cubes-with-disjoint-patch-decomposition.md)
Decomposes into patches. Choose for parallel processing.

### [Transvoxel](6277-transvoxel.md)
Multi-resolution voxel rendering. Choose for dynamic level-of-detail.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel is processed once per frame [S1, unofficial].
- **Memory Complexity:** $O(1)$ for the fixed-size buffer, plus $O(N)$ for the scalar field if stored [S1, unofficial].
- **Measured Performance:** Limited by CPU bottlenecks at high resolutions (e.g., 128³) or large blob counts (e.g., 100 blobs) [S1, unofficial].
- **Scaling:** Performance degrades linearly with resolution and blob count due to CPU-bound field generation and geometry processing [S1, unofficial].
- **Strengths:** Simple implementation, serves as a reference, minimal GPU memory usage [S1, unofficial].
- **Limitations:** Poor performance at scale, CPU bottleneck, no persistent mesh storage [S1, unofficial].
- **When to Use:** Reference implementation, low-resolution scenes, educational purposes [S1, unofficial].
- **When Not to Use:** High-resolution scenes, real-time applications with many blobs, persistent mesh requirements [S1, unofficial].

## Background
The CPU Immediate Mode Marching Cubes method is inspired by Gregg Tavares' (Greggman) metaballs demo in JavaScript [S1, unofficial]. It uses classic Marching Cubes tables from Paul Bourke [S1, unofficial]. The implementation is part of the WebGL2-GPU-Marching-Cubes project by 3merillon [S1, unofficial].

## Sources
[S1] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
