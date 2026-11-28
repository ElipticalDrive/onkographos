# Parallel Jumping Farthest Away (JFA) — Compute Shader

Also called: 1+JFA, Compute Shader JFA, GPU JFA for SDF, JFA Pass, Jump Flooding Algorithm (JFA) for SDF, SDF Recalculation (JFA)

## Overview
This method accelerates Signed Distance Field (SDF) voxelization by dispatching a compute shader per triangle to seed distances into a 3D texture, followed by a parallel jump flooding pass to propagate accurate distances across the grid [S2, unofficial]. It differs from sequential JFA by leveraging GPU compute shaders with atomic operations for concurrent triangle-voxel updates, and from the OpenMP parallel JFA by performing both seeding and flooding on the GPU [S2, unofficial].

## How it works
The algorithm executes two compute shader passes: an initial distance seeding pass and an iterative jump flooding pass [S2, unofficial].

### Inputs and Outputs
*   **Input:** A triangle mesh, a 3D voxel grid defined by its dimensions $(W, H, D)$, and the grid's spatial bounds [S2, unofficial].
*   **Output:** A 3D image (`image3D`) storing the signed distance value for each voxel [S2, unofficial].

### Data Structures
*   **Voxel Grid:** Stored as a 3D image (`image3D`) in GPU memory [S2, unofficial].
*   **Texel Layout:** Each texel holds a single floating-point distance value [G].
*   **Triangle Representation:** Defined by three vertices in 3D space [S2, unofficial].

### Parameters
*   **Number of Iterations ($N$):** Calculated as $\lceil \log_2(\max(W, H, D)) \rceil$ [S2, unofficial].
*   **Step Length ($S_i$):** For iteration $i$ (where $0 \le i < N$), the step length is $S_i = N - i$ [S2, unofficial]. The source explicitly defines the step length as the number of iterations minus the current iteration, which differs from standard binary JFA implementations that use powers of two [S2, unofficial].

### Steps
1.  **Pass 1: Triangle-Centric Seeding**
    *   Dispatch a compute shader where each invocation processes a triangle [S2, unofficial].
    *   Compute an Axis-Aligned Bounding Box (AABB) that encapsulates the triangle [S2, unofficial].
    *   Iterate through every voxel grid cell that overlaps the AABB [S2, unofficial].
    *   Calculate the Euclidean distance from the cell center to the triangle surface [S2, unofficial].
    *   Use atomic operations to load and store the distance to the `image3D` texture, ensuring concurrent writes from multiple overlapping triangles are handled correctly [S2, unofficial].
    *   Cells within a certain threshold distance to the original triangles are treated as seeds and are not updated in subsequent passes [S2, unofficial].

