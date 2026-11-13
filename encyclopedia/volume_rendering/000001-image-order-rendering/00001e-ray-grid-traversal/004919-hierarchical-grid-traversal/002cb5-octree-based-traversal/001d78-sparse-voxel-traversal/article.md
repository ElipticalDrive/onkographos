# Sparse Voxel Traversal

Also called: Sparse Voxel Octree (SVO) Ray Intersection, Voxel Tree Raycasting

## Overview
Sparse Voxel Traversal is a hierarchical ray-intersection method that accelerates volume rendering by recursively traversing a sparse voxel octree (SVO), skipping empty regions to reduce sampling cost [S1][S2, unofficial]. It solves the performance bottleneck of dense voxel traversal by allowing the ray to take larger steps through empty space, significantly reducing the total number of intersection tests and node visits required to reach active data [S1]. Unlike flat grid traversal which checks every voxel, hierarchical traversal skips entire subtrees that do not intersect the ray or are marked inactive, enabling efficient rendering of sparse or multi-resolution datasets [S1]. It differs from its siblings by explicitly tracking spatial bounds and using recursive descent with mid-plane comparisons to select child nodes, rather than relying on fixed-step iteration or hash-based lookups [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A hierarchical voxel tree (SVO) structure, a ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and an optional bounding box or dilation count [S1][S2, S4, unofficial].
*   **Output:** A list of hit times ($t_0, t_1$) or specific leaf node data encountered by the ray [S1][S2, unofficial].

**Data Structures:**
*   **Tree Structure:** The SVO is typically a boolean tree where nodes are classified as `Empty`, `Leaf`, or `Branch` [S2, unofficial].
    *   `Empty`: Node contains no data or is fully interior/exterior. Traversal skips this node [S2, unofficial].
    *   `Leaf`: Node contains actual data (voxel values or SDF evaluation). A callback is invoked [S2, unofficial].
    *   `Branch`: Node has 8 children. The ray recursively descends into branches by computing child coordinates based on ray intersection with the branch center [S2, unofficial].
*   **VoxelBounds:** Stores coordinates as multiples of $2^{\text{lg\_size}}$ and the log2 of the voxel size [S2, unofficial].
    *   Fields: `x: i32`, `y: i32`, `z: i32`, `lg_size: i16` [S2, unofficial].
    *   World size: $2^{\text{lg\_size}}$ if $\text{lg\_size} \ge 0$, else $1.0 / 2^{-\text{lg\_size}}$ [S2, unofficial].
*   **TOI (Time of Intersection):** A wrapper for `f32` implementing `Ord` for sorting entry/exit times [S2, unofficial].
*   **Entry/Exit:** Tracks the side and TOI of ray entry/exit to determine traversal direction [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Construct the intersector with the grid and optional dilation count [S1]. Set the ray (index or world space) [S1]. If the ray misses the grid's bounding box, terminate [S1].
2.  **Recursive Traversal (`cast_ray`):**
    *   **Empty Node:** Do nothing; fall through to exit calculation [S2, unofficial].
    *   **Leaf Node:** Invoke the action callback `act(bounds, leaf)` [S2, unofficial]. If the callback returns a result, return it [S2, unofficial]. Otherwise, fall through to exit calculation [S2, unofficial].
    *   **Branch Node:**
        1.  Compute the midpoint of the current node: $\mathbf{mid} = [(\text{bounds}.x + 0.5) \cdot \text{size}, (\text{bounds}.y + 0.5) \cdot \text{size}, (\text{bounds}.z + 0.5) \cdot \text{size}]$ [S2, unofficial].
        2.  Calculate the intersection point $\mathbf{intersect}$ along the ray at the entry time $t_{entry}$ (0.0 if no entry) [S2, unofficial]:
            $$ \mathbf{intersect} = \mathbf{o} + t_{entry} \cdot \mathbf{d} $$
        3.  Determine child coordinates $\mathbf{c}$:
            $$ c_i = \begin{cases} 1 & \text{if } \text{intersect}_i \ge \text{mid}_i \\ 0 & \text{otherwise} \end{cases} $$
        4.  Recursively call `cast_ray` on the child at $\mathbf{c}$ with updated bounds [S2, unofficial].
3.  **Exit Calculation:**
    *   Define sides array: $[\text{bounds}.x, \text{bounds}.y, \text{bounds}.z, \text{bounds}.x+1, \text{bounds}.y+1, \text{bounds}.z+1]$ [S2, unofficial].
    *   For each side $s$, compute dimension $dim = s \% 3$ and bound value $B = s \cdot \text{size}$ [S2, unofficial].
    *   If $\mathbf{d}_{dim} == 0.0$, skip this side (parallel to slab) [S2, unofficial].
    *   Compute TOI: $t = (B - \mathbf{o}_{dim}) / \mathbf{d}_{dim}$ [S2, unofficial].
    *   Validate TOI: If no entry, accept if $t \ge 0.0$ [S2, unofficial]. If entry exists, accept if $t_{entry} \le t$ [S2, unofficial].
    *   Find the minimum valid TOI among all sides, excluding the entry side if an entry exists [S2, unofficial].
    *   Return `Err(Exit { side, toi })` [S2, unofficial].
4.  **Traversal Continuation:**
    *   If a child returns an exit error, update coordinates based on the exit side and direction [S2, unofficial]:
        *   If $\mathbf{d}_{dim} < 0.0$: if $\mathbf{c}_{dim} == 0$, return exit; else set $\mathbf{c}_{dim} = 0$ [S2, unofficial].
        *   If $\mathbf{d}_{dim} \ge 0.0$: if $\mathbf{c}_{dim} == 1$, return exit; else set $\mathbf{c}_{dim} = 1$ [S2, unofficial].
    *   Update entry to `Entry::from_exit(exit)` [S2, unofficial].
    *   Loop to the next sibling/child [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from dataclasses import dataclass
from typing import Optional, Callable, Any, Tuple

@dataclass
class VoxelBounds:
    x: int
    y: int
    z: int
    lg_size: int
    
    def size(self) -> float:
        if self.lg_size >= 0:
            return 2.0 ** self.lg_size
        return 1.0 / (2.0 ** (-self.lg_size))

@dataclass
class TOI:
    value: float

@dataclass
class Entry:
    side: int
    toi: TOI

@dataclass
class Exit:
    side: int
    toi: TOI

class TreeBody: pass
class Empty(TreeBody): pass
class Leaf(TreeBody):
    def __init__(self, data): self.data = data
class Branch(TreeBody):
    def __init__(self, children): self.children = children # List of 8 TreeBody

def make_bounds(coords: Tuple[int, int, int], parent_bounds: VoxelBounds) -> VoxelBounds:
    new_lg = parent_bounds.lg_size - 1
    new_x = (parent_bounds.x << 1) + coords[0]
    new_y = (parent_bounds.y << 1) + coords[1]
    new_z = (parent_bounds.z << 1) + coords[2]
    return VoxelBounds(x=new_x, y=new_y, z=new_z, lg_size=new_lg)

def cast_ray(node: TreeBody, origin: Tuple[float, float, float], 
             direction: Tuple[float, float, float], 
             bounds: VoxelBounds, 
             entry: Optional[Entry],
             act: Callable[[VoxelBounds, Any], Optional[Any]]) -> Tuple[bool, Any, Optional[Exit]]:
    """
    Returns (is_ok, result_or_None, exit_or_None)
    """
    if isinstance(node, Empty):
        pass # Fall through to exit calculation
    elif isinstance(node, Leaf):
        result = act(bounds, node.data)
        if result is not None:
            return (True, result, None)
        # Fall through to exit calculation
    elif isinstance(node, Branch):
        mid = [
            (bounds.x + 0.5) * bounds.size(),
            (bounds.y + 0.5) * bounds.size(),
            (bounds.z + 0.5) * bounds.size()
        ]
        t_entry = entry.toi.value if entry else 0.0
        intersect = [
            origin[0] + t_entry * direction[0],
            origin[1] + t_entry * direction[1],
            origin[2] + t_entry * direction[2]
        ]
        coords = [
            1 if intersect[0] >= mid[0] else 0,
            1 if intersect[1] >= mid[1] else 0,
            1 if intersect[2] >= mid[2] else 0
        ]
        child_bounds = make_bounds(coords, bounds)
        is_ok, res, exit_info = cast_ray(node.children[coords[0]][coords[1]][coords[2]], 
                                         origin, direction, child_bounds, entry, act)
        if is_ok:
            return (True, res, None)
        if exit_info is not None:
            return (False, None, exit_info)
        # If child returned no hit and no exit, fall through to exit calculation
        pass

    # Exit Calculation
    sides = [bounds.x, bounds.y, bounds.z, bounds.x + 1, bounds.y + 1, bounds.z + 1]
    min_exit = None
    
    for side_idx, bound_val in enumerate(sides):
        dim = side_idx % 3
        bound_world = bound_val * bounds.size()
        
        if direction[dim] == 0.0:
            continue
            
        toi = (bound_world - origin[dim]) / direction[dim]
        
        if entry is None:
            if toi < 0.0:
                continue
        else:
            if toi < entry.toi.value:
                continue
                
        if entry is not None and side_idx == entry.side:
            continue
            
        exit_candidate = Exit(side=side_idx, toi=TOI(toi))
        
        if min_exit is None or toi < min_exit.toi.value:
            min_exit = exit_candidate
            
    if min_exit is None:
        return (False, None, Exit(side=0, toi=TOI(float('inf'))))
        
    return (False, None, min_exit)

def traverse(root: TreeBody, origin: Tuple[float, float, float], 
             direction: Tuple[float, float, float],
             act: Callable[[VoxelBounds, Any], Optional[Any]]) -> Any:
    bounds = VoxelBounds(x=0, y=0, z=0, lg_size=0)
    coords = [0, 0, 0]
    entry = None
    
    while True:
        node = root.get(coords) if hasattr(root, 'get') else root.children[coords[0]][coords[1]][coords[2]]
        
        is_ok, res, exit_info = cast_ray(node, origin, direction, bounds, entry, act)
        
        if is_ok:
            return res
        
        if exit_info is None:
            break
            
        dim = exit_info.side % 3
        
        if direction[dim] < 0.0:
            if coords[dim] == 0:
                return None # Exited tree
            coords[dim] = 0
        else:
            if coords[dim] == 1:
                return None # Exited tree
            coords[dim] = 1
            
        entry_side = exit_info.side
        entry_toi = exit_info.toi.value
        entry = Entry(side=(entry_side + 3) % 6, toi=TOI(entry_toi))
        
        bounds = make_bounds(coords, bounds)
```

**Parameters:**
*   `dilationCount` (int, default 0): Number of voxel dilations on a boolean copy of the grid to account for interpolation kernel sizes [S1].
*   `NodeLevel` (int): The tree node level at which traversal stops and hit times are resolved (e.g., Leaf nodes) [S1].
*   `origin` (f32[3]): Ray origin in world coordinates [S2, unofficial].
*   `direction` (f32[3]): Ray direction vector [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $\mathbf{d}_{dim} == 0.0$, the TOI calculation is skipped for that side [S2, unofficial].
*   **Thread Safety:** The `ValueAccessor` is not thread-safe. Use the copy constructor to create a separate instance per computational thread [S1].
*   **Index vs World Space:** $t_0$ and $t_1$ are computed in index space. Convert to world positions using `getWorldPos(time)` or `getIndexPos(time)` [S1].
*   **Boundary Conditions:** If the ray direction is negative and coordinate is 0 (or positive and coordinate is 1), the ray exits the tree [S2, unofficial].

## Subtopics

### [Octree Traversal](1587-octree-traversal.md)
Octree Traversal is a hierarchical ray-intersection method that recursively subdivides 3D space into eight equal octants, skipping empty regions to reduce sampling cost. It solves the performance bottleneck of dense voxel traversal by allowing the ray to take larger steps through empty space, significantly reducing the total number of intersection tests and node visits required to reach active data. Unlike Empty-Space Skipping which uses primitive hulls, Octrees use hierarchical spatial subdivision to find non-empty space faster. It is distinguished from its siblings by its explicit use of regular 2x2x2 subdivision and mid-plane sorting logic for interior nodes, whereas other methods may focus purely on traversal optimization or specific data structures like hash grids.

### [Voxel Pruning](1693-voxel-pruning.md)
Voxel pruning is a technique to reduce model size and memory usage in volumetric representations by omitting voxels with low importance scores, based on the observation that a small percentage of voxels contribute the majority of rendering importance. It solves the problem of redundant storage and computation in sparse voxel grids or neural feature volumes. Unlike [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) which focuses on efficient traversal of existing data, voxel pruning actively modifies the data structure by removing non-essential voxels. It differs from [LightGaussian Pruning Scoring](3213-lightgaussian-pruning-scoring.md) by operating in volumetric space rather than image space to preserve structural fidelity.

### [DeepBrick](2191-deepbrick.md)
DeepBrick is a ray-marching kernel for deep volume raytracing that samples volumetric data, handling empty space skipping and alpha accumulation. It builds on [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) [G].

### [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md)
Voxel Grid Raycasting is a rendering technique that stores scene geometry in a 3D voxel grid and traces rays through the grid by stepping from voxel to voxel to determine intersections and surface normals. It solves the problem of efficiently determining ray-voxel intersections without performing explicit geometric intersection tests against primitives. Unlike [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) which uses hierarchical structures like octrees to skip empty space, this method assumes a uniform grid and steps through every voxel boundary along the ray path. It differs from [Regular Grid Traversal](5939-regular-grid-traversal.md) in that it typically renders volumetric density or solid occupancy directly, whereas the latter accelerates geometric ray tracing by storing primitive pointers.

### [Cap Node Traversal](3003-cap-node-traversal.md)
Cap Node Traversal is a specialized ray traversal routine for "cap nodes" in sparse voxel octrees, which are leaf-like structures containing scalar data for a $2 \times 2 \times 2$ block of voxels. It solves the problem of efficiently retrieving voxel values and performing isosurface intersection without the overhead of recursive descent into child nodes. It differs from standard interior node traversal by replacing the isovalue range check and recursive child call with direct scalar retrieval and local neighbor finding.

### [Sparse Voxel Ray Marching with AABB Exit Planes](3252-sparse-voxel-ray-marching-with-aabb-exit-planes.md)
Sparse Voxel Ray Marching with AABB Exit Planes is a ray traversal algorithm for sparse voxel hierarchies that skips empty space by computing the exit plane of the current empty cell using precomputed Axis-Aligned Distance Fields (AADF). It differs from standard DDA [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md) by avoiding per-voxel checks in empty regions, and from pure octree traversal [Octree Traversal](1587-octree-traversal.md) by using fixed-size hierarchical chunks (chunks, bricks, voxels) with texture-based occupancy masks and distance fields rather than dynamic node pointers.

### [Voxel Ray Traversal (AABB-optimized)](3255-voxel-ray-traversal-aabb-optimized.md)
Voxel Ray Traversal (AABB-optimized) is an iterator that traverses a 3D voxel volume along a ray, returning hits for voxels contained within the volume. It solves the problem of efficiently enumerating voxels by combining a standard 3D DDA traversal with an initial Axis-Aligned Bounding Box (AABB) rejection test. What sets it apart from its siblings, such as [Octree Traversal](1587-octree-traversal.md) or [Sparse Voxel Ray Marching with AABB Exit Planes](3252-sparse-voxel-ray-marching-with-aabb-exit-planes.md), is its reliance on a flat or chunk-based volume structure where the AABB test serves primarily as a "jump ahead" mechanism to skip empty space before the voxel-by-voxel iteration begins, rather than using hierarchical node structures for traversal.

### [Hierarchical Brick-Map DDA](4105-hierarchical-brick-map-dda.md)
Hierarchical Brick-Map DDA is a ray traversal technique that replaces dense voxel buffers with a sparse, hierarchical structure to skip empty regions in volumetric data. It operates by maintaining a top-level grid of cells (typically stride 8 voxels) that point to allocated "bricks" containing fine-grained voxel data; if a top-level cell is empty, the ray strides over the entire cell volume, descending only into occupied bricks for local DDA. This approach significantly reduces memory footprint and traversal time for sparse volumes, such as drone airspace which is >99% empty, compared to dense voxel storage or non-hierarchical sparse methods.

### Sparse Ray Marching for RTAO
Sparse Ray Marching for RTAO is a variant of sparse ray marching used for Ray Traced Ambient Occlusion [S5, unofficial]. It marches from a surface point in a hemisphere direction, counting occlusions up to a radius limit [S5, unofficial]. Unlike the main sparse ray march, this version does not use the hierarchical chunk/brick structure [S5, unofficial]. It only checks voxel_masks and voxel_aadf [S5, unofficial]. It also accumulates distance and stops at t_max [S5, unofficial].

### Robust Interval Arithmetic Ray Traversal
Robust Interval Arithmetic Ray Traversal is a ray traversal method that uses interval arithmetic with correct rounding modes to guarantee that no octant is missed during traversal, solving precision errors that occur when subdivision levels approach floating-point limits [S6]. Unlike Gargantini's method which only partially eliminates errors and requires restricted conditions, this method guarantees correctness for all cases with minimal performance impact [S6].

### Hybrid Ray Tracing and Path Tracing
Hybrid Ray Tracing and Path Tracing is a hybrid rendering technique using the CGC data structure where cell rasterisation handles primary rays and path tracing handles secondary rays for global illumination [S7]. Using the CGC data structure for ray tracing or path tracing of isosurfaces, leveraging pre-computed cluster bounds for hierarchical traversal [S7].

## Operating characteristics
*   **Performance:** Efficient for sparse volumes [S1]. Performance degrades if the volume is dense, as the overhead of tree traversal may exceed simple grid traversal [S1].
*   **Strengths:** Significant speedup for sparse data; supports multi-resolution; handles large bounding boxes efficiently [S1].
*   **Limitations:** Higher memory overhead than dense grids; complex implementation; potential precision issues at high subdivision levels (mitigated by interval arithmetic) [S2, unofficial][S6].
*   **When to Use:** Sparse volumetric data, large scenes with empty space, multi-resolution datasets [S1].
*   **When Not to Use:** Dense volumes where the overhead of tree traversal outweighs the benefits; simple uniform grids [S1].

## Background
Sparse Voxel Traversal is used in OpenVDB v13.1.0 for ray marching through density or generic volumes [S1]. It is based on the principles of hierarchical voxel trees, where explicit bounds tracking and entry/exit side tracking allow for efficient skipping of empty branches [S2, unofficial]. The method distinguishes itself from simple grid traversal by using a callback for leaf processing rather than fixed-step iteration [S2, unofficial].

## Sources
[S1] OpenVDB: VolumeRayIntersector< GridT, NodeLevel, RayT > Class Template ..., https://www.openvdb.org/documentation/doxygen/classopenvdb_1_1v13__1_1_1tools_1_1VolumeRayIntersector.html, fetched 2026-10-26
[S2] bfops/bit-svo, https://github.com/bfops/bit-svo, fetched 2026-10-26 (unofficial)
[S3] | Evolve, https://www.evolvebenchmark.com/blog-posts/intro-to-rendering-of-animated-volumetric-models-in-breda, fetched 2026-10-27 (unofficial)
[S4] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S5] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
[S6] Arbitrary 3D Resolution Discrete Ray Tracing of Implicit Surfaces, https://doi.org/10.1007/978-3-540-31965-8_38, fetched 2026-10-24
[S7] Clustered Grid Cell Data Structure for Isosurface Rendering, https://doi.org/10.24132/jwscg.2020.28.2, fetched 2026-10-26
