# Sparse Voxel Traversal

Also called: OpenVDB Ray Intersector, Sparse Voxel Traversal and Sampling, Structure sampling, VolumeRayIntersector, Voxel Tree Raycasting, getVoxel

## Overview
Sparse Voxel Traversal is a hierarchical ray-intersection algorithm that accelerates volume rendering by navigating a spatial hierarchy (such as a Sparse Voxel Octree) to skip empty space and reduce the number of ray-casting steps [S2, unofficial][S3, unofficial]. It solves the performance bottleneck of non-hierarchical voxel traversal, where every voxel along a ray must be visited, by allowing the ray to leap over large empty regions [S2, unofficial]. Unlike its siblings, it operates on a recursive tree structure where each parent node divides into eight equal sub-blocks ($2 \times 2 \times 2$ subdivision), enabling larger steps through empty space [S2, unofficial][S3, unofficial]. It distinguishes itself from standard grid traversal by using interval arithmetic with correct rounding modes to guarantee that no octant is missed during traversal, solving precision errors that occur when subdivision levels approach floating-point limits [S5].

## How it works
**Inputs and Outputs:**
*   **Input:** A hierarchical voxel tree (e.g., VoxelTree, OpenVDB Grid) containing nodes of types `Empty`, `Leaf`, or `Branch` [S3, unofficial]. A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [S3, unofficial].
*   **Output:** Intersection times (TOI) or a callback result from leaf nodes [S2, unofficial].

