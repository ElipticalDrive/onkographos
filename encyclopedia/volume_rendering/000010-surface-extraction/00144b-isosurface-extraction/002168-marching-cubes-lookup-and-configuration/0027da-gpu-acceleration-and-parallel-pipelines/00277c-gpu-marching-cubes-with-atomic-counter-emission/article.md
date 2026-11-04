# GPU Marching Cubes with Atomic Counter Emission

Also called: atomic-counter MC, indirect-draw MC, r=mesh

## Overview
This method is a real-time isosurface extraction technique that maps a 3D scalar grid to a triangle mesh using one compute thread per grid cell, avoiding prefix sums by reserving output slots via atomic addition on indirect draw arguments [S1, unofficial]. It differs from sibling methods like [GPU Marching Cubes with Histogram Pyramid](2317-gpu-marching-cubes-with-histogram-pyramid.md) by using atomic counters instead of hierarchical reduction, and from [GPU Marching Cubes Terrain Pipeline with Indirect Rendering](3803-gpu-marching-cubes-terrain-pipeline-with-indirect-rendering.md) by integrating the counter directly into the draw argument buffer structure [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tent-blurred density grid of dimensions $D_x \times D_y \times D_z$ and an isosurface threshold parameter $\text{iso}$ [S1, unofficial].
*   **Output:** A triangle mesh rendered via `drawIndirect`, consisting of vertex positions and gradient normals stored in a capped pool buffer [S1, unofficial].

**Data Structures:**
1.  **Indirect Draw Args Buffer:** A buffer containing standard indirect draw parameters. Word 0-3 contain standard arguments (e.g., vertex count, instance count). Word 4 stores an overflow flag [S1, unofficial].
2.  **Vertex Pool:** A fixed-size buffer of capacity $N_{pool} = 1.5 \times 10^6$ vertices. Each vertex entry stores a 3D position and a 3D gradient normal (48 MB total for the default capacity) [S1, unofficial].
3.  **Triangle Table:** A lookup table generated at load time, mapping the 8-bit cube configuration index to a set of cut edges and triangles. It is constructed by generating marching squares faces per cube face, forming closed loops, and applying fan triangulation with a rotation-invariant ambiguity rule [S1, unofficial].

**Numbered Steps:**

1.  **Dispatch:** Launch one compute thread for each cell in the $D_x \times D_y \times D_z$ grid [S1, unofficial].
2.  **Classification:** For the current cell, sample the scalar values at its 8 corner vertices. Classify the cell by comparing these values against the isosurface threshold $\text{iso}$ to determine the 8-bit configuration index $I \in [0, 255]$ [S1, unofficial].
    *   *Edge Case:* If a lattice sample is out-of-range (e.g., at the boundary), treat its density as 0 to cap the isosurface against the walls [S1, unofficial].
3.  **Lookup:** Use index $I$ to retrieve the set of cut edges from the Triangle Table [S1, unofficial].
4.  **Slot Reservation:** Perform an atomic addition (`atomicAdd`) on the vertex count field within the Indirect Draw Args Buffer to reserve a contiguous block of slots in the Vertex Pool. The number of slots reserved equals $3 \times N_{triangles}$, where $N_{triangles}$ is the number of triangles generated for index $I$ [S1, unofficial].
5.  **Vertex Emission:** For each cut edge identified in the lookup:
    *   Compute the interpolation parameter $\lambda$ along the edge connecting vertices with scalar values $v_1$ and $v_2$:
        $$
        \lambda = \frac{\text{iso} - v_1}{v_2 - v_1}
        $$
    *   Calculate the vertex position $\mathbf{p}$:
        $$
        \mathbf{p} = \text{lerp}(\mathbf{pos}_1, \mathbf{pos}_2, \lambda)
        $$
    *   Calculate the normal $\mathbf{n}$ by interpolating the gradients $\mathbf{g}_1$ and $\mathbf{g}_2$ at the edge endpoints:
        $$
        \mathbf{n} = \text{normalize}(\text{lerp}(\mathbf{g}_1, \mathbf{g}_2, \lambda))
        $$
    *   Write $\mathbf{p}$ and $\mathbf{n}$ into the reserved slots in the Vertex Pool [S1, unofficial].
    *   *Optimization:* Cache interpolated position and normal per unique cut edge to avoid recomputation per triangle corner [S1, unofficial].
6.  **Finalization:** Launch a single-thread kernel to clamp the total vertex count in the Indirect Draw Args Buffer to the pool capacity $N_{pool}$. If the pre-clamp count exceeds $N_{pool}$, set the overflow flag in Word 4 of the Indirect Draw Args Buffer [S1, unofficial].
7.  **Rendering:** Execute `drawIndirect` using the Indirect Draw Args Buffer. The renderer pulls vertices from the Vertex Pool [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader: One thread per grid cell (thread_id = x + y*Dx + z*Dx*Dy)
def mc_atomic_counter_kernel(grid, iso, indirect_args, vertex_pool, table, pool_cap):
    # 1. Get cell coordinates
    z = thread_id // (Dx * Dy)
    y = (thread_id % (Dx * Dy)) // Dx
    x = thread_id % Dx
    
    # 2. Sample 8 corners
    # Handle out-of-range samples as density 0
    corners = []
    for dx in [0, 1]:
        for dy in [0, 1]:
            for dz in [0, 1]:
                cx, cy, cz = x + dx, y + dy, z + dz
                if 0 <= cx < Dx and 0 <= cy < Dy and 0 <= cz < Dz:
                    corners.append(grid[cx, cy, cz])
                else:
                    corners.append(0.0) # Out-of-range cap
    
    # 3. Classify
    index = 0
    for i, val in enumerate(corners):
        if val < iso:
            index |= (1 << i)
            
    # 4. Lookup edges
    cut_edges = table[index]
    
    # 5. Reserve slots
    # atomicAdd returns the starting index in the pool for this cell's vertices
    start_idx = atomic_add(indirect_args.vertex_count, len(cut_edges) * 3)
    
    # 6. Emit vertices
    current_vertex_idx = start_idx
    for edge in cut_edges:
        v1_idx, v2_idx = edge.vertices
        pos1 = grid_positions[corners[v1_idx]] # Assuming positions are known or derived from grid coords
        pos2 = grid_positions[corners[v2_idx]]
        val1 = corners[v1_idx]
        val2 = corners[v2_idx]
        
        # Interpolate position
        lam = (iso - val1) / (val2 - val1)
        pos = lerp(pos1, pos2, lam)
        
        # Interpolate normal (gradient)
        # Note: Gradients must be available or computed from grid
        grad1 = grid_gradients[corners[v1_idx]]
        grad2 = grid_gradients[corners[v2_idx]]
        normal = normalize(lerp(grad1, grad2, lam))
        
        vertex_pool[current_vertex_idx].position = pos
        vertex_pool[current_vertex_idx].normal = normal
        current_vertex_idx += 1

# Finalization Kernel: One thread
def mc_finalize_kernel(indirect_args, pool_cap):
    count = indirect_args.vertex_count
    if count > pool_cap:
        indirect_args.vertex_count = pool_cap
        indirect_args.overflow_flag = 1
    else:
        indirect_args.overflow_flag = 0
```

**Parameters:**
*   **Vertex Pool Size:** 1.5 million vertices (1.5M) [S1, unofficial].
*   **Isosurface Threshold:** Controlled by the `?iso=` parameter [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Overflow:** If the vertex count exceeds the pool cap, the finalize kernel clamps the count to the cap. The overflow flag is set in Word 4 of the indirect draw args, not in `firstInstance`, because non-zero `firstInstance` no-ops the indirect draw without the `indirect-first-instance` feature [S1, unofficial].
*   **Wall Capping:** Lattice 0 is treated as outside density (0) to cap the isosurface on all six walls. Blur bleed outside the walls (cells $2 \dots \text{GRID}-2$) is discarded [S1, unofficial].
*   **Triangle Table Generation:** The table is generated at load time from marching squares faces to closed loops to fan triangulation, ensuring rotation-invariant ambiguity rules and crack-free results, rather than transcribing the classic 256x16 constant table [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(D_x D_y D_z)$ for the main compute pass, as one thread processes each cell. The finalization is $O(1)$.
*   **Memory Complexity:** $O(N_{pool})$ for the vertex pool and $O(1)$ for the triangle table (256 entries). The vertex pool size is fixed at 1.5M vertices [S1, unofficial].
*   **Performance:** 4.12 ms/frame at $128^3$ resolution and $384^2$ screen size on M-series hardware [S1, unofficial].
*   **Strengths:** Avoids the overhead of prefix sum/histogram calculations. Simple implementation using standard atomic operations.
*   **Limitations:** Requires a fixed-size vertex pool; overflow handling requires careful buffer management. Atomic contention may occur in highly complex scenes with many cut cells [S1, unofficial].
*   **When to use:** Real-time applications where prefix sum overhead is prohibitive and a fixed vertex budget is acceptable.
*   **When not to use:** Scenes requiring guaranteed exact vertex counts without clamping, or where atomic contention is expected to be a bottleneck.

## Background
This method was introduced in the "waterfall" WebGL2 fluid simulation project as the `r=mesh` renderer, specifically for WebGPU support, with a fallback to screen-space fluid (ssf) for WebGL2 [S1, unofficial]. It was developed by nikolai3d and merged into the main branch in July 2026 [S1, unofficial]. The approach was designed to provide a real-time isosurface meshing technique that integrates seamlessly with raytraced scene backgrounds, using shading techniques like Fresnel reflection and Beer-Lambert absorption [S1, unofficial].

## Sources
[S1] Renderer: marching-cubes isosurface mesh (r=mesh, WebGPU), https://github.com/nikolai3d/waterfall/pull/17, fetched 2026-10-27 (unofficial)
