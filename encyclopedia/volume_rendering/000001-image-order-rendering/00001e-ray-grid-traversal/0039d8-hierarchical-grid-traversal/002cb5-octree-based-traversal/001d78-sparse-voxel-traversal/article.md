# Sparse Voxel Traversal

Also called: OpenVDB Ray Intersector, Sparse Voxel Traversal and Sampling, Structure sampling, VolumeRayIntersector, Voxel Tree Raycasting, getVoxel

## Overview
Sparse Voxel Traversal is a hierarchical ray-intersection method that accelerates volume rendering by recursively traversing a sparse voxel tree (such as an Octree or VoxelTree) to find intersections with leaf nodes or branches, skipping empty space efficiently [S1][S2, unofficial]. It differs from flat grid traversal by using a tree structure to bypass large volumes of empty space, and from Empty-Space Skipping which uses primitive hulls, by employing hierarchical spatial subdivision to locate non-empty space faster [S1]. It sets itself apart from its siblings, such as [Octree Partitioning for Sphere Tracing](1541-octree-partitioning-for-sphere-tracing.md), by using explicit bounds tracking and entry/exit side tracking with a callback for leaf processing rather than fixed-step iteration [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A hierarchical voxel tree (GridT) containing `Empty`, `Branch`, and `Leaf` nodes; a ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; and optional parameters for dilation and node level [S1][S2, unofficial].
*   **Output:** A boolean indicating intersection with active values, and entry/exit times ($t_0, t_1$) along the ray, or a specific hit result via a callback [S1][S2, unofficial].

**Data Structures:**
*   **TreeT:** A boolean tree used to accelerate ray marching, defined as `tree::Tree<typename RootType::template ValueConverter<bool>::Type>` [S1].
*   **VoxelBounds:** Struct with fields `x: i32`, `y: i32`, `z: i32` (coordinates as multiples of $2^{\text{lg\_size}}$) and `lg_size: i16` (log_2 of voxel size) [S2, unofficial]. Size in world units is $2^{\text{lg\_size}}$ if $\text{lg\_size} \ge 0$, else $1.0 / 2^{-\text{lg\_size}}$ [S2, unofficial].
*   **TOI:** A wrapper for `f32` implementing `Ord` for sorting intersection times [S2, unofficial].
*   **Entry/Exit:** Structures tracking the side index (0-5) and Time of Intersection (TOI) when entering or leaving a voxel [S2, unofficial].

**Steps:**
1.  **Initialization:** Construct the intersector with the grid and optional `dilationCount` (default 0) to account for interpolation kernel sizes [S1]. Throw `RuntimeError` if voxels are non-uniform or the grid is empty [S1].
2.  **Ray Setup:** Set the ray using `setIndexRay` (index space) or `setWorldRay` (world space). Returns `false` if the ray misses the grid's bounding box [S1].
3.  **Traversal Loop (`cast_ray_branches`):**
    *   Get the child node at current coordinates `coords` via `this.get(coords[0], coords[1], coords[2])` [S2, unofficial].
    *   Compute bounds via `make_bounds(coords)` [S2, unofficial].
    *   Call `cast_ray` on the child [S2, unofficial].
    *   If `cast_ray` returns `Ok(r)`, return immediately [S2, unofficial].
    *   If `cast_ray` returns `Err(exit)`:
        *   Calculate dimension `dim = exit.side % 3` [S2, unofficial].
        *   If `direction[dim] < 0.0`: check if `coords[dim] == 0`; if so, return the exit error. Otherwise, set `coords[dim] = 0` [S2, unofficial].
        *   If `direction[dim] >= 0.0`: check if `coords[dim] == 1`; if so, return the exit error. Otherwise, set `coords[dim] = 1` [S2, unofficial].
        *   Update `entry` to `Entry::from_exit(exit)`, which maps the exit side to the opposite entry side (e.g., side 0 becomes 3) [S2, unofficial].
4.  **Node Processing (`cast_ray`):**
    *   **Empty:** Do nothing and fall through to exit calculation [S2, unofficial].
    *   **Leaf:** Call the action callback `act(bounds, leaf)`. If it returns `Some(r)`, return `Ok(r)`. Otherwise, fall through to calculate the exit [S2, unofficial].
    *   **Branch:**
        *   Compute midpoint $\mathbf{mid} = [(\text{bounds}.x + 0.5) \cdot \text{size}, (\text{bounds}.y + 0.5) \cdot \text{size}, (\text{bounds}.z + 0.5) \cdot \text{size}]$ [S2, unofficial].
        *   Calculate intersection point $\mathbf{intersect} = \mathbf{o} + \text{entry\_toi} \cdot \mathbf{d}$, where $\text{entry\_toi}$ is the entry TOI or 0.0 if none [S2, unofficial].
        *   Determine child coordinates: $\text{coords}[i] = 1$ if $\mathbf{intersect}[i] \ge \text{mid}[i]$, else $0$ [S2, unofficial].
        *   Recursively call `cast_ray_branches` with the new coordinates and updated bounds (shifting and adding coordinates) [S2, unofficial].
5.  **Exit Calculation:**
    *   Define sides array $[\text{bounds}.x, \text{bounds}.y, \text{bounds}.z, \text{bounds}.x+1, \text{bounds}.y+1, \text{bounds}.z+1]$ [S2, unofficial].
    *   For each side, compute dimension `dim = side % 3` and bound value `bound = side_val * bounds.size()` [S2, unofficial].
    *   If `direction[dim] == 0.0`, return `None` for that side [S2, unofficial].
    *   Compute $\text{toi} = \frac{\text{bound} - \mathbf{o}[\text{dim}]}{\mathbf{d}[\text{dim}]}$ [S2, unofficial].
    *   Accept if: if `entry` is `None`, $\text{toi} \ge 0.0$; if `entry` is `Some`, $\text{entry.toi} \le \text{toi}$ [S2, unofficial].
    *   Find the minimum valid TOI among accepted exits. If `entry` is `Some`, exclude the entry side ($i \neq \text{entry.side}$) before finding the minimum [S2, unofficial].
    *   Return `Err(exit)` with the minimum TOI [S2, unofficial].
6.  **Hit Resolution:** The `march(t0, t1)` function updates $t_0$ and $t_1$ with entry and exit times along the INDEX ray. These are resolved at the specified `NodeLevel` (e.g., leaf nodes), not individual active voxels [S1]. Convert index times to world positions using `getWorldPos(time)` if needed [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cast_ray_branches(this, origin, direction, entry, coords, make_bounds, act):
    """
    Recursively traverse branch nodes.
    this: Branches object
    origin: [f32, f32, f32]
    direction: [f32, f32, f32]
    entry: Optional[Entry]
    coords: [usize, usize, usize]
    make_bounds: function taking coords -> VoxelBounds
    act: function taking bounds, leaf -> Optional[Result]
    """
    while True:
        child = this.get(coords[0], coords[1], coords[2])
        bounds = make_bounds(coords)
        
        result = cast_ray(child, origin, direction, bounds, entry, act)
        
        if result.is_ok():
            return result
        
        exit = result.err()
        dim = exit.side % 3
        
        if direction[dim] < 0.0:
            if coords[dim] == 0:
                return Err(exit)
            coords[dim] = 0
        else:
            if coords[dim] == 1:
                return Err(exit)
            coords[dim] = 1
            
        entry = Entry.from_exit(exit)

def cast_ray(this, origin, direction, bounds, entry, act):
    """
    Process a single node (Empty, Leaf, or Branch).
    """
    if this is Empty:
        pass # Fall through to exit calculation
    elif this is Leaf:
        result = act(bounds, this.leaf)
        if result is not None:
            return Ok(result)
        # Fall through to exit calculation
    elif this is Branch:
        size = bounds.size()
        mid = [
            (bounds.x + 0.5) * size,
            (bounds.y + 0.5) * size,
            (bounds.z + 0.5) * size
        ]
        
        entry_toi = entry.toi if entry else 0.0
        intersect = [
            origin[0] + entry_toi * direction[0],
            origin[1] + entry_toi * direction[1],
            origin[2] + entry_toi * direction[2]
        ]
        
        new_coords = [
            1 if intersect[0] >= mid[0] else 0,
            1 if intersect[1] >= mid[1] else 0,
            1 if intersect[2] >= mid[2] else 0
        ]
        
        # Update bounds for child
        new_bounds = make_bounds(new_coords)
        
        return cast_ray_branches(this.branch, origin, direction, entry, new_coords, make_bounds, act)

    # Calculate exit
    sides = [bounds.x, bounds.y, bounds.z, bounds.x + 1, bounds.y + 1, bounds.z + 1]
    valid_exits = []
    
    for side_idx, side_val in enumerate(sides):
        dim = side_idx % 3
        bound_val = side_val * bounds.size()
        
        if direction[dim] == 0.0:
            continue
            
        toi = (bound_val - origin[dim]) / direction[dim]
        
        # Check validity against entry
        if entry is None:
            if toi < 0.0:
                continue
        else:
            if toi < entry.toi:
                continue
                
        valid_exits.append(Exit(side=side_idx, toi=toi))
        
    if not valid_exits:
        return Err(None) # Should not happen if ray is inside
        
    # Find minimum TOI, excluding entry side if entry exists
    if entry is not None:
        valid_exits = [e for e in valid_exits if e.side != entry.side]
        
    if not valid_exits:
        return Err(None)
        
    exit = min(valid_exits, key=lambda e: e.toi)
    return Err(exit)
```

**Parameters:**
*   `dilationCount` (int, default 0): Number of voxel dilations performed on a boolean copy of the input grid to account for interpolation kernel sizes [S1].
*   `NodeLevel` (int, default `GridT::TreeType::RootNodeType::ChildNodeType::LEVEL`): The tree node level at which traversal stops and hit times are resolved [S1].
*   `origin` (Ray origin in world coordinates, type `[f32; 3]`) [S2, unofficial].
*   `direction` (Ray direction vector, type `[f32; 3]`) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** When subdivision levels approach floating-point limits, standard traversals may mismatch. Use interval arithmetic with correct rounding modes to guarantee no octant is missed [S5].
*   **Division by Zero:** If `direction[dim] == 0.0`, the TOI calculation returns `None` for that side, avoiding division by zero [S2, unofficial].
*   **Thread Safety:** The `ValueAccessor` is not thread-safe. Use the copy constructor to create a separate instance per computational thread [S1].
*   **Index vs World Space:** $t_0$ and $t_1$ are computed with respect to the index ray. Client code must use `getIndexPos` or maintain an index ray instance [S1].

## Subtopics

### [Octree Traversal](1587-octree-traversal.md)
Octree Traversal is a hierarchical ray-intersection method that accelerates volume rendering by recursively subdividing 3D space into eight equal octants, allowing rays to skip empty regions without sampling. It differs from flat grid traversal by using a tree structure to bypass large volumes of empty space, and from Empty-Space Skipping which uses primitive hulls, by employing hierarchical spatial subdivision to locate non-empty space faster. It is theoretically optimal for the fewest traversal steps when objects are uniformly contained within cells with no overlap.

### [Voxel Pruning](1693-voxel-pruning.md)
Voxel Pruning is a model compression technique for volumetric representations that reduces storage size by omitting voxels with low rendering importance scores. It operates on the observation that a small percentage of voxels contribute the majority of rendering importance, while the majority have minimal effect on the final image. Unlike sibling methods that focus on traversal acceleration (e.g., [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md)) or structural optimization (e.g., [DeepBrick](2191-deepbrick.md)), Voxel Pruning directly modifies the data structure by removing inactive entries based on a learned importance metric.

### [DeepBrick](2191-deepbrick.md)
DeepBrick is a ray-marching kernel for deep volume raytracing that samples volumetric data, handling empty space skipping and alpha accumulation. It builds on [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) [G].

### [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md)
Voxel Grid Raycasting is a rendering technique that stores scene geometry in a 3D voxel grid and traces rays through the grid by stepping from voxel to voxel, checking for occupied voxels to determine intersections and surface normals. It solves the problem of efficiently determining ray-voxel intersections without explicit geometric intersection tests. It is distinct from [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) by assuming a dense, regular grid layout rather than a sparse octree or brick-based structure.

### [Cap Node Traversal](3003-cap-node-traversal.md)
Cap Node Traversal is a specialized ray traversal routine for "cap nodes" in sparse voxel octrees, which are leaf-like structures containing scalar data for a $2 \times 2 \times 2$ block of voxels. It solves the problem of efficiently retrieving voxel values and performing isosurface intersection without the overhead of recursive descent into child nodes. It differs from standard interior node traversal by replacing the isovalue range check and recursive child call with direct scalar retrieval and local neighbor finding. It is distinct from [Scalar Leaves](1525-sparse-voxel-traversal.md) (which have homogeneous values) and [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) (which handles general sparse structures) by specifically optimizing the $2 \times 2 \times 2$ boundary case.

### [Sparse Voxel Ray Marching with AABB Exit Planes](3252-sparse-voxel-ray-marching-with-aabb-exit-planes.md)
Sparse Voxel Ray Marching with AABB Exit Planes is a ray traversal algorithm for sparse voxel hierarchies that skips empty space by computing the exit plane of the current empty cell using precomputed Axis-Aligned Distance Fields (AADF). It differs from standard DDA [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md) by avoiding per-voxel checks in empty regions, and from pure octree traversal [Octree Traversal](1587-octree-traversal.md) by using fixed-size hierarchical chunks (chunks, bricks, voxels) with texture-based occupancy masks and distance fields rather than dynamic node pointers.

### [Voxel Ray Traversal (AABB-optimized)](3255-voxel-ray-traversal-aabb-optimized.md)
Voxel Ray Traversal (AABB-optimized) is an iterator-based algorithm that traverses a 3D voxel volume along a ray, returning hits for voxels contained within the volume. It solves the problem of efficiently enumerating occupied voxels in sparse volumes by combining a standard 3D DDA (Digital Differential Analyzer) traversal with an initial Axis-Aligned Bounding Box (AABB) rejection test. What sets it apart from its siblings, such as [Octree Traversal](1587-octree-traversal.md) or [Sparse Voxel Ray Marching with AABB Exit Planes](3252-sparse-voxel-ray-marching-with-aabb-exit-planes.md), is its reliance on a flat or chunk-based volume structure where the AABB test serves primarily as a "jump ahead" mechanism to skip empty space before the voxel-by-voxel iteration begins, rather than using hierarchical node structures for traversal.

### [Hierarchical Brick-Map DDA](4105-hierarchical-brick-map-dda.md)
Hierarchical Brick-Map DDA is a ray traversal technique that replaces a dense voxel buffer with a sparse hierarchical structure (brick-map) to skip empty regions. It sets itself apart from its parent [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) by using a fixed-depth, bit-packed hierarchy with exact cell-boundary snapping, and from its sibling [Octree Traversal](1587-octree-traversal.md) by relying on a fixed 2-level grid-of-bricks rather than a variable-depth tree, which simplifies traversal logic at the cost of potential wasted space in moderately sparse regions.

### Sparse Ray Marching for RTAO
A variant of sparse ray marching used for Ray Traced Ambient Occlusion. It marches from a surface point in a hemisphere direction, counting occlusions up to a radius limit [S4, unofficial]. It starts at the floor of the ray origin, initializes local position and distance to 0, and loops up to $3 \times \text{uniforms.grid\_ext}$ times [S4, unofficial]. It checks if `ipos` is outside the grid; if so, returns false [S4, unofficial]. It checks voxel occupancy using `voxel_masks`; if occupied, returns true (occlusion found) [S4, unofficial]. It loads the voxel AADF and calculates the exit plane [S4, unofficial]. It calculates `side_dist` and `t` as in the main sparse ray march, adds `t` to the accumulated distance, and if `distance > t_max`, returns false [S4, unofficial]. It advances the ray position, updating `ipos` and `local_pos` [S4, unofficial]. Unlike the main sparse ray march, this version does not use the hierarchical chunk/brick structure. It only checks `voxel_masks` and `voxel_aadf`. It also accumulates distance and stops at `t_max` [S4, unofficial].(Variant)

### Robust Interval Arithmetic Ray Traversal
A ray traversal method that uses interval arithmetic with correct rounding modes to guarantee that no octant is missed during traversal, solving precision errors that occur when subdivision levels approach floating-point limits [S5]. It performs ray traversal using interval arithmetic with correct rounding modes (using SSE2 instructions) to determine the order of sub-octants [S5]. It uses 30 bits precision (NL=30) and Octree level 9 (512^3 resolution) [S5]. When subdivision levels approach the limits of floating-point precision, standard traversals mismatch. This is handled by using 128-bit precision binary floating-point arithmetic for verification and interval arithmetic with correct rounding modes for the actual algorithm [S5]. Unlike Gargantini's method which only partially eliminates errors and requires restricted conditions, this method guarantees correctness for all cases with minimal performance impact [S5].(Variant)

### Hybrid Ray Tracing and Path Tracing
A hybrid rendering technique using the CGC data structure where cell rasterisation handles primary rays and path tracing handles secondary rays for global illumination [S6]. Using the CGC data structure for ray tracing or path tracing of isosurfaces, leveraging pre-computed cluster bounds for hierarchical traversal [S6]. It uses cell rasterisation for primary rays [S6]. It uses path tracing for secondary rays [S6]. It traverses and tests blocks, clusters, metacells, and individual voxals hierarchically using the pre-computed cluster bound stored in the last metacell of each cluster [S6]. For shadow and ambient occlusion rays, it fetches only metacell data [S6]. For additional bounces in path tracing, it fetches grid cell data to compute a normal at each intersection point [S6]. Primary rays can be either traced or rasterised (hybrid approach) [S6]. It uses 3 path tracing bounces [S6].(Variant)

## Operating characteristics
*   **Time Complexity:** $O(\log N)$ for sparse structures where $N$ is the number of voxels, as empty branches are skipped [S1][S2, unofficial].
*   **Memory Complexity:** Proportional to the number of non-empty nodes in the tree [S1].
*   **Performance:** The method exhibits similar performance to Gargantini's algorithm, with interval arithmetic variants being slightly slower but guaranteeing correctness [S5].
*   **Strengths:** Efficiently skips large volumes of empty space; handles arbitrary tree depths; thread-safe with copy constructor [S1].
*   **Limitations:** Requires uniform voxel sizes; precision issues at high subdivision levels without interval arithmetic [S1, S5].
*   **When to Use:** Sparse volumes with significant empty space; when interpolation kernels require dilation [S1].
*   **When Not to Use:** Dense volumes where flat grid traversal might be faster due to cache locality; non-uniform voxel grids [S1].

## Background
Sparse Voxel Traversal is used in OpenVDB v13.1.0 for ray marching through density or generic volumes [S1]. The core mechanism involves recursively traversing a tree where each node represents a cubic voxel or block, testing the ray against the node's AABB [Parent]. The method was developed to accelerate volume rendering by avoiding the sampling of empty space, a common requirement in large-scale VFX and scientific visualization [S1][S2, unofficial].

## Sources
[S1] OpenVDB: VolumeRayIntersector< GridT, NodeLevel, RayT > Class Template Reference, https://www.openvdb.org/documentation/doxygen/classopenvdb_1_1v13__1_1_1tools_1_1VolumeRayIntersector.html, fetched 2026-10-26
[S2] bfops/bit-svo, https://github.com/bfops/bit-svo, fetched 2026-10-26 (unofficial)
[S3] Intro to Rendering of Animated Volumetric Models in Breda, https://www.evolvebenchmark.com/blog-posts/intro-to-rendering-of-animated-volumetric-models-in-breda, fetched 2026-10-27 (unofficial)
[S4] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
[S5] Arbitrary 3D Resolution Discrete Ray Tracing of Implicit Surfaces, https://doi.org/10.1007/978-3-540-31965-8_38, fetched 2026-10-24
[S6] Clustered Grid Cell Data Structure for Isosurface Rendering, https://doi.org/10.24132/jwscg.2020.28.2, fetched 2026-10-26
