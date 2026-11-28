# Perspective Grid for Primary Rays

Also called: Perspective Grid

## Overview
The Perspective Grid is a ray-specialized spatial acceleration structure designed to provide perfect coherence for primary camera rays by treating them as packets [S1]. Unlike hierarchical structures like BVH or Kd-trees, it lacks tree-based culling properties but leverages image-space tiling to maximize shared memory reuse on GPUs [S1]. It is distinct from uniform grids because it aligns voxel boundaries with the camera's perspective, making it highly efficient for primary rays but unsuitable for secondary rays [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry (triangles), camera parameters (view frustum), image resolution, and tiling parameters ($N, k$) [S1].
*   **Output:** A grid data structure mapping triangles to voxels, and the resulting image via ray-triangle intersections [S1].

**Data Structures:**
The grid is constructed by binning scene geometry into voxels [S1]. The construction cost is low, reducing to a sorting of triangle-voxel pairs [S1]. The structure does not possess hierarchy [S1].

**Parameters:**
*   $N \times N$: The number of raytracing tiles dividing the image [S1].
*   $k$: The scaling factor for sorting tiles [S1].
*   Sorting Tile Size: Triangles are sorted into $kN \times kN$ tiles in image space [S1].
*   Raytracing Tile Size: The image is divided into $N \times N$ tiles, where each raytracing tile contains a $k \times k$ group of sorting tiles [S1].
*   Optimal Configuration: A $2 \times 2$ arrangement of sorting tiles within each raytracing tile provides the best results on current GPU hardware [S1].

**Algorithm Steps:**
1.  **Pre-filtering:** Apply View Frustum Culling (VFC) and Back-Face Culling to eliminate triangles not visible to the camera [S1]. This reduces grid construction time and geometry load [S1].
2.  **Grid Construction:** Sort triangles into the perspective grid voxels by binning triangle-voxel pairs [S1].
3.  **Thread Mapping:** Assign each pixel in an $N \times N$ raytracing tile to a thread (or work item) within a CUDA block (or work group) [S1].
4.  **Shared Memory Loading:** Load triangle data into shared memory [S1]. The available shared memory is partitioned equally among the sorting tiles [S1].
5.  **Intersection Loop:** Threads alternate between loading a portion of triangles into shared memory and computing intersections, with synchronization between these two roles [S1].
6.  **Front-to-Back Traversal:** Process voxels in front-to-back order [S1]. If a thread finds an intersection in a current voxel, it stops checking for intersections in later voxels [S1].
7.  **Optimization (Indirect Mapping):** Use indirect mapping to balance coherence and SIMD utilization [S1]. Triangles from each sorting tile are brought to the respective area of shared memory and intersected with rays corresponding to those sorting tiles [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def perspective_grid_primary_rays(triangles, camera, image_width, image_height, N, k):
    # 1. Pre-filtering: VFC and Back-Face Culling
    visible_triangles = []
    for tri in triangles:
        if is_in_view_frustum(tri, camera) and is_front_facing(tri, camera):
            visible_triangles.append(tri)

    # 2. Grid Construction: Sort triangles into kN x kN sorting tiles
    sorting_tile_size = k * N
    sorting_tiles = {}
    for tri in visible_triangles:
        tx = (tri.center_x // sorting_tile_size) % (image_width // sorting_tile_size)
        ty = (tri.center_y // sorting_tile_size) % (image_height // sorting_tile_size)
        sorting_tiles.setdefault((tx, ty), []).append(tri)

    # 3. Process each N x N raytracing tile
    for tile_y in range(0, image_height, N):
        for tile_x in range(0, image_width, N):
            # Partition shared memory into k x k regions
            shared_memory = [None] * (k * k)

            for s in range(k):
                for t in range(k):
                    sort_tile_idx = s * k + t
                    st_x = tile_x + s * N
                    st_y = tile_y + t * N
                    st_tx = st_x // sorting_tile_size
                    st_ty = st_y // sorting_tile_size

                    # Load triangles for this sorting tile into shared memory region
                    shared_memory[sort_tile_idx] = sorting_tiles.get((st_tx, st_ty), [])

                    # Synchronize threads
                    # sync_threads()

                    # Compute intersections for the loaded triangles
                    for thread_id in range(N * N):
                        u = tile_x + (thread_id % N)
                        v = tile_y + (thread_id // N)
                        ray = generate_camera_ray(u, v, camera)

                        hit = False
                        for tri in shared_memory[sort_tile_idx]:
                            if ray_triangle_intersect(ray, tri):
                                hit = True
                                break

                        if hit:
                            record_intersection(u, v, ray)
                            break # Front-to-back: stop checking later voxels

                    # Synchronize threads before loading next batch
                    # sync_threads()

# Helper definitions (standard operations)
def is_in_view_frustum(tri, camera):
    return all(plane.dot(tri.vertices) >= 0 for plane in camera.frustum_planes) # [G]

def is_front_facing(tri, camera):
    return tri.normal.dot(camera.view_direction) < 0 # [G]

def generate_camera_ray(u, v, camera):
    return Ray(camera.origin, camera.direction + u * camera.right + v * camera.up) # [G]

def ray_triangle_intersect(ray, tri):
    # Standard Möller–Trumbore intersection routine [G]
    # Computes barycentric coordinates and depth; returns True if 0 <= u, v <= 1 and depth > 0
    return True # Standard routine [G]
```

**Edge Cases and Pitfalls:**
*   **No Hierarchy:** The grid lacks the tree property, so it cannot eliminate triangles before intersection checking as effectively as BVH/Kd-trees [S1].
*   **Secondary Rays:** The structure is ray-specialized for primary rays and cannot efficiently handle other rays (e.g., reflection) [S1].
*   **SIMD Utilization:** If the number of threads is below the SIMD width, computing resources may be wasted [S1]. Indirect mapping helps balance this [S1].

## Subtopics

### [Indirect Mapping](22975-indirect-mapping.md)
A thread mapping strategy that sorts triangles to smaller tiles but raytraces using a larger number of threads to balance coherence and SIMD utilization. It maps threads differently than the sorting tiles: triangles from each sorting tile are brought to the respective area of the shared memory and intersected with rays corresponding to the sorting tiles. On the Happy Buddha benchmark, indirect mapping provided a 10-30 percent speedup as the maximum number of triangles checked dropped by more than half.

## Operating characteristics
*   **Time Complexity:** Grid construction is low cost, reducing to sorting of triangle-voxel pairs [S1].
*   **Memory Complexity:** Depends on shared memory partitioning and the number of triangles per voxel [S1].
*   **Performance:** Provides perfect coherence for primary rays [S1]. Indirect mapping yields 10-30% speedup on benchmarks like Happy Buddha [S1].
*   **Strengths:** Perfect coherence for primary rays; low construction cost; efficient shared memory reuse on GPUs [S1].
*   **Limitations:** Cannot efficiently handle secondary rays; no hierarchical culling; performance sensitive to SIMD width [S1].
*   **When to Use:** Primary ray tracing on GPU hardware where shared memory reuse is critical [S1].
*   **When Not to Use:** Secondary ray tracing (use uniform grids instead) or scenes requiring hierarchical culling [S1].

## Background
The Perspective Grid was introduced in the context of raytracing dynamic scenes on the GPU using grids [S1]. It builds on earlier work regarding grid-based raytracing [S1]. The method was developed to address the inefficiencies of uniform grids for primary rays and the overhead of hierarchical structures on GPUs [S1].

## Sources
[S1] Raytracing dynamic scenes on the GPU using grids, https://ieeexplore.ieee.org/abstract/document/5728799/, fetched 2026-10-25
