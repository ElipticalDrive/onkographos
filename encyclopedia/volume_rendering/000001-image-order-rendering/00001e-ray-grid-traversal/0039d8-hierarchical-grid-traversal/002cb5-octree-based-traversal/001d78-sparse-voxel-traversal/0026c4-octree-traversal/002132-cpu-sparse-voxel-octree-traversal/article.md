# CPU Sparse Voxel Octree Traversal
Also called: Stack-based Sparse Voxel Octree Traversal

## Overview
CPU Sparse Voxel Octree Traversal is a ray traversal algorithm for 3D sparse voxel octrees stored in a linear block structure, implemented on the CPU [S1, unofficial]. It uses a stack to manage traversal state, handling valid internal nodes, leaf nodes, and empty space by casting rays to the next occupied voxel [S1, unofficial]. It sets itself apart from GPU-based methods by relying on explicit stack management and discrete slab boundaries for child selection, differing from parametric approaches that use continuous intervals [S2, S4, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A Sparse Voxel Octree (SVO) stored in a linear block structure; screen coordinates $(i, j)$; camera parameters (width, height, field of view).
*   **Output:** Pixel color values (RGB) for the given screen coordinate.

**Data Structures**
*   **Voxel:** Represents an internal node. Contains bitmasks for `valid` (occupancy) and `leaf` (child type), and an `address_of` field pointing to children [S1, unofficial].
*   **Leaf:** Represents a leaf node containing color/opacity data [S1, unofficial].
*   **VoxelStack:** Manages the traversal state [S1, unofficial].
    *   `entries`: Array of `StackEntry` objects.
    *   `top`: Current stack depth (index of the top entry).
    *   `box_size`: Size of the bounding box at the current stack level.
*   **StackEntry:** Represents a single node in the traversal stack [S1, unofficial].
    *   `voxel`: Pointer to the current `Voxel` node.
    *   `octant`: The index (0-7) of the child octant being processed within the current voxel.
    *   `corner`: The 3D coordinate of the corner of the current bounding box.
    *   `ray`: The ray being traced (origin and direction).

**Algorithm Steps**
1.  **Initialize Screen Coordinates:**
    Calculate normalized screen coordinates for pixel $(i, j)$:
    $$
    \text{screen\_x} = \frac{i \cdot \text{fov}}{\text{WIDTH}} - 0.5
    $$
    $$
    \text{screen\_y} = \frac{j \cdot \text{fov}}{\text{HEIGHT}} - 0.5
    $$
    Where `WIDTH` is 1280, `HEIGHT` is 720, and `fov` is 1.2 (default) [S1, unofficial].

2.  **Compute Ray Direction:**
    $$
    \text{direction} = \text{Vector}(\text{screen\_x} \cdot \text{scale}.x, -1 \cdot \text{scale}.y, \text{screen\_y} \cdot \text{scale}.z)
    $$
    [S1, unofficial].

3.  **Push Root to Stack:**
    Initialize the stack with the root voxel.
    $$
    \text{stack.push\_root}(\&\text{block}.\text{back}<\text{Voxel}>(), \text{Vector}(-1, -1, -1), \text{ray})
    $$
    [S1, unofficial].

4.  **Traversal Loop:**
    While the stack is not empty:
    a.  **Get Current Octant:**
        $$
        \text{oct} = \text{stack}.\text{octant}
        $$
    b.  **Check Validity and Leaf Status:**
        $$
        \text{valid} = (\text{stack}.\text{voxel}.\text{valid} \gg \text{oct}) \& 1
        $$
        $$
        \text{leaf} = (\text{stack}.\text{voxel}.\text{leaf} \gg \text{oct}) \& 1
        $$
    c.  **If Leaf:**
        i.  Get leaf pointer:
            $$
            \text{Leaf}^* \text{leaf} = \&\text{block}.\text{at}<\text{Leaf}>(\text{stack}.\text{voxel}.\text{address\_of}(\text{oct}))
            $$
        ii. Compute lightness:
            $$
            \text{lightness} = \frac{1}{\text{ray}.\text{square\_distance}() + 1}
            $$
        iii. Set pixel color:
            $$
            \text{leaf}.\text{set\_color}(\text{pixel}, \text{lightness})
            $$
        iv. Break loop [S1, unofficial].
    d.  **If Valid (Internal Node):**
        i.  Push child to stack:
            $$
            \text{stack.push}(\&\text{block}.\text{at}<\text{Voxel}>(\text{stack}.\text{voxel}.\text{address\_of}(\text{oct})), \text{ray})
            $$
        ii. Update pixel green channel:
            $$
            \text{pixel}[1] += \frac{0\text{xff} - \text{pixel}[1]}{64}
            $$
        iii. Continue loop [S1, unofficial].
    e.  **If Invalid (Empty Space):**
        i.  Compute child corner:
            $$
            \text{child\_corner} = \text{stack}.\text{corner}
            $$
            $$
            \text{child\_corner}.\text{adjust\_corner}(\text{child\_size}, \text{oct})
            $$
            Where $\text{child\_size} = \text{stack}.\text{box\_size} \cdot 0.5$.
        ii. Compute mask:
            $$
            \text{mask} = \text{ray}.\text{octant\_mask}()
            $$
        iii. Mirror ray and corner for ray-box intersection:
            $$
            \text{mirror\_origin} = \text{ray}.\text{origin}.\text{mirror}(\text{mask})
            $$
            $$
            \text{mirror\_direction} = \text{ray}.\text{direction}.\text{mirror}(\text{mask})
            $$
            $$
            \text{mirror\_corner} = \text{child\_corner}.\text{mirror}(\text{mask})
            $$
            $$
            \text{mirror\_corner}.\text{adjust\_corner}(-\text{child\_size}, \text{mask})
            $$
        iv. Compute intersection distances:
            $$
            t_x = \frac{\text{mirror\_corner}.x - \text{mirror\_origin}.x}{\text{mirror\_direction}.x}
            $$
            $$
            t_y = \frac{\text{mirror\_corner}.y - \text{mirror\_origin}.y}{\text{mirror\_direction}.y}
            $$
            $$
            t_z = \frac{\text{mirror\_corner}.z - \text{mirror\_origin}.z}{\text{mirror\_direction}.z}
            $$
        v.  Find smallest positive $t$ and hit face:
            Initialize $t = \infty$, $\text{hit\_face} = 0$.
            If $t_x > 0$, $t = t_x$, $\text{hit\_face} = 4$.
            If $t_y > 0$ and $t_y < t$, $t = t_y$, $\text{hit\_face} = 2$.
            If $t_z > 0$ and $t_z < t$, $t = t_z$, $\text{hit\_face} = 1$.
        vi. March ray:
            $$
            \text{ray}.\text{march}(t)
            $$
        vii. Transfer to sibling:
            $$
            \text{stack}.\text{octant} \text{ ^= } \text{hit\_face}
            $$
        viii. Continue loop [S1, unofficial].

5.  **Edge Case: Empty Stack**
    If the stack is empty after popping, the ray is outside the root octree. Set pixel blue channel to `0xc0` and break [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_svo_traversal(i, j, block, root_voxel, width=1280, height=720, fov=1.2):
    # 1. Initialize Screen Coordinates
    screen_x = (i * fov) / width - 0.5
    screen_y = (j * fov) / height - 0.5
    
    # 2. Compute Ray Direction
    # Note: scale.x, scale.y, scale.z are assumed to be derived from camera setup
    direction = Vector(screen_x * scale.x, -1 * scale.y, screen_y * scale.z)
    ray = Ray(origin=Vector(0, 0, 0), direction=direction)
    
    # 3. Push Root to Stack
    stack = VoxelStack()
    stack.push_root(block.back<Voxel>(), Vector(-1, -1, -1), ray)
    
    pixel = Color(0, 0, 0)
    
    while not stack.is_empty():
        # 4. Get Current Octant
        oct = stack.octant
        voxel = stack.voxel
        
        # Check validity and leaf status
        valid = (voxel.valid >> oct) & 1
        leaf_flag = (voxel.leaf >> oct) & 1
        
        if leaf_flag:
            # 4c. If Leaf
            leaf_ptr = block.at<Leaf>(voxel.address_of(oct))
            lightness = 1.0 / (ray.square_distance() + 1)
            leaf_ptr.set_color(pixel, lightness)
            break
            
        elif valid:
            # 4d. If Valid (Internal Node)
            child_voxel = block.at<Voxel>(voxel.address_of(oct))
            stack.push(child_voxel, ray)
            
            # Update pixel green channel
            pixel.g += (0xff - pixel.g) / 64.0
            
        else:
            # 4e. If Invalid (Empty Space)
            child_size = stack.box_size * 0.5
            child_corner = stack.corner.adjust_corner(child_size, oct)
            mask = ray.octant_mask()
            
            # Mirror ray and corner
            mirror_origin = ray.origin.mirror(mask)
            mirror_direction = ray.direction.mirror(mask)
            mirror_corner = child_corner.mirror(mask).adjust_corner(-child_size, mask)
            
            # Compute intersection distances
            tx = (mirror_corner.x - mirror_origin.x) / mirror_direction.x
            ty = (mirror_corner.y - mirror_origin.y) / mirror_direction.y
            tz = (mirror_corner.z - mirror_origin.z) / mirror_direction.z
            
            # Find smallest positive t and hit face
            t = float('inf')
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
                
            # March ray
            ray.march(t)
            
            # Transfer to sibling
            stack.octant ^= hit_face
            
    # 5. Edge Case: Empty Stack
    if stack.is_empty():
        pixel.b = 0xc0
        
    return pixel
```

## Subtopics

### [Parametric Octree Traversal](2361-parametric-octree-traversal.md)
Parametric Octree Traversal is a top-down recursive algorithm for finding all voxels in a sparse voxel octree intersected by a ray. It solves the problem of efficiently traversing non-uniform octrees without maintaining neighbor pointers or sorting potentially visited sub-voxels. It sets itself apart from bottom-up methods like Samet by avoiding neighbor finding, and from HERO by computing the sequence of visited sub-voxels sequentially using simple parametric comparisons rather than sorting.

### [Iterative Octree Traversal](2618-iterative-octree-traversal.md)
Iterative Octree Traversal is a method for raycasting against a 3D Sparse Voxel Octree (SVO) that replaces recursive function calls with an explicit stack to comply with hardware constraints (such as HLSL recursion limits) or to optimize CPU cache behavior. It solves the problem of unbounded call stack depth by simulating the traversal state using fixed-size arrays or dynamic stacks, employing a greedy approach that prioritizes the closest node while handling false positives via backtracking. It sets itself apart from [Recursive Octree Traversal](2632-recursive-octree-traversal.md) by explicitly managing the call stack and from [Parametric Octree Traversal](2361-parametric-octree-traversal.md) by often using discrete slab boundaries or DDA steps rather than parametric intervals for child selection.

### [Recursive Octree Traversal](2632-recursive-octree-traversal.md)
Recursive Octree Traversal is a depth-first algorithm that processes a 3D sparse voxel octree by visiting nodes to either render them as geometry or process them for mesh generation. It differs from [Iterative Octree Traversal](2618-iterative-octree-traversal.md) by using the call stack instead of an explicit stack, and from [Parametric Octree Traversal](2361-parametric-octree-traversal.md) by relying on hierarchical node masks rather than parametric ray-marching. It builds on [CPU Sparse Voxel Octree Traversal](2455-cpu-sparse-voxel-octree-traversal.md) by replacing the stack-based state machine with direct recursive function calls.

### [Compact Voxel Octree Raycasting](2980-compact-voxel-octree-raycasting.md)
Compact Voxel Octree Raycasting is a CPU-based ray traversal algorithm for Sparse Voxel Octrees that optimizes memory layout and traversal logic by compacting active ray-voxel interactions and decoding node masks on-the-fly. It solves the problem of inefficient traversal in sparse structures by using an explicit stack to manage non-sequential memory layouts and a parallel "COMPACTIFY" step to filter out non-intersecting rays, ensuring that only valid ray-voxel pairs are processed in subsequent levels. Unlike recursive or parametric approaches, it relies on a linear block structure with explicit state management, making it suitable for high-performance CPU implementations where memory bandwidth and cache locality are critical. It builds on [CPU Sparse Voxel Octree Traversal](2455-cpu-sparse-voxel-octree-traversal.md) and differs from [Parametric Octree Traversal](2361-parametric-octree-traversal.md), [Iterative Octree Traversal](2618-iterative-octree-traversal.md), [Recursive Octree Traversal](2632-recursive-octree-traversal.md), and [Sparse Voxel Octree Path Tracing](2988-sparse-voxel-octree-path-tracing.md) by explicitly managing traversal state and compacting active ray lists between levels.

### [Sparse Voxel Octree Path Tracing](2988-sparse-voxel-octree-path-tracing.md)
Sparse Voxel Octree Path Tracing is a rendering technique that traverses a Sparse Voxel Octree (SVO) structure on the CPU to perform path tracing, writing results to a texture displayed via OpenGL. It differs from GPU-based SVO rendering by performing all ray marching and shading calculations on the host processor, using a linear block structure for the octree and a stack-based traversal mechanism. It builds on [CPU Sparse Voxel Octree Traversal](2455-cpu-sparse-voxel-octree-traversal.md) by integrating the traversal logic into a full path tracing loop that handles light bounces and material reflectivity.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Explicit stack management allows for fine-grained control over traversal state, suitable for CPU implementations where cache locality and memory bandwidth are critical [S2, unofficial].
*   **Limitations:** Relies on discrete slab boundaries which may be less efficient than parametric approaches in certain scenarios [S2, S4, unofficial].
*   **When to Use:** High-performance CPU implementations where memory bandwidth and cache locality are critical [S2, unofficial].
*   **When Not to Use:** GPU implementations with recursion limits or scenarios requiring parametric interval comparisons [S2, S4, unofficial].

## Background
CPU Sparse Voxel Octree Traversal was introduced by Gabriel Muller in the "zazen" project, a 3D sparse voxel octree engine [S1, unofficial]. The method is detailed in a dissertation in Portuguese and associated presentation materials [S1, unofficial]. It is used for rendering 3D models using sparse voxel octrees on the CPU [S1, unofficial].

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
