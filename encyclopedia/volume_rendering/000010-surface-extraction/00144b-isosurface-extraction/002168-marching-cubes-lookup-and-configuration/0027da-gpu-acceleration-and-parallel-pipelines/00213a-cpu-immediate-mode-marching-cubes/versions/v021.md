# CPU Immediate Mode Marching Cubes

Also called: CPU Immediate Sim, Immediate Mode Marching Cubes, Legacy Marching Cubes

## Overview
CPU Immediate Mode Marching Cubes is a classic CPU-based implementation of the Marching Cubes algorithm that generates mesh vertices and normals in immediate mode, rendering them directly via WebGL draw calls without storing a persistent mesh buffer [S1, unofficial]. It differs from modern buffered or GPU-accelerated approaches by computing the scalar field and geometry on the CPU every frame and flushing small batches of primitives directly to the GPU, serving primarily as a reference implementation or for low-resolution comparisons [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D grid of voxels (resolution $N \times N \times N$), a set of metaball sources (position, radius, strength), and a floor term.
- **Output:** A stream of triangle primitives rendered to the screen.

**Data Structures:**
The implementation uses fixed-size typed arrays for batching geometry. The maximum batch size is defined as `maxCount = 4096` vertices [S1, unofficial].
- `posArray`: `Float32Array` of size `maxCount * 3` (x, y, z per vertex).
- `normalArray`: `Float32Array` of size `maxCount * 3` (nx, ny, nz per vertex).
- `uvArray`: `Float32Array` of size `maxCount * 2`.
- `colorArray`: `Float32Array` of size `maxCount * 4`.
- `count`: Integer tracking the number of vertices currently in the batch.
- `hasPos`, `hasNormal`, `hasUV`, `hasColor`: Boolean flags indicating which attributes are populated for the current batch.
- WebGL buffers (`posBuf`, `normalBuf`, etc.) are bound to these arrays using `gl.bufferData` with `DYNAMIC_DRAW` [S1, unofficial].

**Step-by-Step Method:**

1.  **Field Generation:**
    For each voxel $(i, j, k)$ in the grid, compute the scalar field value $V(i,j,k)$ by accumulating contributions from all metaballs and a floor term [S1, unofficial].
    $$ V(i,j,k) = \sum_{b \in \text{blobs}} \frac{S_b}{R_b^2 - D_{b,i,j,k}^2} + V_{\text{floor}} $$
    Where $S_b$ is the strength of blob $b$, $R_b$ is its radius, and $D_{b,i,j,k}$ is the distance from the voxel center to the blob center. The floor term is a constant value added to all voxels to define a base surface [S1, unofficial].

2.  **Marching Cubes Extraction:**
    Iterate over every cube in the grid. For each cube:
    a. Determine the cube index by comparing the scalar values at its 8 vertices against the isovalue (typically 0).
    b. Use precomputed lookup tables (e.g., Paul Bourke's tables) to identify which edges are intersected by the isosurface [S1, unofficial].
    c. Interpolate the exact intersection points on the edges using linear interpolation based on the vertex scalar values.
    d. Compute vertex normals at these intersection points, typically by averaging the gradients of the metaball fields at the intersection point.

3.  **Immediate Mode Rendering Loop:**
    The generated geometry is added to the batch arrays and flushed to the GPU in chunks.
    
    **Pseudocode:**

    ```python
    # Constants
    MAX_COUNT = 4096
    PRIM_SIZE = 3  # For triangles

    # State
    count = 0
    hasPos = False
    hasNormal = False
    posArray = Float32Array(MAX_COUNT * 3)
    normalArray = Float32Array(MAX_COUNT * 3)
    
    # WebGL setup (assumed initialized)
    gl = get_webgl_context()
    posBuf = gl.createBuffer()
    normalBuf = gl.createBuffer()

    def flush_batch():
        """Flush the current batch to the GPU."""
        nonlocal count, hasPos, hasNormal
        
        if count == 0:
            return

        # Zero out remaining entries to prevent garbage data
        for i in range(count * 3, MAX_COUNT * 3):
            posArray[i] = 0.0
        for i in range(count * 3, MAX_COUNT * 3):
            normalArray[i] = 0.0

        # Bind and upload position data if present
        if hasPos:
            gl.bindBuffer(gl.ARRAY_BUFFER, posBuf)
            gl.bufferData(gl.ARRAY_BUFFER, posArray, gl.DYNAMIC_DRAW)
            gl.enableVertexAttribArray(0) # Assuming location 0 for position
            gl.vertexAttribPointer(0, 3, gl.FLOAT, False, 0, 0)

        # Bind and upload normal data if present
        if hasNormal:
            gl.bindBuffer(gl.ARRAY_BUFFER, normalBuf)
            gl.bufferData(gl.ARRAY_BUFFER, normalArray, gl.DYNAMIC_DRAW)
            gl.enableVertexAttribArray(1) # Assuming location 1 for normal
            gl.vertexAttribPointer(1, 3, gl.FLOAT, False, 0, 0)

        # Draw the batch
        gl.drawArrays(gl.TRIANGLES, 0, count)

        # Reset state
        count = 0
        hasPos = False
        hasNormal = False

    def add_vertex_position(x, y, z):
        """Add a vertex position to the batch."""
        nonlocal count, hasPos
        if count >= MAX_COUNT - 3:
            flush_batch()
        
        idx = count * 3
        posArray[idx + 0] = x
        posArray[idx + 1] = y
        posArray[idx + 2] = z
        hasPos = True
        count += 1

    def add_vertex_normal(nx, ny, nz):
        """Add a vertex normal to the batch."""
        nonlocal count, hasNormal
        # Normals are stored in parallel with positions
        idx = count * 3
        normalArray[idx + 0] = nx
        normalArray[idx + 1] = ny
        normalArray[idx + 2] = nz
        hasNormal = True

    def add_triangle(o1, o2, o3):
        """
        Add a triangle defined by offsets into the pos/norm arrays.
        o1, o2, o3 are indices of vertices already added to the batch.
        """
        nonlocal count
        
        # Check if we have space for 3 vertices
        if count >= MAX_COUNT - 3:
            flush_batch()

        # Copy positions
        for k in range(3):
            posArray[count * 3 + k] = posArray[o1 * 3 + k]
            posArray[(count + 1) * 3 + k] = posArray[o2 * 3 + k]
            posArray[(count + 2) * 3 + k] = posArray[o3 * 3 + k]
            
            # Copy normals
            normalArray[count * 3 + k] = normalArray[o1 * 3 + k]
            normalArray[(count + 1) * 3 + k] = normalArray[o2 * 3 + k]
            normalArray[(count + 2) * 3 + k] = normalArray[o3 * 3 + k]

        count += 3

    def main_loop():
        """Execute one frame of CPU Immediate Mode Marching Cubes."""
        # 1. Compute Field
        field = compute_field(grid_resolution, blobs, floor_term)
        
        # 2. Extract Mesh
        for each cube in grid:
            cube_index = get_cube_index(field, cube_vertices)
            intersections = get_intersections(cube_index, cube_vertices, field)
            normals = get_normals(intersections, blobs)
            
            # 3. Add to Batch
            # Assuming intersections returns a list of vertex indices in the local cube
            # and we map them to global batch indices
            for triangle in get_triangles_from_table(cube_index):
                v1, v2, v3 = triangle
                add_vertex_position(*intersections[v1])
                add_vertex_normal(*normals[v1])
                add_vertex_position(*intersections[v2])
                add_vertex_normal(*normals[v2])
                add_vertex_position(*intersections[v3])
                add_vertex_normal(*normals[v3])
                add_triangle(count - 3, count - 2, count - 1)
        
        # 4. Flush remaining
        flush_batch()
    ```

**Parameters:**
- `maxCount`: 4096 vertices per batch [S1, unofficial].
- `prim`: Primitive type, typically `TRIANGLES` (size 3) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Buffer Overflow:** The code explicitly checks `if count >= maxCount - 3` before adding a triangle to ensure there is enough space in the array for 3 vertices. If this condition is met, `flush_batch()` is called immediately to free up space [S1, unofficial].
- **Precision:** Linear interpolation is used for vertex positions. Gradient calculation for normals may suffer from precision issues if metaball radii are very small or distances are large [S1, unofficial].
- **Boundary Conditions:** The grid boundaries must be handled such that voxels outside the defined domain do not contribute to the field or are clamped appropriately.

## Operating characteristics
- **Time Complexity:** $O(N^3 \cdot B)$, where $N$ is the grid resolution and $B$ is the number of metaballs. Each voxel requires evaluating all blobs.
- **Memory Complexity:** $O(N^3)$ for the scalar field storage, plus $O(1)$ for the immediate mode buffers (fixed size 4096 vertices) [S1, unofficial].
- **Performance:** Limited by CPU bottlenecks at high resolutions or high blob counts. It is significantly slower than GPU-based implementations because field computation and geometry generation occur on the CPU every frame [S1, unofficial].
- **Strengths:** Simple implementation, no persistent mesh data structures required, useful for debugging and reference.
- **Limitations:** Poor scalability, high CPU usage, limited by the fixed batch size and frequent draw calls if not batched effectively.

## Background
The CPU Immediate Mode Marching Cubes implementation is inspired by Gregg Tavares' (Greggman) "Metaballs in JavaScript" demo, which demonstrated real-time metaball rendering using WebGL immediate mode [S1, unofficial]. It builds upon the classic Marching Cubes algorithm described by Paul Bourke, using his precomputed lookup tables for edge intersections [S1, unofficial]. This approach was popular in early WebGL demonstrations to show that complex geometry could be generated on the fly without complex buffering strategies.

## Sources
[S1] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
