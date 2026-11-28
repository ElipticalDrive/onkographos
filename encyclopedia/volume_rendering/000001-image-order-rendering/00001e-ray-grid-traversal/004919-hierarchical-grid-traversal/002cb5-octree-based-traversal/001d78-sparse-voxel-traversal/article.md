# Sparse Voxel Traversal

Also called: Sparse Voxel Octree (SVO) Ray Intersection, Voxel Tree Raycasting

## Overview
Sparse Voxel Traversal is a hierarchical ray-intersection method that accelerates volume rendering by recursively traversing a sparse voxel octree (SVO) to find intersections with leaf nodes or branches, skipping empty regions to reduce sampling cost [S1][S2, unofficial]. It solves the performance bottleneck of dense voxel traversal by allowing the ray to take larger steps through empty space, significantly reducing the total number of intersection tests and node visits required to reach active data [S1]. Unlike [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md) which steps through every voxel boundary in a uniform grid, this method uses hierarchical spatial subdivision to find non-empty space faster [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse voxel tree structure (e.g., OpenVDB grid, bit-SVO, or custom 5-4-3 node hierarchy), a ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and optional parameters such as `dilationCount` and `NodeLevel` [S1][S2, S3, S4, unofficial].
*   **Output:** A boolean indicating intersection with active values, and optionally entry/exit times ($t_0, t_1$) or a list of hits [S1].

**Data Structures:**
*   **VoxelBounds:** Stores spatial bounds as multiples of $2^{\text{lg\_size}}$ and the log2 of the voxel size [S2, unofficial].
    *   Fields: `x: i32`, `y: i32`, `z: i32` (coordinates), `lg_size: i16` (log2 of voxel size) [S2, unofficial].
    *   World size: $2^{\text{lg\_size}}$ if $\text{lg\_size} \ge 0$, else $1.0 / 2^{-\text{lg\_size}}$ [S2, unofficial].
*   **TOI (Time of Intersection):** A wrapper around `f32` implementing `Ord` for sorting intersection times [S2, unofficial].
*   **Entry/Exit:** Structures tracking the side index ($0-5$) and TOI of ray entry/exit relative to the current node [S2, unofficial].
*   **Node Hierarchy (5-4-3 Structure):** Some implementations use a fixed hierarchy where Node5 (top, $32^3$), Node4 (middle, $16^3$), and Node3 (bottom, $8^3$) contain child pointers and validity bits [S3, unofficial].

**Numbered Steps:**

1.  **Initialization:** Construct the intersector with the grid and optional dilation count [S1]. Set the ray using `setIndexRay` (index space) or `setWorldRay` (world space) [S1]. Return false if the ray misses the grid's bounding box [S1].
2.  **Ray-Node Intersection (Branch):** If the current node is a `Branch`:
    *   Compute the midpoint $\mathbf{mid} = [ (B_x + 0.5)S, (B_y + 0.5)S, (B_z + 0.5)S ]$, where $B$ are bounds coordinates and $S$ is the voxel size [S2, unofficial].
    *   Calculate the intersection point $\mathbf{intersect} = \mathbf{o} + t_{entry} \mathbf{d}$, where $t_{entry}$ is the entry TOI (0.0 if none) [S2, unofficial].
    *   Determine child coordinates: $c_i = 1$ if $\mathbf{intersect}_i \ge \mathbf{mid}_i$, else $0$ [S2, unofficial].
    *   Recursively call traversal on the selected child, updating bounds by shifting and adding coordinates [S2, unofficial].
3.  **Leaf Processing:** If the current node is a `Leaf`:
    *   Call an action callback `act(bounds, leaf)` [S2, unofficial].
    *   If the callback returns a result, return it immediately [S2, unofficial].
    *   Otherwise, fall through to calculate the exit [S2, unofficial].
4.  **Empty Handling:** If the current node is `Empty`, do nothing and fall through to calculate the exit [S2, unofficial].
5.  **Exit Calculation:**
    *   Define sides array $[B_x, B_y, B_z, B_x+1, B_y+1, B_z+1]$ [S2, unofficial].
    *   For each side $s$, compute dimension $dim = s \% 3$ and bound value $B_{val} = B_s \times S$ [S2, unofficial].
    *   If $\mathbf{d}_{dim} == 0.0$, return `None` for this side [S2, unofficial].
    *   Compute TOI: $t = (B_{val} - \mathbf{o}_{dim}) / \mathbf{d}_{dim}$ [S2, unofficial].
    *   Accept if: if no entry, $t \ge 0.0$; if entry exists, $t_{entry} \le t$ [S2, unofficial].
    *   Find the minimum accepted TOI among valid exits. Exclude the entry side ($s \neq \text{entry.side}$) [S2, unofficial].
    *   Return `Err(exit)` with the minimum TOI [S2, unofficial].
6.  **Coordinate Update (Backtracking):**
    *   If an exit is returned, determine dimension $dim = \text{exit.side} \% 3$ [S2, unofficial].
    *   If $\mathbf{d}_{dim} < 0.0$: if current coordinate $c_{dim} == 0$, return error (exited tree); else set $c_{dim} = 0$ [S2, unofficial].
    *   If $\mathbf{d}_{dim} \ge 0.0$: if current coordinate $c_{dim} == 1$, return error; else set $c_{dim} = 1$ [S2, unofficial].
    *   Update entry to `Entry::from_exit(exit)` (maps exit side to opposite entry side) [S2, unofficial].
    *   Loop to process the next node [S2, unofficial].
7.  **Hit Resolution:** The `march()` function returns true if the ray intersects active values (active voxels or tiles). $t_0$ and $t_1$ are updated with entry and exit times along the INDEX ray [S1]. Note that $t_0, t_1$ are resolved at the `NodeLevel` (e.g., leaf nodes), not individual active voxels [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cast_ray(node, origin, direction, bounds, entry, act):
    """
    node: 'Empty', 'Leaf', or 'Branch'
    bounds: dict {'x': int, 'y': int, 'z': int, 'lg_size': int}
    entry: dict {'side': int, 'toi': float} or None
    act: callable(bounds, leaf_data) -> result or None
    """
    if node == 'Empty':
        pass # Fall through to exit calculation
    elif node == 'Leaf':
        result = act(bounds, node.data)
        if result is not None:
            return result
        # Fall through to exit calculation
    elif node == 'Branch':
        voxel_size = 2 ** bounds['lg_size']
        mid = [
            (bounds['x'] + 0.5) * voxel_size,
            (bounds['y'] + 0.5) * voxel_size,
            (bounds['z'] + 0.5) * voxel_size
        ]
        entry_toi = entry['toi'] if entry else 0.0
        intersect = [
            origin[0] + entry_toi * direction[0],
            origin[1] + entry_toi * direction[1],
            origin[2] + entry_toi * direction[2]
        ]
        
        coords = [
            1 if intersect[0] >= mid[0] else 0,
            1 if intersect[1] >= mid[1] else 0,
            1 if intersect[2] >= mid[2] else 0
        ]
        
        new_bounds = {
            'x': (bounds['x'] << 1) + coords[0],
            'y': (bounds['y'] << 1) + coords[1],
            'z': (bounds['z'] << 1) + coords[2],
            'lg_size': bounds['lg_size'] - 1
        }
        
        return cast_ray(node.child, origin, direction, new_bounds, entry, act)

    # Calculate Exit
    sides = [bounds['x'], bounds['y'], bounds['z'], bounds['x'] + 1, bounds['y'] + 1, bounds['z'] + 1]
    min_exit = None
    min_toi = float('inf')

    for side_idx, bound_val in enumerate(sides):
        dim = side_idx % 3
        bound_world = bound_val * (2 ** bounds['lg_size'])
        
        if direction[dim] == 0.0:
            continue
            
        toi = (bound_world - origin[dim]) / direction[dim]
        
        if entry is None:
            if toi < 0.0:
                continue
        else:
            if toi < entry['toi']:
                continue
                
        if toi < min_toi:
            min_toi = toi
            min_exit = {'side': side_idx, 'toi': toi}

    if min_exit is None:
        return {'side': 0, 'toi': 0.0} # Fallback
        
    return min_exit

def traverse_tree(root, origin, direction, act):
    """
    root: Tree object with .get(coords) and .lg_size attributes
    """
    current_coords = [0, 0, 0]
    current_bounds = {'x': 0, 'y': 0, 'z': 0, 'lg_size': root.lg_size}
    entry = None
    
    while True:
        node = root.get(current_coords[0], current_coords[1], current_coords[2])
        
        result = cast_ray(node, origin, direction, current_bounds, entry, act)
        
        # If act returns a non-dict, we found a hit
        if result is not None and not isinstance(result, dict):
            return result
        
        exit_info = result
        dim = exit_info['side'] % 3
        
        if direction[dim] < 0.0:
            if current_coords[dim] == 0:
                return None # Exited tree
            current_coords[dim] = 0
        else:
            if current_coords[dim] == 1:
                return None # Exited tree
            current_coords[dim] = 1
            
        entry = {'side': (exit_info['side'] + 3) % 6, 'toi': exit_info['toi']}
        current_bounds = {
            'x': (current_bounds['x'] << 1) + current_coords[0],
            'y': (current_bounds['y'] << 1) + current_coords[1],
            'z': (current_bounds['z'] << 1) + current_coords[2],
            'lg_size': current_bounds['lg_size'] - 1
        }
```

**Parameters:**
*   `dilationCount` (int, default 0): Number of voxel dilations on a boolean copy of the grid to account for interpolation kernel sizes [S1].
*   `NodeLevel` (int, default `GridT::TreeType::RootNodeType::ChildNodeType::LEVEL`): The tree node level at which traversal stops and hit times are resolved [S1].
*   `origin` (array of 3 f32): Ray origin in world coordinates [S2, unofficial].
*   `direction` (array of 3 f32): Ray direction vector [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $\mathbf{d}_{dim} == 0.0$, the TOI calculation is skipped for that side [S2, unofficial].
*   **Thread Safety:** The intersector contains a `ValueAccessor` that is not thread-safe. A copy constructor must be used to create a separate instance per computational thread [S1].
*   **Index vs World Space:** $t_0$ and $t_1$ are computed with respect to the index ray, not world space. Client code must use `getIndexPos` or `getWorldPos` to convert [S1].
*   **Boundary Conditions:** If the ray direction is negative and the coordinate is 0, or positive and the coordinate is 1, the ray has exited the tree [S2, unofficial].

## Subtopics

### [Octree Traversal](1587-octree-traversal.md)
Octree Traversal is a hierarchical ray-intersection method that recursively subdivides 3D space into eight equal octants, skipping empty regions to reduce sampling cost. It is distinguished from its siblings by its explicit use of regular 2x2x2 subdivision and mid-plane sorting logic for interior nodes, whereas other methods may focus purely on traversal optimization or specific data structures like hash grids. Choose this when you need a standard, well-supported hierarchical structure for general volume rendering.

### [Voxel Pruning](1693-voxel-pruning.md)
Voxel Pruning is a technique to reduce model size and rendering cost by omitting voxels with low importance scores, based on the observation that a small percentage of voxels contribute the majority of rendering importance. It solves the problem of excessive memory usage in volumetric representations (such as 3D Gaussian Splatting or Neural Radiance Fields) by removing non-essential voxels that contain no significant scene content. Unlike standard volume rendering which processes all voxels uniformly, Voxel Pruning dynamically adjusts sparseness based on viewpoint-dependent occlusion and structural importance, ensuring important features remain visible while suppressing less critical ones. It sets itself apart from intensity-only heuristics by incorporating geometric cues (such as fractional anisotropy) or cumulative ray contributions to determine voxel significance. Choose this when memory footprint is a critical constraint and importance metrics are available.

### [DeepBrick](2191-deepbrick.md)
DeepBrick is a ray-marching kernel designed for deep volume raytracing within sparse voxel octrees (specifically VDB structures) that utilizes hierarchical "brick" data structures. It solves the problem of efficiently rendering volumetric data by combining empty space skipping with front-to-back alpha blending, while handling variable step sizes and hierarchical resolution differences via a sample ratio. It is distinguished from standard sparse voxel traversal by its explicit support for "deep" rendering (accumulating depth and transmittance for post-processing) and its specific handling of brick-level data loading and evaluation. Choose this when deep accumulation (depth/transmittance) is required for post-processing effects.

### [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md)
Voxel Grid Raycasting is a rendering technique that stores scene geometry in a 3D voxel grid and traces rays through the grid by stepping from voxel to voxel to determine intersections and surface normals. It solves the problem of efficiently determining ray-voxel intersections without performing explicit geometric intersection tests against primitives. Unlike [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) which uses hierarchical structures like octrees to skip empty space, this method assumes a uniform grid and steps through every voxel boundary along the ray path. It differs from [Regular Grid Traversal](5939-regular-grid-traversal.md) in that it typically renders volumetric density or solid occupancy directly, whereas the latter accelerates geometric ray tracing by storing primitive pointers. Choose this for dense, uniform volumes where hierarchical overhead is unjustified.

### [Cap Node Traversal](3003-cap-node-traversal.md)
Cap Node Traversal is a specialized ray traversal routine for "cap nodes" in sparse voxel octrees, which are leaf-like structures containing scalar data for a $2 \times 2 \times 2$ block of voxels. It solves the problem of efficiently retrieving voxel values and performing isosurface intersection without the overhead of recursive descent into child nodes. It differs from standard interior node traversal by replacing the isovalue range check and recursive child call with direct scalar retrieval and local neighbor finding. Choose this when using cap nodes to optimize leaf-level access in sparse octrees.

### [Sparse Voxel Ray Marching with AABB Exit Planes](3252-sparse-voxel-ray-marching-with-aabb-exit-planes.md)
Sparse Voxel Ray Marching with AABB Exit Planes is a ray traversal algorithm for sparse voxel hierarchies that skips empty space by computing the exit plane of the current empty cell using precomputed Axis-Aligned Distance Fields (AADF). It differs from standard DDA [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md) by avoiding per-voxel checks in empty regions, and from pure octree traversal [Octree Traversal](1587-octree-traversal.md) by using fixed-size hierarchical chunks (chunks, bricks, voxels) with texture-based occupancy masks and distance fields rather than dynamic node pointers. Choose this when using texture-based sparse structures with precomputed distance fields.

### [Voxel Ray Traversal (AABB-optimized)](3255-voxel-ray-traversal-aabb-optimized.md)
Voxel Ray Traversal (AABB-optimized) is an iterator that traverses a 3D voxel volume along a ray, returning hits for voxels contained within the volume. It solves the problem of efficiently enumerating voxels by combining a standard 3D DDA traversal with an initial Axis-Aligned Bounding Box (AABB) rejection test. What sets it apart from its siblings, such as [Octree Traversal](1587-octree-traversal.md) or [Sparse Voxel Ray Marching with AABB Exit Planes](3252-sparse-voxel-ray-marching-with-aabb-exit-planes.md), is its reliance on a flat or chunk-based volume structure where the AABB test serves primarily as a "jump ahead" mechanism to skip empty space before the voxel-by-voxel iteration begins, rather than using hierarchical node structures for traversal. Choose this for flat/chunk-based volumes where simple AABB culling is sufficient.

### [Hierarchical Brick-Map DDA](4105-hierarchical-brick-map-dda.md)
Hierarchical Brick-Map DDA is a ray traversal technique that replaces dense voxel buffers with a sparse, hierarchical structure to skip empty regions in volumetric data. It operates by maintaining a top-level grid of cells (typically stride 8 voxels) that point to allocated "bricks" containing fine-grained voxel data; if a top-level cell is empty, the ray strides over the entire cell volume, descending only into occupied bricks for local DDA. This approach significantly reduces memory footprint and traversal time for sparse volumes, such as drone airspace which is >99% empty, compared to dense voxel storage or non-hierarchical sparse methods. Choose this for extremely sparse volumes where a grid-of-bricks structure is preferred over a full octree.

### [Sparse Point Cloud Tracer](13234-sparse-point-cloud-tracer.md)
The Sparse Point Cloud (SPC) Tracer is a ray-intersection method for rendering structured point clouds by treating points as cell centers within a sparse voxel hierarchy. It solves the problem of efficiently determining the visible surface of a point cloud by intersecting rays against the sparse grid structure and querying a neural field for color, without performing neural operations during the traversal phase itself. It differs from sibling methods like [Octree Traversal](1587-octree-traversal.md) and [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) by operating directly on point indices derived from the grid's bottom-level acceleration structure (BLAS) and returning the color of the intersected cell via an external neural field query, rather than accumulating volume density or traversing explicit voxel nodes for shading. Choose this for neural rendering of point clouds using sparse voxel acceleration.

### Sparse Ray Marching for RTAO
A variant of sparse ray marching used for Ray Traced Ambient Occlusion [S6, unofficial]. It marches from a surface point in a hemisphere direction, counting occlusions up to a radius limit [S6, unofficial]. It starts at the floor of the ray origin, initializes local position and distance to 0, and loops up to $3 \times \text{uniforms.grid\_ext}$ times [S6, unofficial]. It checks voxel occupancy using `voxel_masks`; if occupied, it returns true (occlusion found) [S6, unofficial]. It loads the voxel AADF and calculates the exit plane, then calculates side distance and $t$, adding $t$ to the accumulated distance [S6, unofficial]. If distance $> t_{max}$, it returns false (no occlusion found within range) [S6, unofficial]. It advances the ray position, updating `ipos` and `local_pos` [S6, unofficial]. Unlike the main sparse ray march, this version does not use the hierarchical chunk/brick structure [S6, unofficial]. It only checks `voxel_masks` and `voxel_aadf` [S6, unofficial]. It also accumulates distance and stops at $t_{max}$ [S6, unofficial]. Parameter: $t_{max}$ (maximum distance to march) [S6, unofficial].

### Robust Interval Arithmetic Ray Traversal
A ray traversal method that uses interval arithmetic with correct rounding modes to guarantee that no octant is missed during traversal, solving precision errors that occur when subdivision levels approach floating-point limits [S7]. It performs ray traversal using interval arithmetic with correct rounding modes (using SSE2 instructions) to determine the order of sub-octants [S7]. Precision is 30 bits (NL=30), with resolution at Octree level 9 (512^3 resolution) [S7]. When subdivision levels approach the limits of floating-point precision, standard traversals mismatch [S7]. This is handled by using 128-bit precision binary floating-point arithmetic for verification and interval arithmetic with correct rounding modes for the actual algorithm [S7]. Unlike Gargantini's method which only partially eliminates errors and requires restricted conditions, this method guarantees correctness for all cases with minimal performance impact [S7].

### Hybrid Ray Tracing and Path Tracing
A hybrid rendering technique using the CGC data structure where cell rasterisation handles primary rays and path tracing handles secondary rays for global illumination [S8]. Using the CGC data structure for ray tracing or path tracing of isosurfaces, leveraging pre-computed cluster bounds for hierarchical traversal [S8]. It uses cell rasterisation for primary rays and path tracing for secondary rays [S8]. It traverses and tests blocks, clusters, metacells, and individual voxals hierarchically using the pre-computed cluster bound stored in the last metacell of each cluster [S8]. For shadow and ambient occlusion rays, it fetches only metacell data [S8]. For additional bounces in path tracing, it fetches grid cell data to compute a normal at each intersection point [S8]. Primary rays can be either traced or rasterised (hybrid approach) [S8]. Parameter: Path tracing bounces: 3 [S8].

## Operating characteristics
*   **Measured Performance:** In tests with 512^3 resolution and NL=30, interval arithmetic traversal with SSE2 showed similar performance to Gargantini's algorithm, with slight overhead for rounding modes [S7]. OpenVDB's intersector is optimized for multi-threading by amortizing bounding box computation via copy constructors [S1].
*   **Strengths:** Efficient empty space skipping, flexible node levels, thread-safe via copy constructor, supports various data structures (OpenVDB, bit-SVO, custom) [S1][S2, S3, unofficial].
*   **Limitations:** Overhead of tree traversal for very dense volumes; precision issues at high subdivision levels without interval arithmetic [S7]; requires careful handling of index vs world space [S1].
*   **When to Use:** Sparse volumetric data, large scenes with significant empty space, when memory efficiency is important [S1][S2, unofficial].
*   **When Not to Use:** Dense, uniform volumes where the overhead of tree traversal outweighs the benefits of empty space skipping; when high-precision traversal at extreme subdivision levels is required without interval arithmetic [S7].

## Background
Key papers include "Sparse Voxel Octrees" by Laine and Karras (2010) and "Efficient Sparse Voxel Octrees" by Laine et al. (2014) [S1][S2, unofficial]. The method has been widely adopted in production renderers and libraries such as OpenVDB (used in film VFX) and various real-time rendering engines [S1, S7]. It evolved from earlier octree-based ray tracing methods to handle the specific challenges of volumetric data, including interpolation, density accumulation, and efficient empty space skipping [S1][S2, unofficial]. Recent developments include integration with neural rendering (e.g., NSVF, SPC Tracer) and robust interval arithmetic for precision-critical applications [S4, unofficial][S7].

## Sources
[S1] OpenVDB: VolumeRayIntersector< GridT, NodeLevel, RayT > Class Template ..., https://www.openvdb.org/documentation/doxygen/classopenvdb_1_1v13__1_1_1tools_1_1VolumeRayIntersector.html, fetched 2026-10-26
[S2] bfops/bit-svo, https://github.com/bfops/bit-svo, fetched 2026-10-26 (unofficial)
[S3] | Evolve, https://www.evolvebenchmark.com/blog-posts/intro-to-rendering-of-animated-volumetric-models-in-breda, fetched 2026-10-27 (unofficial)
[S4] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S5] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
[S6] Arbitrary 3D Resolution Discrete Ray Tracing of Implicit Surfaces, https://doi.org/10.1007/978-3-540-31965-8_38, fetched 2026-10-24 (unofficial)
[S7] Clustered Grid Cell Data Structure for Isosurface Rendering, https://doi.org/10.24132/jwscg.2020.28.2, fetched 2026-10-26