**Data Structures:**
*   **VoxelBounds:** Struct with fields `x: i32`, `y: i32`, `z: i32` (coordinates as multiples of $2^{\text{lg\_size}}$) and `lg_size: i16` (log_2 of voxel size) [S2, unofficial]. The size in world units is $2^{\text{lg\_size}}$ if $\text{lg\_size} \ge 0$, else $1.0 / 2^{-\text{lg\_size}}$ [S2, unofficial].
*   **TOI:** A wrapper around `f32` implementing `Ord` for sorting intersection times [S2, unofficial].
*   **Entry/Exit:** Structures tracking the side index (0-5) and TOI of ray entry/exit [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Construct the intersector with the grid and optional dilation count [S1]. Set the ray using `setIndexRay` (index space) or `setWorldRay` (world space) [S1]. If the ray misses the grid's bounding box, return false [S1].
2.  **Traversal Loop:** Recursively traverse the tree starting from the root [S2, unofficial].
    *   **Empty Node:** If the current node is `Empty`, do nothing and fall through to calculate the exit [S2, unofficial].
    *   **Leaf Node:** If the current node is `Leaf`, call the action callback `act(bounds, leaf)` [S2, unofficial]. If it returns a hit, return `Ok(result)` [S2, unofficial]. Otherwise, fall through to calculate the exit [S2, unofficial].
    *   **Branch Node:** If the current node is `Branch`:
        1.  Compute the midpoint of the current bounds: $\mathbf{mid} = [(\text{bounds}.x + 0.5) \cdot \text{size}, (\text{bounds}.y + 0.5) \cdot \text{size}, (\text{bounds}.z + 0.5) \cdot \text{size}]$ [S2, unofficial].
        2.  Calculate the intersection point $\mathbf{intersect}$ at the current entry time: $\mathbf{intersect} = \mathbf{o} + \text{entry\_toi} \cdot \mathbf{d}$, where $\text{entry\_toi}$ is the TOI of the entry (0.0 if none) [S2, unofficial].
        3.  Determine child coordinates: $\text{coords}[i] = 1$ if $\text{intersect}[i] \ge \text{mid}[i]$, else $0$ [S2, unofficial].
        4.  Update bounds for the child: shift coordinates, decrement `lg_size`, and add child coordinates [S2, unofficial].
        5.  Recursively call `cast_ray_branches` with the child branch, origin, direction, entry, new coords, and updated bounds [S2, unofficial].
3.  **Exit Calculation:** If no return value is found in the node (Empty or Leaf without hit):
    1.  Define sides array: $[\text{bounds}.x, \text{bounds}.y, \text{bounds}.z, \text{bounds}.x + 1, \text{bounds}.y + 1, \text{bounds}.z + 1]$ [S2, unofficial].
    2.  For each side, compute the time of intersection (TOI):
        *   Get dimension $\text{dim} = \text{side} \% 3$ and bound value $b = \text{bound} \cdot \text{bounds.size()}$ [S2, unofficial].
        *   If $\mathbf{d}[\text{dim}] == 0.0$, skip that side [S2, unofficial].
        *   Compute $\text{toi} = (b - \mathbf{o}[\text{dim}]) / \mathbf{d}[\text{dim}]$ [S2, unofficial].
        *   Accept if: if entry is `None`, $\text{toi} \ge 0.0$; if entry is `Some`, $\text{entry.toi} \le \text{toi}$ [S2, unofficial].
    3.  Find the minimum valid TOI among the sides. If entry exists, exclude the entry side [S2, unofficial].
    4.  Return `Err(exit)` with the minimum TOI [S2, unofficial].
4.  **Coordinate Update (for Branches):** If `cast_ray` returns `Err(exit)`:
    1.  Calculate dimension $\text{dim} = \text{exit.side} \% 3$ [S2, unofficial].
    2.  If $\mathbf{d}[\text{dim}] < 0.0$:
        *   If $\text{coords}[\text{dim}] == 0$, return the exit error (ray exited tree boundary) [S2, unofficial].
        *   Otherwise, set $\text{coords}[\text{dim}] = 0$ [S2, unofficial].
    3.  If $\mathbf{d}[\text{dim}] \ge 0.0$:
        *   If $\text{coords}[\text{dim}] == 1$, return the exit error [S2, unofficial].
        *   Otherwise, set $\text{coords}[\text{dim}] = 1$ [S2, unofficial].
    4.  Update entry: $\text{entry} = \text{Entry::from\_exit}(\text{exit})$, which maps the exit side to the opposite face [S2, unofficial].
    5.  Loop to the next child [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelBounds:
    x: int
    y: int
    z: int
    lg_size: int
    def size(self) -> float:
        return 2.0 ** self.lg_size if self.lg_size >= 0 else 1.0 / (2.0 ** (-self.lg_size))

class TOI:
    val: float
    def __lt__(self, other): return self.val < other.val

class Entry:
    side: int
    toi: TOI
    @staticmethod
    def from_exit(exit):
        return Entry(exit.side + 3 if exit.side < 3 else exit.side - 3, exit.toi)

class Exit:
    side: int
    toi: TOI

def cast_ray(node, origin, direction, bounds, entry, act):
    if node.type == 'Empty':
        pass
    elif node.type == 'Leaf':
        result = act(bounds, node.data)
        if result is not None:
            return ('Ok', result)
    elif node.type == 'Branch':
        mid = [(bounds.x + 0.5) * bounds.size(), (bounds.y + 0.5) * bounds.size(), (bounds.z + 0.5) * bounds.size()]
        entry_toi = entry.toi.val if entry else 0.0
        intersect = [origin[i] + entry_toi * direction[i] for i in range(3)]
        child_coords = [1 if intersect[i] >= mid[i] else 0 for i in range(3)]
        child_bounds = VoxelBounds(
            x=(bounds.x << 1) + child_coords[0],
            y=(bounds.y << 1) + child_coords[1],
            z=(bounds.z << 1) + child_coords[2],
            lg_size=bounds.lg_size - 1
        )
        return cast_ray_branches(node.child, origin, direction, child_bounds, entry, child_coords, act)

    sides = [bounds.x, bounds.y, bounds.z, bounds.x + 1, bounds.y + 1, bounds.z + 1]
    valid_exits = []
    for side, bound in enumerate(sides):
        dim = side % 3
        if direction[dim] == 0.0:
            continue
        bound_val = bound * bounds.size()
        toi = (bound_val - origin[dim]) / direction[dim]
        if entry is None:
            if toi >= 0.0:
                valid_exits.append(Exit(side, TOI(toi)))
        else:
            if entry.toi.val <= toi:
                valid_exits.append(Exit(side, TOI(toi)))

    if not valid_exits:
        return ('Err', Exit(0, TOI(float('inf'))))

    min_exit = None
    for exit in valid_exits:
        if entry is not None and exit.side == entry.side:
            continue
        if min_exit is None or exit.toi < min_exit.toi:
            min_exit = exit
    return ('Err', min_exit)

def cast_ray_branches(node, origin, direction, bounds, entry, coords, act):
    while True:
        child = node.get(coords[0], coords[1], coords[2])
        res = cast_ray(child, origin, direction, bounds, entry, act)
        status, val = res
        if status == 'Ok':
            return res
        exit = val
        dim = exit.side % 3
        if direction[dim] < 0.0:
            if coords[dim] == 0:
                return res
            coords[dim] = 0
        else:
            if coords[dim] == 1:
                return res
            coords[dim] = 1
        entry = Entry.from_exit(exit)
```

**Parameters:**
*   **Precision:** 30 bits (NL=30) [S5].
*   **Resolution:** Octree level 9 ($512^3$ resolution) [S5].
*   **Dilation Count:** Number of voxel dilations on a boolean copy of the input grid to account for interpolation kernel sizes (default 0) [S1].
*   **NodeLevel:** The tree node level at which traversal stops and hit times are resolved (default: Leaf) [S1].

**Edge Cases and Pitfalls:**
*   **Precision Errors:** When subdivision levels approach floating-point limits, standard traversals may mismatch [S5]. This is handled by using interval arithmetic with correct rounding modes (SSE2) to guarantee correctness [S5].
*   **Division by Zero:** If $\mathbf{d}[\text{dim}] == 0.0$, the TOI calculation is skipped for that side [S2, unofficial].
*   **Thread Safety:** The `ValueAccessor` is not thread-safe. A copy constructor must be used to create a separate instance per computational thread [S1].
*   **Index vs World Space:** Hit times $t_0$ and $t_1$ are computed in index space. Client code must use `getIndexPos` or `getWorldPos` to convert [S1].

## Subtopics

### [Octree Traversal](1587-octree-traversal.md)
Octree Traversal is a hierarchical ray-intersection algorithm that accelerates volume rendering by navigating a spatial hierarchy to skip empty space and reduce the number of ray-casting steps. It solves the performance bottleneck of non-hierarchical voxel traversal, where every voxel along a ray must be visited, by allowing the ray to leap over large empty regions. Unlike its siblings, it operates on a recursive tree structure where each parent node divides into eight equal sub-blocks ($2 \times 2 \times 2$ subdivision), enabling larger steps through empty space.

### [Voxel Pruning](1693-voxel-pruning.md)
Voxel Pruning is a model compression technique for volumetric representations that reduces storage size by omitting voxels with low rendering importance scores. It operates on the observation that a small percentage of voxels contribute the majority of rendering importance, while the majority have minimal effect on the final image. Unlike sibling methods that focus on traversal acceleration (e.g., [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md)) or structural optimization (e.g., [DeepBrick](2191-deepbrick.md)), Voxel Pruning directly modifies the data structure by removing inactive entries based on a learned importance metric.

### [DeepBrick](2191-deepbrick.md)
DeepBrick is a ray-marching kernel for deep volume raytracing that samples volumetric data stored in hierarchical "brick" structures, handling empty space skipping and alpha accumulation. It differs from standard sparse voxel traversal by operating on pre-integrated brick textures and adjusting sampling density based on a "sample ratio" relative to the brick's internal resolution, allowing for efficient rendering of deep volumes with variable step sizes. It builds on [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md).

### [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md)
Voxel Grid Raycasting is a rendering technique that stores scene geometry in a 3D voxel grid and traces rays through the grid by stepping from voxel to voxel, checking for occupied voxels to determine intersections and surface normals. It solves the problem of efficiently determining ray-voxel intersections without explicit geometric intersection tests. It is distinct from [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) by assuming a dense, regular grid layout rather than a sparse octree or brick-based structure.

### [Cap Node Traversal](3003-cap-node-traversal.md)
Cap Node Traversal is a specialized ray traversal routine for "cap nodes" in sparse voxel octrees, which are leaf-like structures containing scalar data for a $2 \times 2 \times 2$ block of voxels. It solves the problem of efficiently retrieving voxel values and performing isosurface intersection without the overhead of recursive descent into child nodes. It differs from standard interior node traversal by replacing the isovalue range check and recursive child call with direct scalar retrieval and local neighbor finding. It is distinct from [Scalar Leaves](1525-sparse-voxel-traversal.md) (which have homogeneous values) and [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) (which handles general sparse structures) by specifically optimizing the $2 \times 2 \times 2$ boundary case.

### [Sparse Voxel Ray Marching with AABB Exit Planes](3252-sparse-voxel-ray-marching-with-aabb-exit-planes.md)
Sparse Voxel Ray Marching with AABB Exit Planes is a ray traversal algorithm for sparse voxel hierarchies that skips empty space by computing the exit plane of the current empty cell using precomputed Axis-Aligned Distance Fields (AADF). It differs from standard DDA [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md) by avoiding per-voxel checks in empty regions, and from pure octree traversal [Octree Traversal](1587-octree-traversal.md) by using fixed-size hierarchical chunks (chunks, bricks, voxels) with texture-based occupancy masks and distance fields rather than dynamic node pointers.

### [Voxel Ray Traversal (AABB-optimized)](3255-voxel-ray-traversal-aabb-optimized.md)
Voxel Ray Traversal (AABB-optimized) is an iterator-based algorithm that traverses a 3D voxel volume along a ray, returning hits for voxels contained within the volume. It solves the problem of efficiently enumerating occupied voxels in sparse volumes by combining a standard 3D DDA (Digital Differential Analyzer) traversal with an initial Axis-Aligned Bounding Box (AABB) rejection test. What sets it apart from its siblings, such as [Octree Traversal](1587-octree-traversal.md) or [Sparse Voxel Ray Marching with AABB Exit Planes](3252-sparse-voxel-ray-marching-with-aabb-exit-planes.md), is its reliance on a flat or chunk-based volume structure where the AABB test serves primarily as a "jump ahead" mechanism to skip empty space before the voxel-by-voxel iteration begins, rather than using hierarchical node structures for traversal.

### [Hierarchical Brick-Map DDA](4105-hierarchical-brick-map-dda.md)
A ray marching technique that replaces a dense voxel buffer with a sparse hierarchical structure (brick-map) to skip empty regions. It uses a top-level grid of cells (stride 8 voxels) to quickly identify occupied bricks, then descends into those bricks for fine-grained DDA. This reduces memory footprint and traversal time in sparse volumes like drone airspace (>99% empty).

### [Sparse Voxel Grid Raycasting with Occupancy Grid](4152-sparse-voxel-grid-raycasting-with-occupancy-grid.md)
A volume rendering estimator that accelerates ray traversal and sampling by using a hierarchical sparse voxel grid (often implemented via VDB/NanoVDB structures) to skip empty space along rays, returning only sample locations within occupied cells. It differs from standard [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md) by replacing dense texture lookups with a hierarchical occupancy check that reduces texture reads by a factor of 3, and from [Octree Traversal](1587-octree-traversal.md) by using a fixed-resolution sparse grid with hierarchical levels rather than a dynamic tree structure for the primary traversal logic.

### Sparse Ray Marching for RTAO
A variant of sparse ray marching used for Ray Traced Ambient Occlusion. It marches from a surface point in a hemisphere direction, counting occlusions up to a radius limit [S4, unofficial]. It starts at the floor of the ray origin, initializes local position and distance to 0, and loops up to $3 \times \text{uniforms.grid\_ext}$ times [S4, unofficial]. It checks voxel occupancy using `voxel_masks`; if occupied, it returns true [S4, unofficial]. It loads the voxel AADF and calculates the exit plane, then advances the ray position [S4, unofficial]. It accumulates distance and stops at `t_max` [S4, unofficial]. Unlike the main sparse ray march, this version does not use the hierarchical chunk/brick structure, only checking `voxel_masks` and `voxel_aadf` [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Logarithmic with respect to the depth of the octree, $O(\log N)$, where $N$ is the number of voxels, due to skipping empty branches [S1][S3, unofficial].
*   **Memory Complexity:** Proportional to the number of non-empty nodes in the sparse voxel tree [S2, unofficial].
*   **Performance:** On a PC with a Pentium-4 2.4GHz processor and 512Mbytes of main memory, generating $512 \times 512$ images for various surfaces (e.g., asterisk(9,18)) took approximately 410-445 seconds using the algorithm with SSE2 rounding modes, comparable to Gargantini's algorithm [S5]. The version without rounding modes was several seconds faster [S5].
*   **Strengths:** Guarantees correctness for all cases with minimal performance impact by using interval arithmetic with correct rounding modes [S5]. Efficiently skips empty space in sparse volumes [S3, unofficial].
*   **Limitations:** Requires careful handling of floating-point precision at high subdivision levels [S5]. The `ValueAccessor` is not thread-safe, requiring per-thread instances [S1].
*   **When to Use:** High-resolution sparse volumes where empty space skipping is critical and precision errors are a concern [S1][S2, unofficial].
*   **When Not to Use:** Dense volumes where the overhead of tree traversal outweighs the benefit of skipping empty space, or when hardware does not support SSE2 instructions for interval arithmetic [S5].

## Background
Sparse Voxel Traversal originated from the need to efficiently render high-resolution implicit surfaces and volumetric data. The method using interval arithmetic with correct rounding modes was introduced to solve precision errors in Gargantini's algorithm, which only partially eliminated errors under restricted conditions [S5]. The implementation in OpenVDB (VolumeRayIntersector) is used in OpenVDB v13.1.0 for ray marching through density or generic volumes [S2, unofficial]. The 5-4-3 node structure (Node5, Node4, Node3) is commonly used in game industry rendering for VDB data, as seen in benchmarks like Evolve [S3, unofficial]. The variant for Ray Traced Ambient Occlusion (RTAO) is implemented in real-time path tracing pipelines [S4, unofficial].

## Sources
[S1] OpenVDB: VolumeRayIntersector< GridT, NodeLevel, RayT > Class Template ..., https://www.openvdb.org/documentation/doxygen/classopenvdb_1_1v13__1_1_1tools_1_1VolumeRayIntersector.html, fetched 2026-10-26
[S2] bfops/bit-svo, https://github.com/bfops/bit-svo, fetched 2026-10-26 (unofficial)
[S3] | Evolve, https://www.evolvebenchmark.com/blog-posts/intro-to-rendering-of-animated-volumetric-models-in-breda, fetched 2026-10-27 (unofficial)
[S4] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
[S5] Arbitrary 3D Resolution Discrete Ray Tracing of Implicit Surfaces, https://doi.org/10.1007/978-3-540-31965-8_38, fetched 2026-10-24
