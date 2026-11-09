# GPU Marching Cubes with Atomic Counter Emission

Also called: atomic-counter MC, indirect-draw MC, r=mesh

## Overview
This method is a real-time GPU-based isosurface extraction technique that eliminates the need for prefix sums or histogram pyramids by using atomic operations to reserve vertex slots directly within indirect draw arguments [S1, unofficial]. It assigns one compute thread per grid cell to classify density, emit interpolated vertices into a capped pool buffer, and update a global vertex counter atomically, enabling immediate rendering via `drawIndirect` [S1, unofficial]. Unlike sibling methods that rely on pre-computed output sizes or hierarchical data structures, this approach dynamically manages memory allocation on the GPU with minimal synchronization overhead, making it suitable for real-time fluid simulation visualizations where mesh topology changes rapidly [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   **Density Grid:** A 3D scalar field representing fluid density, typically tent-blurred to smooth the surface [S1, unofficial].
*   **Isosurface Threshold ($I$):** A scalar value (e.g., $0.2$) defining the surface level, controlled by the `?iso=` parameter [S1, unofficial].
*   **Grid Dimensions:** Defined by $GRID$, where the density array has dimensions $GRID \times GRID \times GRID$ [S1, unofficial].

**Outputs:**
*   **Mesh Vertices:** A buffer of vertex positions and gradient normals, capped at a maximum size [S1, unofficial].
*   **Draw Arguments:** An indirect draw buffer containing the vertex count and instance parameters, updated atomically [S1, unofficial].
*   **Overflow Flag:** A boolean indicator stored in the indirect draw arguments if the vertex pool is exceeded [S1, unofficial].

### Data Structures

**1. Indirect Draw Arguments Buffer**
A buffer containing the parameters for `drawIndirect`. It is structured as an array of 32-bit integers (words).
*   **Word 0:** Vertex count (updated atomically).
*   **Word 1:** Instance count (typically 1).
*   **Word 2:** First vertex offset (typically 0).
*   **Word 3:** First instance offset (typically 0).
*   **Word 4:** Overflow flag. Set to 1 if the pre-clamp vertex count exceeds the pool cap [S1, unofficial].

**2. Vertex Pool Buffer**
A linear buffer storing vertex data.
*   **Capacity:** 1.5 million vertices ($1.5 \times 10^6$) [S1, unofficial].
*   **Size:** 48 MB [S1, unofficial].
*   **Layout:** Each vertex consists of a 3D position (float3) and a 3D normal (float3), totaling 6 floats (24 bytes) per vertex [S1, unofficial].

**3. Triangle Table**
A dynamically generated lookup table mapping case indices to triangle edge indices.
*   **Generation:** Created at load time using a script that derives triangles from marching squares faces, forms closed loops, and applies fan triangulation with a rotation-invariant ambiguity rule [S1, unofficial].
*   **Validation:** Asserted against the cut-edge set of all 256 cases to ensure crack-free geometry [S1, unofficial].
*   **Structure:** Similar to the standard `triangleTable`, mapping each case index to a sequence of edge indices terminated by -1, with a maximum of 5 triangles per case [S1, unofficial].

### Numbered Steps

1.  **Dispatch Compute Threads:** Launch a compute shader dispatch with one thread group per grid cell, resulting in $GRID^3$ threads [S1, unofficial].
2.  **Classify Cell:** Each thread reads the density values at the 8 corners of its assigned voxel from the tent-blurred density grid [S1, unofficial].
3.  **Compute Case Index:** Calculate the 8-bit case index $C$ by comparing each corner density $V_i$ against the isosurface threshold $I$. If $V_i \ge I$, the $i$-th bit is set; otherwise, it is 0 [S1, unofficial].
4.  **Lookup Edges:** Use $C$ to index into the generated triangle table to identify the intersected edges [S1, unofficial].
5.  **Reserve Slots:** Perform an atomic addition (`atomicAdd`) on the vertex count in the Indirect Draw Arguments buffer (Word 0) to reserve a contiguous block of vertex slots for the new triangles [S1, unofficial].
6.  **Compute Vertices:** For each intersected edge identified in Step 4:
    *   Interpolate the vertex position along the edge using trilinear interpolation of the corner densities [S1, unofficial].
    *   Compute the gradient normal at the interpolated position [S1, unofficial].
    *   Cache the position and normal if the edge has been processed by another thread (to avoid duplicate vertices) [S1, unofficial].
7.  **Write Vertices:** Write the computed vertex positions and normals into the Vertex Pool Buffer at the indices reserved in Step 5 [S1, unofficial].
8.  **Finalize Count:** Dispatch a single-thread kernel to clamp the vertex count in the Indirect Draw Arguments buffer to the pool cap (1.5 million) [S1, unofficial].
9.  **Check Overflow:** If the pre-clamp vertex count exceeded the cap, set the Overflow Flag in Word 4 of the Indirect Draw Arguments buffer [S1, unofficial].
10. **Render:** Issue a `drawIndirect` call using the updated arguments. If the Overflow Flag is set, the draw is handled appropriately (e.g., clamped to valid triangles) to prevent rendering garbage [S1, unofficial].
11. **Shade:** Render the mesh with depth testing against the raytraced scene background. Apply shading using Fresnel reflection over the analytic scene along one refracted ray, Beer-Lambert absorption using 6 interior density taps, and momentum foam effects [S1, unofficial].

### Formulas

**Case Index Calculation:**
$$ C = \sum_{i=0}^{7} b_i 2^i $$
where $b_i = 1$ if $V_i \ge I$, else $0$, and $V_i$ is the density at the $i$-th corner of the voxel [S1, unofficial].

**Trilinear Interpolation:**
For an edge between corners with densities $V_a$ and $V_b$ and positions $\mathbf{p}_a$ and $\mathbf{p}_b$:
$$ t = \frac{I - V_a}{V_b - V_a} $$
$$ \mathbf{p}_{vertex} = (1 - t)\mathbf{p}_a + t\mathbf{p}_b $$
$$ \mathbf{n}_{vertex} = \nabla V(\mathbf{p}_{vertex}) $$
where $\nabla V$ is the gradient of the density field [S1, unofficial].

### Parameters

*   **Vertex Pool Cap:** 1,500,000 vertices [S1, unofficial].
*   **Isosurface Threshold ($I$):** Configurable via `?iso=`, default example $0.2$ [S1, unofficial].
*   **Grid Resolution:** Tested at $128^3$ [S1, unofficial].

### Edge Cases and Pitfalls

*   **Overflow:** If the number of generated vertices exceeds 1.5 million, the finalize kernel clamps the count to the cap. The overflow flag is raised to indicate this, ensuring that only complete triangles within the valid range are drawn, preventing garbage rendering [S1, unofficial].
*   **Overflow Flag Location:** The overflow flag is stored in Word 4 of the indirect draw arguments, not in `firstInstance`. This is because setting `firstInstance` to a non-zero value would no-op the indirect draw on hardware lacking the `indirect-first-instance` feature [S1, unofficial].
*   **Wall Capping:** To ensure the isosurface is capped on all six walls of the simulation domain, lattice index 0 is treated as having outside density (0). Blur bleed outside the walls (cells $2 \dots GRID-2$) is discarded. Out-of-range lattice samples in `mcEmit` are treated as density 0 [S1, unofficial].
*   **Table Consistency:** The triangle table is generated at load time rather than hardcoded to prevent drift between the JavaScript generator and the WGSL kernel. Assertions ensure the table matches the cut-edge sets of all 256 cases [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader: One thread per voxel
kernel MarchingCubesCompute(thread_id: int, grid_size: int, iso: float, 
                            density_grid: Texture3D, vertex_pool: Buffer, 
                            indirect_args: Buffer, triangle_table: Buffer):
    
    # 1. Calculate voxel coordinates
    x = thread_id % grid_size
    y = (thread_id / grid_size) % grid_size
    z = thread_id / (grid_size * grid_size)
    
    # 2. Read 8 corner densities
    # Handle out-of-range samples as density 0 for wall capping
    densities = [0.0] * 8
    for i in range(8):
        cx = x + (i & 1)
        cy = y + ((i >> 1) & 1)
        cz = z + ((i >> 2) & 1)
        if 0 <= cx < grid_size and 0 <= cy < grid_size and 0 <= cz < grid_size:
            densities[i] = density_grid.sample(cx, cy, cz)
        else:
            densities[i] = 0.0 # Outside density
            
    # 3. Compute case index
    case_index = 0
    for i in range(8):
        if densities[i] >= iso:
            case_index |= (1 << i)
            
    # 4. Lookup triangle table
    # triangle_table[case_index] returns a list of edge indices, terminated by -1
    edge_indices = triangle_table[case_index]
    
    if not edge_indices or edge_indices[0] == -1:
        return # No triangles for this case
        
    # 5. Reserve vertex slots atomically
    # Each triangle has 3 vertices. Reserve 3 * num_triangles slots.
    # Note: The table structure may vary; assuming standard MC where 
    # we know the number of triangles or iterate through edges.
    # The source mentions emitting edge-interpolated vertices.
    # We need to determine how many vertices to reserve.
    # Standard MC: 1-5 triangles per case.
    # Let's assume we iterate through the table to find triangles.
    
    num_triangles = 0
    triangles = []
    i = 0
    while i < len(edge_indices) and edge_indices[i] != -1:
        e1 = edge_indices[i]
        e2 = edge_indices[i+1]
        e3 = edge_indices[i+2]
        if e1 == -1 or e2 == -1 or e3 == -1:
            break
        triangles.append((e1, e2, e3))
        num_triangles += 1
        i += 3
        
    vertices_to_reserve = num_triangles * 3
    
    # Atomic add to indirect args word 0
    current_count = indirect_args[0]
    new_count = current_count + vertices_to_reserve
    indirect_args[0] = new_count
    
    # Base index for this voxel's vertices in the pool
    base_index = current_count
    
    # 6. Emit vertices
    # Cache for unique vertices per voxel to avoid duplicates
    vertex_cache = {} 
    
    for tri in triangles:
        for edge_idx in tri:
            edge = edge_indices[edge_idx] # Assuming edge_indices maps to actual edge definitions
            # Get edge endpoints
            # This part depends on the specific edge table structure.
            # Standard MC edge table maps case to edges.
            # Let's assume we have a function get_edge_vertices(edge_idx)
            # that returns the two corner indices of the edge.
            
            corner_indices = get_edge_vertices(edge)
            p1 = corner_indices[0]
            p2 = corner_indices[1]
            
            # Interpolate
            v1 = densities[p1]
            v2 = densities[p2]
            t = (iso - v1) / (v2 - v1)
            
            pos1 = get_corner_position(p1, x, y, z)
            pos2 = get_corner_position(p2, x, y, z)
            vertex_pos = lerp(pos1, pos2, t)
            
            # Compute normal (gradient)
            vertex_normal = compute_gradient(densities, x, y, z, vertex_pos)
            
            # Cache key to avoid duplicate vertices for shared edges
            # Use a hash of the edge or position
            cache_key = hash_edge(edge)
            
            if cache_key not in vertex_cache:
                vertex_cache[cache_key] = (vertex_pos, vertex_normal)
                
            # Write to pool
            pool_index = base_index + vertex_cache[cache_key].index_in_tri
            vertex_pool[pool_index].position = vertex_pos
            vertex_pool[pool_index].normal = vertex_normal
            
    # Note: The actual implementation in the source uses a specific 
    # edge interpolation logic and caching. The pseudocode above 
    # abstracts the specific edge-to-corner mapping.

kernel FinalizeIndirectArgs(indirect_args: Buffer, pool_cap: int):
    # Single thread execution
    current_count = indirect_args[0]
    overflow = 0
    
    if current_count > pool_cap:
        overflow = 1
        indirect_args[0] = pool_cap
        
    indirect_args[4] = overflow

# Helper functions (conceptual)
function get_edge_vertices(edge_idx):
    # Returns the two corner indices of the edge defined by the edge table
    pass

function get_corner_position(corner_idx, x, y, z):
    # Returns the 3D position of the corner
    pass

function compute_gradient(densities, x, y, z, pos):
    # Computes the gradient of the density field at pos
    pass

function hash_edge(edge_idx):
    # Creates a unique key for the edge
    pass
```

## Operating characteristics

*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels ($GRID^3$), as each voxel is processed by one thread. The atomic operations add a small constant overhead but avoid the $O(N \log N)$ or $O(N)$ prefix sum passes required by other methods.
*   **Memory Complexity:** $O(V_{max})$ for the vertex pool, where $V_{max}$ is the cap (1.5 million vertices). The triangle table is $O(1)$ (fixed 256 entries).
*   **Performance:** Benchmarked at 4.12 ms/frame on M-series hardware for a $128^3$ grid and $384^2$ screen resolution [S1, unofficial].
*   **Strengths:**
    *   Real-time performance due to parallel voxel processing and lack of prefix sums.
    *   Simple memory management with a fixed-size pool.
    *   Crack-free geometry via load-time table generation.
*   **Limitations:**
    *   Fixed vertex pool size may lead to overflow and clipping if the surface complexity exceeds 1.5 million vertices.
    *   Requires WebGPU for full functionality; WebGL2 falls back to a screen-space fluid (ssf) path [S1, unofficial].
    *   Atomic contention on the vertex counter may become a bottleneck at very high resolutions, though the source does not provide data on this scaling.

## Background

This method was developed by nikolai3d as part of the "waterfall" WebGL2 fluid simulation project, specifically for the `r=mesh` renderer [S1, unofficial]. It was introduced in Pull Request #17 to the `nikolai3d/waterfall` repository in July 2026 [S1, unofficial]. The technique was designed to provide a real-time isosurface mesh for MLS-MPM GPU fluid simulations, replacing previous rendering modes. It distinguishes itself from classic Marching Cubes implementations by generating the triangle table at load time and using atomic counters for vertex emission, avoiding the need for large constant buffers or prefix sum algorithms [S1, unofficial].

## Sources

[S1] Renderer: marching-cubes isosurface mesh (r=mesh, WebGPU), https://github.com/nikolai3d/waterfall/pull/17, fetched 2026-10-27 (unofficial)