2.  **Pass 2: Parallel Jump Flooding**
    *   Dispatch a compute shader where each invocation corresponds to a single voxel cell $(x, y, z)$ [S2, unofficial].
    *   Repeat for $i = 0$ to $N-1$:
        *   Calculate the step length $S_i = N - i$ [S2, unofficial].
        *   Read the current distance $d_{curr}$ from the `image3D` texture at $(x, y, z)$ [G].
        *   Sample distances from neighboring cells in the four cardinal directions and diagonals, offset by $\pm S_i$ in each axis [S2, unofficial].
        *   For each neighbor at offset $\vec{o}$ with distance $d_{neighbor}$, calculate the Euclidean distance between the current cell and the neighbor cell: $dist_{cell\_neighbor} = \|\vec{o}\|$ [S2, unofficial].
        *   Compute the potential new distance: $d_{new} = d_{neighbor} + dist_{cell\_neighbor}$ [S2, unofficial].
        *   Select the minimum distance among all neighbors and update $d_{curr} = \min(d_{curr}, d_{new})$ [S2, unofficial].
        *   Write the updated minimum distance back to the `image3D` texture at $(x, y, z)$ [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_aabb(tri):
    """Returns (min_x, min_y, min_z, max_x, max_y, max_z) for a triangle."""
    xs = [v[0] for v in tri]
    ys = [v[1] for v in tri]
    zs = [v[2] for v in tri]
    return (min(xs), min(ys), min(zs), max(xs), max(ys), max(zs))

def get_overlapping_voxels(aabb, grid_dims, voxel_size, origin):
    """Returns list of voxel indices overlapping the AABB."""
    min_x = int((aabb[0] - origin[0]) / voxel_size)
    max_x = int((aabb[3] - origin[0]) / voxel_size)
    min_y = int((aabb[1] - origin[1]) / voxel_size)
    max_y = int((aabb[4] - origin[1]) / voxel_size)
    min_z = int((aabb[2] - origin[2]) / voxel_size)
    max_z = int((aabb[5] - origin[2]) / voxel_size)
    voxels = []
    for x in range(min_x, max_x + 1):
        for y in range(min_y, max_y + 1):
            for z in range(min_z, max_z + 1):
                voxels.append((x, y, z))
    return voxels

def distance_voxel_to_triangle(voxel_idx, tri, voxel_size, origin):
    """Returns Euclidean distance from voxel center to triangle surface [G]."""
    cx = origin[0] + (voxel_idx[0] + 0.5) * voxel_size
    cy = origin[1] + (voxel_idx[1] + 0.5) * voxel_size
    cz = origin[2] + (voxel_idx[2] + 0.5) * voxel_size
    # Standard point-to-triangle distance calculation [G]
    # Simplified to distance to first vertex for pseudocode completeness
    return math.sqrt((cx - tri[0][0])**2 + (cy - tri[0][1])**2 + (cz - tri[0][2])**2)

def in_bounds(coord, grid_dims):
    """Checks if a voxel coordinate is within grid dimensions."""
    return 0 <= coord[0] < grid_dims[0] and 0 <= coord[1] < grid_dims[1] and 0 <= coord[2] < grid_dims[2]

def get_current_voxel_coord():
    """Returns the global invocation ID mapped to voxel coordinates [G]."""
    return (0, 0, 0)  # Placeholder for shader invocation ID mapping

def triangle_seeding_shader(triangles, grid_dims, voxel_size, origin, image_3d):
    """Pass 1: Seeds distances using triangle-centric compute shader."""
    for tri in triangles:
        aabb = compute_aabb(tri)
        voxels = get_overlapping_voxels(aabb, grid_dims, voxel_size, origin)
        for coord in voxels:
            dist = distance_voxel_to_triangle(coord, tri, voxel_size, origin)
            # Atomic compare-and-swap to keep minimum distance [S2, unofficial]
            old_dist = image_3d[coord]
            if dist < old_dist:
                image_3d[coord] = dist

def jump_flooding_shader(grid_dims, image_3d):
    """Pass 2: Propagates distances using parallel jump flooding."""
    N = math.ceil(math.log2(max(grid_dims)))
    for i in range(N):
        step = N - i  # Step length per source definition [S2, unofficial]
        coord = get_current_voxel_coord()
        current_dist = image_3d[coord]
        
        # Check neighbors in cardinal and diagonal directions offset by step [S2, unofficial]
        for dx in [-step, 0, step]:
            for dy in [-step, 0, step]:
                for dz in [-step, 0, step]:
                    if dx == 0 and dy == 0 and dz == 0:
                        continue
                    neighbor_coord = (coord[0] + dx, coord[1] + dy, coord[2] + dz)
                    if not in_bounds(neighbor_coord, grid_dims):
                        continue
                    neighbor_dist = image_3d[neighbor_coord]
                    cell_dist = math.sqrt(dx*dx + dy*dy + dz*dz)
                    new_dist = neighbor_dist + cell_dist
                    if new_dist < current_dist:
                        current_dist = new_dist
        image_3d[coord] = current_dist
```

### Edge Cases and Pitfalls
*   **Grid Bounds:** Triangles extending outside the grid bounds result in geometry being cut off if not properly transformed or clipped [S2, unofficial].
*   **Atomic Operations:** Concurrent writes from overlapping triangles require atomic operations to prevent data races during the seeding phase [S2, unofficial].
*   **Step Length Definition:** The source defines step length linearly as $N - i$, which requires careful implementation to match the documented behavior rather than assuming standard binary JFA power-of-two steps [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V)$ for the flooding pass, where $N = \lceil \log_2(\max(W, H, D)) \rceil$ and $V$ is the number of voxels [S2, unofficial]. The seeding pass is $O(T \cdot C)$, where $T$ is the number of triangles and $C$ is the average number of cells per triangle AABB [S2, unofficial].
*   **Memory Complexity:** $O(V)$ for the `image3D` texture storage [S2, unofficial].
*   **Performance:** Highly parallelizable on GPU hardware [S2, unofficial]. The seeding phase may experience contention when multiple triangles update the same voxels simultaneously, mitigated by atomic operations [S2, unofficial]. The flooding phase is memory-bound due to texture sampling [S2, unofficial].
*   **Strengths:** Fast generation of approximate SDFs for large voxel grids using standard compute shaders [S2, unofficial].
*   **Limitations:** Produces an approximation of the exact SDF rather than an exact distance field [S1]. Performance scales with triangle count and grid resolution [S2, unofficial].

## Background
The Jump Flooding Algorithm (JFA) was originally introduced by Rong and Tan (2006) for nearest neighbor field computation [S2, unofficial]. This specific implementation adapts JFA for SDF voxelization using Vulkan Compute Shaders, as documented by William Whitehouse in 2022/23 [S2, unofficial]. The method is used to convert a Truncated Signed Distance Field (TSDF) into a full SDF to enable sphere tracing [S1].

## Sources
[S1] Interactive Editing of Voxel-Based Signed Distance Fields, http://wscg.zcu.cz/WSCG2022/journal/B97-full.pdf, fetched 2026-10-30
[S2] Generating Signed Distance Fields from Triangle Meshes using Vulkan Compute Shaders, https://williamwhitehouse.dev/assets/projects/2023/sdf-gen/gen-sdf-from-meshes-using-vulkan-compute-report.pdf, fetched 2026-10-30 (unofficial)
