# Triangulated Active Brick Boundary Mesh for Space Skipping

Also called: Active Brick Boundary Mesh, Active Region Boundary Geometry, Boundary Mesh Space Skipping, GPU-Extracted Boundary Geometry, RTX-Accelerated Empty Space Skipping using Triangulated Active Region Boundary Geometry, Space Skipping with Boundary Mesh, Triangulated Active Region Boundary Geometry

## Overview
A GPU-accelerated space-skipping technique for structured volume rendering that replaces clustered proxy boxes with a triangle mesh bounding the surface between active and inactive macrocells. It differs from [Macro-Cell Trimming](1023-macro-cell-trimming.md) and [Adaptive Ray Marching with Macrocell DDA](1500-adaptive-ray-marching-with-macrocell-dda.md) by generating a boundary mesh entirely on the GPU, avoiding host-to-device transfers and user-defined intersection programs, and leveraging native hardware-accelerated triangle intersection to reduce depth complexity and warp divergence.

## How it works
**Inputs:**
- Input volume dimensions $V_x \times V_y \times V_z$ (voxels).
- Brick size $K$ (default 16, also 8 or 32).
- Transfer function (TF).
- OptiX 7.2 (or equivalent DXR/VKR) runtime.

**Outputs:**
- Rendering intervals $[t_0, t_1]$ for volume integration.
- Final pixel color.

**Data Structures:**
1.  **Macrocell Grid:** 3D grid of size $M_x \times M_y \times M_z$, where $M_x = \lceil V_x / K \rceil$, $M_y = \lceil V_y / K \rceil$, $M_z = \lceil V_z / K \rceil$. Each cell stores $\{min\_val, max\_val\}$ of all input voxels mapping to it. Computed on host, uploaded to GPU.
2.  **Active Bricks Mask:** Boolean grid of size $M_x \times M_y \times M_z$ on GPU. `active[i][j][k]` is true if the macrocell's value range maps to any non-zero alpha in the TF.
3.  **Vertex Array:** Pre-allocated device array of `float3` vertices. Size $N_{vtxMax} = (M_x + 1) \times (M_y + 1) \times (M_z + 1)$. Contains all possible macrocell grid vertices.
4.  **Triangle Index Array:** Pre-allocated device array of `int3` indices. Size $N_{triMax}$ (upper bound derived from boundary geometry on macrocell faces).
5.  **numTriangles Counter:** Single device-side integer, initialized to 0. Used as atomic offset for writing triangles.

**Steps:**

1.  **Initial Setup:**
    - Build macrocell grid on host. Upload to GPU.
    - Allocate GPU memory for Active Bricks Mask, Vertex Array, Triangle Index Array, and `numTriangles` counter.

2.  **Active Cell Classification (Rebuild Step 1):**
    - Launch CUDA kernel. Each thread maps to one macrocell $(i, j, k)$.
    - Thread checks if `macrocell_grid[i][j][k].min_val` to `macrocell_grid[i][j][k].max_val` intersects the non-zero alpha range of the current TF.
    - Write result to `active_cells[i][j][k]`.

3.  **Boundary Triangle Generation (Rebuild Step 2):**
    - Reset `numTriangles` to 0.
    - Launch three templated CUDA kernels (for X, Y, Z orientations).
    - For each orientation, threads map to faces perpendicular to that axis.
    - For a face between `curr` and `prev` macrocells:
        - If `active_cells[prev] == active_cells[curr]`, return.
        - Compute indices of four vertices spanning the face.
        - Atomically increment `numTriangles` by 2.
        - Write two triangle indices to `triangles` buffer at offset `numTriangles - 2`.
        - Winding order points away from active to inactive cell.
    - Download `numTriangles` to host.

4.  **BVH Rebuild (Rebuild Step 3):**
    - Update OptiX triangle geometry to reference the generated mesh (triangle count and index array pointer).
    - Ask OptiX to rebuild BVH with 'optimize for traversal' flag.
    - Compact BVH to conserve memory.

5.  **Space Skipping during Rendering:**
    - Trace ray against boundary mesh.
    - **Closest Hit:** Record hit distance $t_{hit}$ and hit kind (front/back).
    - **No Hit:** Terminate ray (no active regions).
    - **Back-Facing Hit:** Ray is inside volume. Integration interval is $[0, t_{hit}]$.
    - **Front-Facing Hit:** Ray enters volume at $t_0 = t_{hit}$. Trace again with $ray.t_{min}=t_0$ to find exit distance $t_1$.
    - Integrate volume over $[t_0, t_1]$ using gradient shading and early ray termination.
    - If early termination, return. Otherwise, iterate: trace new ray from $t_1$ to find next segment.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CUDA Kernel for Boundary Triangle Generation
