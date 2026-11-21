# Marching Slices with Slab-based Caching

Also called: Marching Slices, Slab-based Marching Cubes

## Overview
Marching Slices is an isosurface extraction algorithm that reduces memory usage by processing a volume one slice at a time, caching only the data necessary for the current processing unit [S1]. It solves the problem of high memory residency required by traditional marching cubes by partitioning the volume into slabs and using boundary consistency via slice caches to ensure geometric continuity [S1]. Unlike surface-following marching cubes which keeps all $N_z + 1$ slices in memory, this approach uses significantly less working memory and is faster due to increased locality of memory accesses [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field defined on a grid of size $N_x \times N_y \times N_z$ [S1].
*   **Output:** A stream of triangles (vertices and indices) representing the isosurface [S1].

**Data Structures:**
The algorithm treats the volume as a vertical array of $N_z$ slabs of size $N_x \times N_y \times 1$ cubes [S1]. The slabs are numbered $0$ to $N_z-1$ (where $z$ increases upwards) [S1]. Each slab is bounded above and below by a two-dimensional slice, with adjacent slabs sharing slices [S1]. The slices are numbered $0$ to $N_z$ [S1].

1.  **Todo List:** For each of the $N_z$ slabs, a linked list containing integer $(x, y)$ tuples representing cubes in the slab that may need to be rendered [S1].
2.  **Slice Cache:** Allocated for the slice above and below a slab being polygonized [S1]. Each cache consists of:
    *   A 2D **corner index array** of size $(N_x + 1) \times (N_y + 1)$ [S1]. Each element stores an index to retrieve a computed corner from the corner cache (or $0$ if not computed) [S1].
    *   A **corner cache**: A dynamically resizing vector storing the actual corner values (interpolated positions) [S1].
    *   **Done Flags:** Each element in the corner index array also includes two boolean flags: `done_above` and `done_below` [S1]. These indicate whether the cubes immediately above and below the slice at that corner location have been rendered [S1].

**Algorithm Steps:**

1.  **Initialization:**
    *   Create $N_z$ empty todo lists, one for each slab [S1].
    *   Identify seed cubes [G]. Since field values are not known a-priori, the algorithm cannot simply march back-to-front [S1]. It must find seed cubes and add their $(x, y)$ coordinates to the corresponding slab's todo list [S1].
    *   Allocate a common memory pool for slice caches [S1].

2.  **Processing Loop:**
    While there are non-empty todo lists in any slab:
    1.  Select a slab $s$ with a non-empty todo list [S1].
    2.  Ensure slice caches are allocated for the slice below slab $s$ (slice $s$) and the slice above slab $s$ (slice $s+1$) [S1]. If a cache is not allocated, allocate it and initialize its corner index array and done flags to zero/false [S1].
    3.  While slab $s$'s todo list is not empty:
        1.  Pop a cube $(x, y)$ from slab $s$'s todo list [S1].
        2.  **Polygonize:** Perform the standard marching cubes lookup for the cube defined by corners at $(x, y, z)$ and $(x+1, y+1, z+1)$ within slab $s$ [G].
        3.  **Corner Retrieval:** For each of the 8 corners of the cube:
            *   Check the corner index array of the relevant slice cache (above or below depending on the corner's $z$-coordinate) [S1].
            *   If the index is $0$, compute the corner position (interpolate based on field values) and store it in the corner cache [G]. Update the index array with the new cache index [G].
        4.  **Mark Done:** For each of the 8 corners, update the `done_above` and `done_below` flags in the respective slice caches to indicate that the cubes adjacent to these corners in the current slab have been processed [S1].
        5.  **Neighbor Propagation:** For each of the 8 corners, check the adjacent cubes in neighboring slabs [S1]. If a neighbor cube has not been processed (checked via the `done` flags in the slice cache of the *other* slab), add its $(x, y)$ coordinate to the todo list of that neighbor slab [S1].
            *   Specifically, if a corner is shared with a cube in slab $s-1$ (below) or $s+1$ (above), and that cube's `done` flag is false, push the cube's $(x, y)$ to the respective slab's todo list [S1].
    4.  **Cache Management:**
        *   After slab $s$'s todo list is empty, check if the slice cache below slab $s$ (slice $s$) can be deallocated [S1]. This is safe if the todo list for slab $s-1$ is also empty [S1].
        *   Similarly, check if the slice cache above slab $s$ (slice $s+1$) can be deallocated if the todo list for slab $s+1$ is empty [S1].
        *   When a slice cache is deallocated, reset its corner index arrays by clearing nonzero indices and done flags using the $(x, y)$ entries in its corner caches, and return the memory to the common pool [S1].

3.  **Completion:**
    *   The algorithm completes when there are no more todo list items for any slab in the block [S1].
    *   Flush the accumulated index and vertex buffers to the video card [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_slices(volume, isovalue, Nx, Ny, Nz):
    # Data structures
    todo_lists = [[] for _ in range(Nz)]
    slice_caches = [None] * (Nz + 1)
    corner_pool = []

    # 1. Initialization
    seeds = find_seeds(volume, isovalue, Nx, Ny, Nz)
    for (x, y, z) in seeds:
        if 0 <= z < Nz:
            todo_lists[z].append((x, y))

    # 2. Processing Loop
    while any(todo_lists):
        slab = next(i for i, tl in enumerate(todo_lists) if tl)

        # Ensure caches for slices below (slab) and above (slab+1)
        if slice_caches[slab] is None:
            slice_caches[slab] = allocate_cache(Nx, Ny)
        if slice_caches[slab + 1] is None:
            slice_caches[slab + 1] = allocate_cache(Nx, Ny)

        while todo_lists[slab]:
            x, y = todo_lists[slab].pop()

            # Polygonize cube at (x, y, slab)
            corners = get_cube_corners(x, y, slab)
            triangles = march_cube(volume, isovalue, corners)
            output_triangles(triangles)

            for cx, cy, cz in corners:
                cache = slice_caches[cz]
                if cache.index[cx][cy] == 0:
                    pos = interpolate_corner(volume, (cx, cy, cz), isovalue)
                    idx = len(corner_pool)
                    corner_pool.append(pos)
                    cache.index[cx][cy] = idx + 1
                # Mark done flags
                if cz == slab:
                    cache.done_above[cx][cy] = True
                elif cz == slab + 1:
                    cache.done_below[cx][cy] = True

                # Propagate to neighbors
                if cz == slab + 1 and not slice_caches[slab + 1].done_above[cx][cy]:
                    if slab + 1 < Nz:
                        todo_lists[slab + 1].append((cx, cy))
                if cz == slab and not slice_caches[slab].done_below[cx][cy]:
                    if slab > 0:
                        todo_lists[slab - 1].append((cx, cy))

        # Cache management
        if slab > 0 and not todo_lists[slab - 1]:
            if slice_caches[slab] is not None:
                deallocate_cache(slice_caches[slab])
                slice_caches[slab] = None
        if slab + 1 < Nz and not todo_lists[slab + 1]:
            if slice_caches[slab + 1] is not None:
                deallocate_cache(slice_caches[slab + 1])
                slice_caches[slab + 1] = None

    flush_buffers()

def allocate_cache(Nx, Ny):
    return {
        'index': [[0] * (Ny + 1) for _ in range(Nx + 1)],
        'done_above': [[False] * (Ny + 1) for _ in range(Nx + 1)],
        'done_below': [[False] * (Ny + 1) for _ in range(Nx + 1)]
    }

def deallocate_cache(cache):
    for x in range(len(cache['index'])):
        for y in range(len(cache['index'][0])):
            cache['index'][x][y] = 0
            cache['done_above'][x][y] = False
            cache['done_below'][x][y] = False

def find_seeds(volume, isovalue, Nx, Ny, Nz):
    seeds = []
    for z in range(Nz):
        for y in range(Ny):
            for x in range(Nx):
                if volume[x, y, z] < isovalue and (x == 0 or volume[x-1, y, z] >= isovalue):
                    seeds.append((x, y, z))
    return seeds

def get_cube_corners(x, y, slab):
    return [
        (x, y, slab), (x+1, y, slab), (x, y+1, slab), (x+1, y+1, slab),
        (x, y, slab+1), (x+1, y, slab+1), (x, y+1, slab+1), (x+1, y+1, slab+1)
    ]

def interpolate_corner(volume, corner, isovalue):
    x, y, z = corner
    v0 = volume[x, y, z]
    v1 = volume[x+1, y, z] if x < Nx else volume[x, y, z]
    t = (isovalue - v0) / (v1 - v0) if v1 != v0 else 0.0
    return (x + t, y, z)

def march_cube(volume, isovalue, corners):
    return []

def output_triangles(triangles):
    pass

def flush_buffers():
    pass
```

**Parameters:**
*   **Number of slices in memory:** Typically three slices are kept in memory at the same time [S1].
*   **Grid Size:** For a $110 \times 110 \times 110$ grid, the cache memory used is below 1MB [S1].

**Edge Cases and Pitfalls:**
*   **Seeding Failure:** Seeding algorithms may fail to find all local minima, causing the algorithm to step downwards to render connected cubes in slabs below that were missed [S1]. This requires additional slices to store the abandoned frontier and breaks the unidirectional order of polygonization [S1].
*   **Arbitrary Order:** Because field values are not known a-priori, the algorithm must support arbitrary forward/reverse traversal order, which complicates the simple "march back-to-front" assumption [S1].
*   **Redundant Visits:** Care must be taken to avoid redundant visits to completed cubes [S1]. The done flags in the slice cache prevent this [S1].

## Operating characteristics
*   **Memory Complexity:** Significantly lower than full-volume marching cubes [S1]. For a $110 \times 110 \times 110$ grid, memory usage is below 1MB [S1]. It uses much less working memory than surface-following marching cubes which keeps all $N_z + 1$ slices in memory [S1].
*   **Time Complexity:** Faster than hash-map based approaches due to the avoidance of hash map lookup costs [S1]. The use of 2D arrays for corner indices provides $O(1)$ access time [S1].
*   **Strengths:** Low memory footprint, high performance due to memory locality, straightforward parallelization of blocks [S1].
*   **Limitations:** Seeding failures can lead to complex traversal patterns and potential performance degradation if many slabs need to be revisited [S1].
*   **When to Use:** Suitable for real-time applications or environments with strict memory constraints where the volume is too large to fit in memory [S1].

## Background
The Marching Slices algorithm was introduced in the paper "Real-time particle isosurface extraction" [S1]. It was developed to address the memory limitations of traditional marching cubes when dealing with large volumes or real-time applications [S1]. The method builds on the principles of isosurface extraction but optimizes for memory by processing the volume in slabs and caching only necessary boundary data [S1].

## Sources
[S1] Real-time particle isosurface extraction, https://doi.org/10.1145/1342250.1342256, fetched 2026-11-04
