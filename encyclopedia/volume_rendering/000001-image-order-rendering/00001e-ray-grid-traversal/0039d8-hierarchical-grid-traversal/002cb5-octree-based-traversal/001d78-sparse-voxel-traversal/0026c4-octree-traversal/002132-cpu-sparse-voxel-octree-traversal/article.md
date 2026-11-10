# CPU Sparse Voxel Octree Traversal

Also called: SVO Traversal, Sparse Voxel Octree Ray Casting, Sparse Voxel Octree Tracing, Stack-based Sparse Voxel Octree Traversal

## Overview
CPU Sparse Voxel Octree Traversal is a real-time ray traversal algorithm for sparse voxel octrees that uses a stack-based approach to navigate from the root to leaf voxels, calculating intersections with node boundaries to determine the next octant [S1, S2, unofficial]. It solves the problem of efficiently rendering large sparse voxel datasets on the CPU by avoiding the overhead of recursive function calls and leveraging explicit state management for cache-friendly memory access [S1, S2, unofficial]. It sets itself apart from [Recursive Octree Traversal](2632-recursive-octree-traversal.md) by using an explicit stack instead of the call stack, and from [Parametric Octree Traversal](2361-parametric-octree-traversal.md) by often relying on discrete slab boundaries or DDA steps rather than parametric intervals for child selection [S2, S4, unofficial].

## How it works
The method traverses a Sparse Voxel Octree (SVO) stored in a linear block structure. It initializes a ray for each pixel, pushes the root node onto a stack, and iteratively processes nodes until a leaf voxel is hit or the stack is empty.

### Inputs and Outputs
*   **Input**: A Sparse Voxel Octree (SVO) stored in a linear array of nodes/leaves; a camera position and direction (or per-pixel ray); screen dimensions (`WIDTH`, `HEIGHT`); field of view factor (`fov`).
*   **Output**: A rendered pixel color (e.g., RGBA buffer).

### Data Structures
The SVO nodes are stored in a linear array. Two primary structures are used:

1.  **Voxel Node** (Internal Node):
    *   `leaf_mask`: `uint32_t`, 8-bit bitmask indicating which children are leaf voxels.
    *   `valid_masks`: `uint32_t`, 8-bit bitmask indicating which children exist (are not empty).
    *   `child_pointers`: `uint32_t[8]`, indices into the root array for the 8 children.
    *   *Note*: Some implementations use a union with `Leaf` and a boolean flag `is_leaf` [S2, unofficial].

2.  **Leaf Node**:
    *   Contains color/intensity data.
    *   *Note*: In some implementations, the leaf is accessed via an address offset from the block base [S2, unofficial].

3.  **Stack Entry** (for Iterative/Stack-based traversal):
    *   `voxel`: Pointer or index to the current node in the linear block.
    *   `octant`: The octant index (0-7) of the child being visited within the parent node.
    *   `ray`: The current ray state (origin, direction).
    *   `box_size`: The size of the bounding box at the current stack level.
    *   `corner`: The corner coordinate of the current bounding box.
    *   *Note*: The stack itself is an array of `StackEntry` with a `top` index [S2, unofficial].

4.  **Ray State**:
    *   `origin`: `Vector3` (x, y, z).
    *   `direction`: `Vector3` (x, y, z).
    *   `square_distance()`: Computes the squared distance from the ray origin to the current intersection point [S2, unofficial].

### Numbered Steps

1.  **Initialize Ray**: For each pixel $(i, j)$, compute screen coordinates and ray direction.
    *   $screen\_x = (i \times fov) / WIDTH - 0.5$
    *   $screen\_y = (j \times fov) / HEIGHT - 0.5$
    *   $direction = (screen\_x \times scale.x, -1 \times scale.y, screen\_y \times scale.z)$ [S2, unofficial].
    *   Set ray origin to camera position.

2.  **Push Root**: Push the root node of the octree onto the stack.
    *   Set initial `box_size` to the root box size.
    *   Set initial `corner` to the root box corner (e.g., $(-1, -1, -1)$) [S2, unofficial].
    *   Set initial `octant` to 0 (or the first valid child) [S2, unofficial].