template <int DIM>
__global__ void generateBoundaryMeshKernel(int &numTriangles, int3 *triangles, bool activeCells[][]) {
    // Thread maps to a face perpendicular to DIM
    int3 curr = getGlobalThreadIndex(); // Index of the face
    int3 prev;
    
    // Determine the neighbor macrocell across the face
    if constexpr(DIM == 0) prev = curr - {1, 0, 0};
    else if constexpr(DIM == 1) prev = curr - {0, 1, 0};
    else if constexpr(DIM == 2) prev = curr - {0, 0, 1};

    // Check boundary conditions (treat outside as inactive)
    if (!isValidIndex(prev) || !isValidIndex(curr)) return;

    // Only proceed if active status differs
    if (activeCells[prev] == activeCells[curr]) return;

    // Get vertex list indices for the boundary quad connecting prev and curr
    int vi[4] = getBoundaryQuad(curr, DIM);

    // Generate two more triangles and append to global list
    int triID = atomicAdd(&numTriangles, 2);

    if (activeCells[prev]) {
        // Active boundary, facing from prev to curr
        // Winding order points away from active (prev) to inactive (curr)
        triangles[triID + 0] = {vi[0], vi[1], vi[2]};
        triangles[triID + 1] = {vi[0], vi[2], vi[3]};
    } else if (activeCells[curr]) {
        // Active boundary, facing from curr to prev
        // Winding order points away from active (curr) to inactive (prev)
        triangles[triID + 0] = {vi[1], vi[0], vi[2]};
        triangles[triID + 1] = {vi[2], vi[0], vi[3]};
    }
}

# Rendering Loop
void renderRay(Ray ray) {
    float t_min = 0.0f;
    while (true) {
        // Trace against boundary mesh
        Hit hit = traceRay(ray, t_min);
        
        if (hit.hit == false) {
            // No active regions ahead
            break;
        }
        
        float t_hit = hit.distance;
        bool is_back_facing = hit.kind == BACK_FACE;
        
        if (is_back_facing) {
            // Ray is inside volume, exiting at t_hit
            float t_end = t_hit;
            integrateVolume(ray, t_min, t_end);
            if (earlyTerminationReached()) return;
            t_min = t_end;
        } else {
            // Ray is entering volume at t_hit
            float t_enter = t_hit;
            // Trace again to find exit
            Hit exitHit = traceRay(ray, t_enter);
            if (exitHit.hit == false) {
                // Should not happen if mesh is closed, but handle gracefully
                break;
            }
            float t_exit = exitHit.distance;
            integrateVolume(ray, t_enter, t_exit);
            if (earlyTerminationReached()) return;
            t_min = t_exit;
        }
    }
}
```

**Parameters:**
- $K$: Brick size (8, 16, 32). Default 16.
- $dt$: Fixed sampling rate for volume integration (0.5).
- Interleaved Sampling: Used to reduce aliasing.

**Edge Cases and Pitfalls:**
- **Boundary Faces:** Faces on the outside of the volume have indices mapping outside the macrocell grid. These are treated as inactive macrocells to ensure correct boundary detection [S1, S2].
- **Winding Order:** Must point away from active to inactive cell for correct front/back face detection [S1, S2].
- **Unused Vertices:** Pre-allocating all possible vertices results in unused vertices, accepted as memory cost [S1, S2].
- **Warp Divergence:** Reduced by lowering depth complexity to transitions between active/inactive regions [S1, S2].

## Subtopics

### [RTX-Accelerated Empty Space Skipping using Triangulated Active Bricks](6384-rtx-accelerated-empty-space-skipping-using-triangulated-acti.md)
Accelerates volume rendering by using RTX-accelerated ray tracing to skip empty space. It constructs a BVH over the boundary surface between active and inactive macrocells, rather than over all active regions, to determine ray integration intervals. It differs from this method by explicitly leveraging hardware-accelerated BVH traversal and triangle intersection via OptiX/OWL to replace costly software intersection tests, and from Ganter and Manzke's method by replacing the BVH over active brick regions with a BVH over only the boundary, resulting in lower depth complexity and fewer rays.

## Operating characteristics
- **Time Complexity:** BVH rebuild and triangle generation are GPU-side, avoiding host-GPU transfer bottlenecks. Depth complexity is reduced to transitions between active/inactive regions.
- **Memory Complexity:** Upper bounds for vertices and triangles are 3-5 orders of magnitude lower than the number of voxels [S1].
- **Performance:** Reduces per-pixel cost by freeing CUDA cores for volume integration. Eliminates warp divergence caused by varying numbers of samples per box [S1, S2].
- **Strengths:** No host-GPU transfer for mesh generation. Native hardware acceleration for intersection tests. Lower depth complexity than proxy box methods.
- **Limitations:** Requires BVH rebuild on TF change. Pre-allocated vertex array may waste memory.
- **When to Use:** Structured volumes with interactive TF editing. When hardware ray tracing is available.
- **When Not to Use:** Unstructured volumes (use [Adaptive Ray Marching with Macrocell DDA](1500-adaptive-ray-marching-with-macrocell-dda.md) or unstructured methods). When TF changes are rare and host-side preprocessing is acceptable.

## Background
Originated from Ganter and Manzke's method [GM19], which used proxy boxes and user-defined intersection programs. This method improves on it by using a GPU-generated boundary mesh and native triangle primitives. The core idea draws motivation from unstructured volume rendering where rays are intersected with outside-facing triangles of boundary tetrahedra [BKS97]. Implemented using OptiX 7.2, but maps to DXR and VKR. Used in evaluation datasets: Heptane ($302^3$), Flower ($1024^3$), DNS-4 ($2560 \times 1920 \times 384$), LLNL Richtmyer-Meshkov instability ($2048^2 \times 1920$) [S1, S2].

## Sources
[S1] Faster RTX-Accelerated Empty Space Skipping using Triangulated Active ..., https://www.sci.utah.edu/~wald/Publications/2021/spaceskip/ss.pdf, fetched 2026-10-29
[S2] Faster RTX-Accelerated Empty Space Skipping using Triangulated Active ..., https://diglib.eg.org/items/dcc28036-9ef9-42d2-bff8-7a4bf1269b57, fetched 2026-10-29
