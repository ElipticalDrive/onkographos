# Multi-resolution grids

Also called: Hierarchical voxel grids, JW89, Jevans and Wyvill method, Jevans and Wyvill's approach

## Overview
Multi-resolution grids (JW89) is a recursive hierarchical subdivision algorithm for ray tracing that integrates regular and adaptive spatial subdivision methods [S2]. It solves the problem of efficiently rendering scenes containing both large, regularly distributed objects and small, dense clusters by allowing variable resolution across the spatial domain [S2]. Unlike fixed-resolution grids or purely octree-based approaches, voxels in recursive grids cannot overlap, so each nested grid along the ray path is tested exactly once [S1, unofficial]. It uses a top-down build strategy where internal voxels maintain their own local sub-grids, enabling efficient traversal through deep recursion when paired with a hash-table indexing strategy [S2].

## How it works
**Inputs and Outputs**
*   **Input:** A set of geometric primitives and a scene bounding box.
*   **Output:** A hierarchical grid data structure (root voxel) and a traversal state for ray intersection.

**Data Structures**
The structure is a loosely nested recursive grid [S2].
*   **Voxel:** The fundamental unit.
    *   **Type:** Enum { LEAF, INTERNAL }.
    *   **Leaf Voxel:** Contains a list of primitives intersecting the voxel volume [S2].
    *   **Internal Voxel:** Contains a local voxel grid subdivision (child voxels) and a hash table mapping child coordinates to child voxel pointers [S2].
*   **Hash Table:** Used to maintain each non-empty internal voxel, allowing direct access to child grids and enabling deep recursion without linear search overhead [S2].

**Build Process (Top-Down)**
All primitives are initially inserted into a single root voxel [S2]. The build process is top-down [S2]. The algorithm recursively subdivides internal voxels based on one of the following criteria [S2]:
1.  **Octree-like:** Subdivide into a $2 \times 2 \times 2$ grid (8 children) [S2].
2.  **Maximum Depth:** Stop subdivision when a predefined tree depth is reached [S2].
3.  **Constant Resolution:** Fix the resolution of voxel subgrids to a specific value regardless of content density [S2].
4.  **Variable (Adaptive) Resolution:** Subdivide based on occupancy or primitive count thresholds [S2].

**Parameters**
*   Coarseness of the initial voxel grid [S2].
*   Depth of adaptive subdivision trees [S2].
*   Maximum number of polygons per voxel (termination criterion for leaf nodes) [S2].
*   Maximum depth for grids: typically 2 or 3 [S1, unofficial].

**Traversal**
1.  Intersect ray with the root voxel AABB.
2.  If intersection fails, terminate.
3.  If the current voxel is a **Leaf**, test the ray against all primitives in the leaf's list [G].
4.  If the current voxel is **Internal**, use the hash table to locate the specific child voxel corresponding to the ray's current position [G].
5.  Recursively traverse the child voxel.
6.  Continue until a primitive intersection is found or the ray exits the structure [G].

**Pitfalls**
*   Without a good indexing strategy (such as the hash table used in JW89), it is not possible to use deep levels of recursion efficiently due to the cost of locating child nodes [S2].
*   Build cost is generally higher than general bottom-up strategies [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_grid(primitives, bounds, depth=0, max_depth=3, threshold=10):
    # [G] Standard top-down recursive subdivision
    voxel = Voxel(bounds)
    if len(primitives) <= threshold or depth >= max_depth:
        voxel.type = LEAF
        voxel.primitives = primitives
        return voxel
    voxel.type = INTERNAL
    # [G] Subdivide bounds into 2x2x2 child cells
    child_cells = split_bounds(bounds)
    child_primitives = [[] for _ in range(8)]
    for prim in primitives:
        for i, cell in enumerate(child_cells):
            if intersects(prim, cell):
                child_primitives[i].append(prim)
    # [G] Recursively build children and populate hash table
    hash_table = {}
    for i, cell_prims in enumerate(child_primitives):
        if cell_prims:
            child_bounds = child_cells[i]
            child_voxel = build_grid(child_prims, child_bounds, depth + 1, max_depth, threshold)
            hash_table[i] = child_voxel
    voxel.children = hash_table
    return voxel

def traverse_ray(ray, voxel):
    # [G] Standard ray-voxel traversal
    if not intersects(ray, voxel.bounds):
        return None
    if voxel.type == LEAF:
        return intersect_primitives(ray, voxel.primitives)
    # [G] Compute child index from ray position
    child_idx = get_child_index(ray, voxel.bounds)
    if child_idx in voxel.children:
        return traverse_ray(ray, voxel.children[child_idx])
    return None
```

## Operating characteristics
*   **Time Complexity:** Build time is higher than bottom-up strategies [S2]. Traversal cost depends on recursion depth and hash table lookup [G].
*   **Memory Complexity:** Depends on the number of internal nodes and the hash table storage for non-empty voxels [S2].
*   **Strengths:** Efficiently handles mixed-scale scenes (large regular objects + small dense objects) [S2].
*   **Limitations:** High build cost. Not well-suited for GPU computation due to branching and pointer chasing inherent in hash-table-based recursive traversal [S3, unofficial].
*   **When to use:** CPU-based ray tracing where scene geometry has significant scale variation [S2].
*   **When not to use:** GPU ray tracing (due to branching and memory access patterns) [S3, unofficial].

## Background
The method was proposed by Jevans and Wyvill in 1989 [S2]. It is described in detail in "A Survey on Exploiting Grids for Ray Tracing" [S2] and "Grids in Ray Tracing: A Comprehensive Survey" [S1, unofficial]. The approach integrates regular grid traversal with adaptive subdivision, offering a middle ground between uniform grids and octrees [S2].

## Sources
[S1] RTNv12n1.txt, https://realtimerendering.com/resources/RTNews/text/RTNv12n1.txt, fetched 2026-10-24 (unofficial)
[S2] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S3] Grids in Ray Tracing: A Comprehensive Survey, https://www.scribd.com/document/349616186/GG-400, fetched 2026-10-25 (unofficial)