3.  **Traversal Loop**: While the stack is not empty:
    a.  **Pop/Peek**: Get the current stack entry. Let `node` be the current voxel, `oct` be the current octant.
    b.  **Check Validity**: Check if the child at `oct` exists.
        *   If using bitmasks: `valid = (node.valid_masks >> oct) & 1` [S1, unofficial].
        *   If using union/flags: `valid = node.is_valid` and check child existence [S2, unofficial].
    c.  **If Invalid**:
        *   Compute the child corner: `child_corner = corner`. Adjust `child_corner` by `child_size` and `oct` [S2, unofficial].
        *   Compute `child_size = box_size * 0.5`.
        *   Compute ray-box intersection for the child's bounding box.
            *   Mirror ray and corner if necessary based on `octant_mask` [S2, unofficial].
            *   Compute intersection distances $t_x, t_y, t_z$.
            *   Find smallest positive $t$ and hit face [S2, unofficial].
        *   March ray by $t$: `ray.march(t)` [S2, unofficial].
        *   Transfer to sibling: Update `octant` to the next valid child (e.g., `octant ^= hit_face`) [S2, unofficial].
        *   Continue loop.
    d.  **If Valid**:
        *   Check if Leaf: `leaf = (node.leaf_mask >> oct) & 1` [S1, unofficial] or `node.is_leaf` [S2, unofficial].
        *   **If Leaf**:
            *   Retrieve leaf data.
            *   Compute lighting/color. Example: `lightness = 1 / (ray.square_distance() + 1)` [S2, unofficial].
            *   Set pixel color. Example: `pixel[1] += (0xff - pixel[1]) / 64` [S2, unofficial].
            *   Break loop (ray hit).
        *   **If Internal Node**:
            *   Push child node to stack.
            *   Update `box_size *= 0.5`.
            *   Update `corner` by adding `half_size` if `octant` bit is set [S1, unofficial].
            *   Update `ray` origin to current intersection point.
            *   Set `octant` to 0 (or first valid child of new node).
            *   Continue loop.

4.  **Exit Condition**:
    *   If stack is empty and no voxel hit, return default color (e.g., `0xFF000000` or background color) [S1, S2, unofficial].

### Formulas
*   **Ray-Box Intersection**:
    *   $t_x = (mirror\_corner.x - mirror\_origin.x) / mirror\_direction.x$
    *   $t_y = (mirror\_corner.y - mirror\_origin.y) / mirror\_direction.y$
    *   $t_z = (mirror\_corner.z - mirror\_origin.z) / mirror\_direction.z$
    *   $t = \min(t_x, t_y, t_z)$ where $t > 0$ [S2, unofficial].

*   **Octant Masking**:
    *   $mask = ray.octant\_mask()$ [S2, unofficial].
    *   Used to mirror ray components for slab intersection tests.

### Parameters
*   `MAX_DETAIL_LEVEL`: Maximum recursion depth, typically 20 [S1, unofficial].
*   `WIDTH`: Screen width, e.g., 1280 [S2, unofficial].
*   `HEIGHT`: Screen height, e.g., 720 [S2, unofficial].
*   `fov`: Field of view factor, 1.0 corresponds to 90 degrees, default 1.2 [S2, unofficial].

### Edge Cases and Pitfalls
*   **Empty Stack**: If the stack becomes empty before a leaf is hit, the ray has exited the octree. Return a default background color [S1, S2, unofficial].
*   **Precision**: Floating-point precision issues may cause rays to miss voxels or intersect incorrectly. Use epsilon values for comparisons if necessary [G].
*   **Degenerate Input**: Empty octrees or invalid nodes should be handled gracefully [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_svo_traversal(octree, camera, width, height, fov):
    buffer = [0] * (width * height)
    
    for j in range(height):
        for i in range(width):
            # Initialize Ray
            screen_x = (i * fov) / width - 0.5
            screen_y = (j * fov) / height - 0.5
            direction = Vector(screen_x * scale.x, -scale.y, screen_y * scale.z)
            ray_origin = camera.position
            ray = Ray(ray_origin, direction)
            
            # Push Root
            stack = Stack()
            stack.push(
                voxel=octree.root,
                octant=0,
                ray=ray,
                box_size=octree.root_box_size,
                corner=octree.root_box_corner
            )
            
            hit = False
            while not stack.empty():
                entry = stack.peek()
                node = entry.voxel
                oct = entry.octant
                ray = entry.ray
                box_size = entry.box_size
                corner = entry.corner
                
                # Check Validity
                valid = (node.valid_masks >> oct) & 1
                if not valid:
                    # Compute child corner
                    child_size = box_size * 0.5
                    child_corner = corner.adjust_corner(child_size, oct)
                    
                    # Mirror for intersection
                    mask = ray.octant_mask()
                    mirror_origin = ray.origin.mirror(mask)
                    mirror_direction = ray.direction.mirror(mask)
                    mirror_corner = child_corner.mirror(mask)
                    mirror_corner.adjust_corner(-child_size, mask)
                    
                    # Intersection
                    tx = (mirror_corner.x - mirror_origin.x) / mirror_direction.x
                    ty = (mirror_corner.y - mirror_origin.y) / mirror_direction.y
                    tz = (mirror_corner.z - mirror_origin.z) / mirror_direction.z
                    
                    t = infinity
                    hit_face = 0
                    if tx > 0:
                        t = tx
                        hit_face = 4
                    if ty > 0 and ty < t:
                        t = ty
                        hit_face = 2
                    if tz > 0 and tz < t:
                        t = tz
                        hit_face = 1
                    
                    if t == infinity:
                        stack.pop()
                        continue
                    
                    ray.march(t)
                    entry.octant ^= hit_face
                    stack.update(entry)
                    continue
                
                # Check Leaf
                is_leaf = (node.leaf_mask >> oct) & 1
                if is_leaf:
                    leaf = octree.get_leaf(node.child_pointers[oct])
                    lightness = 1.0 / (ray.square_distance() + 1.0)
                    pixel_index = j * width + i
                    buffer[pixel_index] = compute_color(leaf, lightness)
                    hit = True
                    break
                else:
                    # Push Child
                    child_node = octree.get_node(node.child_pointers[oct])
                    half_size = box_size * 0.5
                    new_corner = corner
                    if oct & 1: new_corner.x += half_size
                    if oct & 2: new_corner.y += half_size
                    if oct & 4: new_corner.z += half_size
                    
                    stack.push(
                        voxel=child_node,
                        octant=0,
                        ray=ray,
                        box_size=half_size,
                        corner=new_corner
                    )
            
            if not hit:
                pixel_index = j * width + i
                buffer[pixel_index] = 0xFF000000 # Default color
    
    return buffer
```

## Subtopics

### [Parametric Octree Traversal](2361-parametric-octree-traversal.md)
Parametric Octree Traversal is a top-down recursive algorithm for finding all voxels in a sparse voxel octree intersected by a ray. It solves the problem of efficiently traversing non-uniform octrees without maintaining neighbor pointers or sorting potentially visited sub-voxels. It sets itself apart from bottom-up methods like Samet by avoiding neighbor finding, and from HERO by computing the sequence of visited sub-voxels sequentially using simple parametric comparisons rather than sorting.

### [Iterative Octree Traversal](2618-iterative-octree-traversal.md)
Iterative Octree Traversal is a method for raycasting against a 3D Sparse Voxel Octree (SVO) that replaces recursive function calls with an explicit stack to comply with hardware constraints (such as HLSL recursion limits) or to optimize CPU cache behavior. It solves the problem of unbounded call stack depth by simulating the traversal state using fixed-size arrays or dynamic stacks, employing a greedy approach that prioritizes the closest node while handling false positives via backtracking. It sets itself apart from [Recursive Octree Traversal](2632-recursive-octree-traversal.md) by explicitly managing the call stack and from [Parametric Octree Traversal](2361-parametric-octree-traversal.md) by often using discrete slab boundaries or DDA steps rather than parametric intervals for child selection.

### [Recursive Octree Traversal](2632-recursive-octree-traversal.md)
Recursive Octree Traversal is a depth-first algorithm that processes a 3D sparse voxel octree by visiting nodes to either render them as geometry or process them for mesh generation. It differs from [Iterative Octree Traversal](2618-iterative-octree-traversal.md) by using the call stack instead of an explicit stack, and from [Parametric Octree Traversal](2361-parametric-octree-traversal.md) by relying on hierarchical node masks rather than parametric ray-marching. It builds on [CPU Sparse Voxel Octree Traversal](2455-cpu-sparse-voxel-octree-traversal.md) by replacing the stack-based state machine with direct recursive function calls.

### [Compact Voxel Octree Raycasting](2980-compact-voxel-octree-raycasting.md)
Compact Voxel Octree Raycasting is a CPU-based ray traversal algorithm for Sparse Voxel Octrees that optimizes memory layout and traversal logic by compacting active ray-voxel interactions and decoding node masks on-the-fly. It solves the problem of inefficient traversal in sparse structures by using an explicit stack to manage non-sequential memory layouts and a parallel "COMPACTIFY" step to filter out non-intersecting rays, ensuring that only valid ray-voxel pairs are processed in subsequent levels. Unlike recursive or parametric approaches, it relies on a linear block structure with explicit state management, making it suitable for high-performance CPU implementations where memory bandwidth and cache locality are critical. It builds on [CPU Sparse Voxel Octree Traversal](2455-cpu-sparse-voxel-octree-traversal.md) and differs from [Parametric Octree Traversal](2361-parametric-octree-traversal.md), [Iterative Octree Traversal](2618-iterative-octree-traversal.md), [Recursive Octree Traversal](2632-recursive-voxel-octree-traversal.md), and [Sparse Voxel Octree Path Tracing](2988-sparse-voxel-octree-path-tracing.md) by explicitly managing traversal state and compacting active ray lists between levels.

### [Sparse Voxel Octree Path Tracing](2988-sparse-voxel-octree-path-tracing.md)
Sparse Voxel Octree Path Tracing is a rendering technique that traverses a Sparse Voxel Octree (SVO) structure on the CPU to perform path tracing, writing results to a texture displayed via OpenGL. It differs from GPU-based SVO rendering by performing all ray marching and shading calculations on the host processor, using a linear block structure for the octree and a stack-based traversal mechanism. It builds on [CPU Sparse Voxel Octree Traversal](2455-cpu-sparse-voxel-octree-traversal.md) by integrating the traversal logic into a full path tracing loop that handles light bounces and material reflectivity.

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of nodes visited along the ray path. In sparse scenes, this is significantly less than dense voxel traversal.
*   **Memory Complexity**: $O(D)$ where $D$ is the depth of the octree, for the stack.
*   **Performance**: Real-time performance is achievable on CPU for moderate resolution scenes [S1, unofficial]. Performance scales with the sparsity of the voxel data.
*   **Strengths**: Efficient for sparse data; no need for neighbor pointers; explicit stack allows for easy optimization and debugging.
*   **Limitations**: Stack management overhead; potential for cache misses if node layout is not optimized.
*   **When to Use**: CPU-based rendering of large sparse voxel datasets; when GPU recursion limits are a concern.
*   **When Not to Use**: Dense voxel data; when GPU acceleration is available and preferred.

## Background
CPU Sparse Voxel Octree Traversal was developed as part of real-time voxel rendering experiments, notably in the `voxl` project by Robert Olsson [S1, unofficial] and the `zazen` engine by Gabriel Muller [S2, unofficial]. These projects aimed to bring real-time sparse voxel rendering to the CPU, inspired by commercial solutions like Euclideon [S1, unofficial]. The method has been used in academic projects and open-source engines to demonstrate efficient CPU-based voxel rendering.

## Sources
[S1] rools/voxl, https://github.com/rools/voxl, fetched 2026-10-26 (unofficial)
[S2] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
