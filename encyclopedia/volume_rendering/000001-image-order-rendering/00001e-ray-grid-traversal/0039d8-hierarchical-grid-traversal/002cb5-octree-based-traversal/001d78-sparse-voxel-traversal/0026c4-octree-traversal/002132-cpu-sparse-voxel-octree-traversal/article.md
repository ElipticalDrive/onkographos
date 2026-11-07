# CPU Sparse Voxel Octree Traversal

Also called: Stack-based Sparse Voxel Octree Traversal

## Overview
CPU Sparse Voxel Octree Traversal is a ray traversal algorithm for 3D sparse voxel octrees stored in a linear block structure, implemented on the CPU [S1, unofficial]. It uses a stack to manage traversal state, handling valid internal nodes, leaf nodes, and empty space by casting rays to the next occupied voxel [S1, unofficial]. It sets itself apart from GPU-based approaches by performing all ray marching and shading calculations on the host processor, using a linear block structure for the octree and a stack-based traversal mechanism [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A Sparse Voxel Octree (SVO) stored in a linear block structure, camera parameters (screen width, height, field of view), and a ray origin/direction for each pixel [S1, unofficial].
*   **Output:** A rendered image (pixel colors) where leaf nodes contribute to the final color based on ray distance [S1, unofficial].

**Data Structures:**
*   **Voxel:** Represents an internal node. Contains bitmasks for `valid` (which children exist) and `leaf` (which children are leaf nodes), and an `address` pointing to children in the block [S1, unofficial].
*   **Leaf:** Represents a leaf node. Contains color data [S1, unofficial].
*   **VoxelStack:** Manages the traversal state [S1, unofficial].
    *   `entries`: Array of `StackEntry` objects.
    *   `top`: Current stack depth (index of the top entry).
    *   `box_size`: Size of the bounding box at the current stack level.
*   **StackEntry:** Represents a node on the traversal stack [S1, unofficial].
    *   `voxel`: Pointer to the current `Voxel` or `Leaf` in the block.
    *   `octant`: The index (0-7) of the child within the parent node that this entry represents.
    *   `corner`: The 3D coordinate of the corner of the current node's bounding box.
    *   `ray`: The ray being traced, potentially mirrored for intersection tests.

**Numbered Steps:**

1.  **Initialize Screen Coordinates:** For each pixel $(i, j)$ in the image:
    $$ \text{screen\_x} = \frac{i \times \text{fov}}{\text{WIDTH}} - 0.5 $$
    $$ \text{screen\_y} = \frac{j \times \text{fov}}{\text{HEIGHT}} - 0.5 $$
    Where `WIDTH` is 1280, `HEIGHT` is 720, and `fov` is 1.2 (default) [S1, unofficial].

2.  **Compute Ray Direction:**
    $$ \text{direction} = \text{Vector}(\text{screen\_x} \times \text{scale}.x, -1 \times \text{scale}.y, \text{screen\_y} \times \text{scale}.z) $$
    [S1, unofficial].

3.  **Push Root to Stack:**
    Initialize the stack with the root node:
    $$ \text{stack.push\_root}(\&\text{block}.\text{back}<\text{Voxel}>(), \text{Vector}(-1, -1, -1), \text{ray}) $$
    The initial corner is $(-1, -1, -1)$ and the ray is the camera ray [S1, unofficial].

4.  **Traversal Loop:** While the stack is not empty:
    a.  **Get Current Octant:** Retrieve the current octant index from the top of the stack:
        $$ \text{oct} = \text{stack}.\text{octant} $$
    b.  **Check Validity and Leaf Status:**
        $$ \text{valid} = (\text{stack}.\text{voxel}.\text{valid} \gg \text{oct}) \& 1 $$
        $$ \text{leaf} = (\text{stack}.\text{voxel}.\text{leaf} \gg \text{oct}) \& 1 $$
    c.  **If Leaf:**
        i.  Get leaf pointer:
            $$ \text{Leaf}^* \text{leaf} = \&\text{block}.\text{at}<\text{Leaf}>(\text{stack}.\text{voxel}.\text{address\_of}(\text{oct})) $$
        ii. Compute lightness:
            $$ \text{lightness} = \frac{1}{\text{ray.square\_distance}() + 1} $$
        iii. Set pixel color:
             $$ \text{leaf}.\text{set\_color}(\text{pixel}, \text{lightness}) $$
        iv. Break loop [S1, unofficial].
    d.  **If Valid (Internal Node):**
        i.  Push child to stack:
            $$ \text{stack.push}(\&\text{block}.\text{at}<\text{Voxel}>(\text{stack}.\text{voxel}.\text{address\_of}(\text{oct})), \text{ray}) $$
        ii. Update pixel green channel (visual feedback/accumulation):
            $$ \text{pixel}[1] += \frac{0\text{xff} - \text{pixel}[1]}{64} $$
        iii. Continue loop [S1, unofficial].
    e.  **If Invalid (Empty Space):**
        i.  Compute child corner:
            $$ \text{child\_corner} = \text{stack}.\text{corner} $$
            $$ \text{child\_corner}.\text{adjust\_corner}(\text{child\_size}, \text{oct}) $$
            Where $\text{child\_size} = \text{stack.box\_size} \times 0.5$ [S1, unofficial].
        ii. Compute mask:
            $$ \text{mask} = \text{ray.octant\_mask}() $$
        iii. Mirror ray and corner for ray-box intersection:
             $$ \text{mirror\_origin} = \text{ray.origin.mirror}(\text{mask}) $$
             $$ \text{mirror\_direction} = \text{ray.direction.mirror}(\text{mask}) $$
             $$ \text{mirror\_corner} = \text{child\_corner.mirror}(\text{mask}) $$
             $$ \text{mirror\_corner}.\text{adjust\_corner}(-\text{child\_size}, \text{mask}) $$
        iv. Compute intersection distances:
            $$ t_x = \frac{\text{mirror\_corner}.x - \text{mirror\_origin}.x}{\text{mirror\_direction}.x} $$
            $$ t_y = \frac{\text{mirror\_corner}.y - \text{mirror\_origin}.y}{\text{mirror\_direction}.y} $$
            $$ t_z = \frac{\text{mirror\_corner}.z - \text{mirror\_origin}.z}{\text{mirror\_direction}.z} $$
        v.  Find smallest positive $t$ and hit face:
            Initialize $t = \infty$, $\text{hit\_face} = 0$ [S1, unofficial].
            If $t_x > 0$, $t = t_x$, $\text{hit\_face} = 4$ [S1, unofficial].
            If $t_y > 0$ and $t_y < t$, $t = t_y$, $\text{hit\_face} = 2$ [S1, unofficial].
            If $t_z > 0$ and $t_z < t$, $t = t_z$, $\text{hit\_face} = 1$ [S1, unofficial].
        vi. March ray:
            $$ \text{ray.march}(t) $$
        vii. Transfer to sibling:
             $$ \text{stack}.\text{octant} \oplus= \text{hit\_face} $$
        viii. Continue loop [S1, unofficial].

5.  **Edge Case:** If the stack is empty after popping, the ray is outside the root octree. Set pixel blue channel to `0xc0` and break [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_sparse_voxel_octree_traversal(block, width, height, fov):
    # Constants
    WIDTH = 1280
    HEIGHT = 720
    FOV = fov
    SCALE_X, SCALE_Y, SCALE_Z = 1.0, 1.0, 1.0

    class Vector:
        def __init__(self, x, y, z):
            self.x, self.y, self.z = x, y, z
        def length_squared(self):
            return self.x**2 + self.y**2 + self.z**2
        def mirror(self, mask):
            return Vector(
                self.x if not (mask & 1) else -self.x,
                self.y if not (mask & 2) else -self.y,
                self.z if not (mask & 4) else -self.z
            )

    class Ray:
        def __init__(self, origin, direction):
            self.origin = origin
            self.direction = direction
        def square_distance(self):
            return self.origin.length_squared()
        def octant_mask(self):
            mask = 0
            if self.direction.x < 0: mask |= 1
            if self.direction.y < 0: mask |= 2
            if self.direction.z < 0: mask |= 4
            return mask
        def march(self, t):
            self.origin = Vector(
                self.origin.x + self.direction.x * t,
                self.origin.y + self.direction.y * t,
                self.origin.z + self.direction.z * t
            )

    class Voxel:
        def __init__(self, valid=0, leaf=0, address=0):
            self.valid = valid
            self.leaf = leaf
            self.address = address
        def address_of(self, oct):
            return self.address + oct

    class Leaf:
        def __init__(self):
            self.color = [0, 0, 0]
        def set_color(self, pixel, lightness):
            self.color = [int(c * lightness) for c in pixel]

    class StackEntry:
        def __init__(self, voxel, octant, corner, ray):
            self.voxel = voxel
            self.octant = octant
            self.corner = corner
            self.ray = ray

    class VoxelStack:
        def __init__(self):
            self.entries = []
            self.top = -1
            self.box_size = 2.0

        def push_root(self, voxel, corner, ray):
            self.entries.append(StackEntry(voxel, 0, corner, ray))
            self.top = 0

        def push(self, voxel, ray):
            self.top += 1
            self.entries.append(StackEntry(voxel, self.entries[self.top].octant, self.entries[self.top].corner, ray))

        def pop(self):
            if self.top >= 0:
                self.top -= 1
                return self.entries.pop()
            return None

        def is_empty(self):
            return self.top < 0

        @property
        def octant(self):
            return self.entries[self.top].octant

        @property
        def voxel(self):
            return self.entries[self.top].voxel

        @property
        def corner(self):
            return self.entries[self.top].corner

        @property
        def ray(self):
            return self.entries[self.top].ray

    def adjust_corner(corner, size, oct):
        corner.x += size if (oct & 1) else -size
        corner.y += size if (oct & 2) else -size
        corner.z += size if (oct & 4) else -size
        return corner

    stack = VoxelStack()

    for i in range(width):
        for j in range(height):
            # 1. Initialize Screen Coordinates
            screen_x = (i * FOV) / WIDTH - 0.5
            screen_y = (j * FOV) / HEIGHT - 0.5

            # 2. Compute Ray Direction
            direction = Vector(screen_x * SCALE_X, -1 * SCALE_Y, screen_y * SCALE_Z)
            ray = Ray(origin=Vector(0, 0, 0), direction=direction)

            # 3. Push Root to Stack
            stack.push_root(Voxel(), Vector(-1, -1, -1), ray)

            pixel = [0, 0, 0]

            # 4. Traversal Loop
            while not stack.is_empty():
                # Get current octant
                oct = stack.octant

                # Check validity and leaf status
                valid = (stack.voxel.valid >> oct) & 1
                leaf = (stack.voxel.leaf >> oct) & 1

                if leaf:
                    # If leaf
                    leaf_ptr = Leaf()
                    lightness = 1.0 / (ray.square_distance() + 1)
                    leaf_ptr.set_color(pixel, lightness)
                    break

                if valid:
                    # If valid internal node
                    child_voxel = Voxel()
                    stack.push(child_voxel, ray)
                    # Update pixel green channel
                    pixel[1] += (0xff - pixel[1]) / 64.0
                    continue

                # If invalid
                child_corner = Vector(stack.corner.x, stack.corner.y, stack.corner.z)
                child_size = stack.box_size * 0.5
                adjust_corner(child_corner, child_size, oct)

                mask = ray.octant_mask()

                mirror_origin = ray.origin.mirror(mask)
                mirror_direction = ray.direction.mirror(mask)
                mirror_corner = child_corner.mirror(mask)
                adjust_corner(mirror_corner, -child_size, mask)

                tx = (mirror_corner.x - mirror_origin.x) / mirror_direction.x
                ty = (mirror_corner.y - mirror_origin.y) / mirror_direction.y
                tz = (mirror_corner.z - mirror_origin.z) / mirror_direction.z

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

                ray.march(t)
                stack.entries[self.top].octant ^= hit_face

            if stack.is_empty():
                pixel[2] = 0xc0

            # Write pixel
            # write_pixel(i, j, pixel)
```

## Subtopics

### [Parametric Octree Traversal](2361-parametric-octree-traversal.md)
Parametric Octree Traversal is a top-down ray-octree traversal algorithm that uses parametric interval arithmetic to determine which child nodes of an octree node are intersected by a ray, handling non-uniform octree depths. It sets itself apart from bottom-up approaches (like DDA traversal) by performing a depth-first search that computes entry and exit parametric times ($t$) for the bounding box, allowing it to skip empty subtrees without requiring a fixed tree depth or explicit stack management for the traversal order itself.

### [Iterative Octree Traversal](2618-iterative-octree-traversal.md)
Iterative Octree Traversal is a CPU-based ray traversal algorithm for Sparse Voxel Octrees (SVO) that uses a fixed-size stack to manage traversal state, handling valid internal nodes, leaf nodes, and empty space by casting rays to the next occupied voxel. It employs a two-phase approach: a register-based DDA with bit-masked coordinate tracking for upper octree levels, and a slab-boundary-based stack traversal for deeper nodes.

### [Recursive Octree Traversal](2632-recursive-octree-traversal.md)
Recursive Octree Traversal is a depth-first algorithm for processing a 3D sparse voxel octree by visiting every node in the tree structure. It serves as the foundational traversal logic for both volume rendering (rendering active leaf nodes as cubes) and isosurface extraction (generating vertex buffers via contouring). It differs from its siblings by explicitly using the call stack to manage recursion depth, whereas [Iterative Octree Traversal](2618-iterative-octree-traversal.md) uses an explicit stack, and [Parametric Octree Traversal](2361-parametric-octree-traversal.md) uses parametric ray-marching logic. It builds on [CPU Sparse Voxel Octree Traversal](2455-cpu-sparse-voxel-octree-traversal.md) by replacing the stack-based state machine with direct recursive function calls.

### [Compact Voxel Octree Raycasting](2980-compact-voxel-octree-raycasting.md)
Compact Voxel Octree Raycasting is a CPU-based ray traversal algorithm for Sparse Voxel Octrees that optimizes memory layout and traversal logic by compacting active ray-voxel interactions and decoding node masks on-the-fly. It solves the problem of inefficient traversal in sparse structures by using an explicit stack to manage non-sequential memory layouts and a parallel "COMPACTIFY" step to filter out non-intersecting rays, ensuring that only valid ray-voxel pairs are processed in subsequent levels. Unlike recursive or parametric approaches, it relies on a linear block structure with explicit state management, making it suitable for high-performance CPU implementations where memory bandwidth and cache locality are critical. It builds on [CPU Sparse Voxel Octree Traversal](2455-cpu-sparse-voxel-octree-traversal.md) and differs from [Parametric Octree Traversal](2361-parametric-octree-traversal.md), [Iterative Octree Traversal](2618-iterative-octree-traversal.md), [Recursive Octree Traversal](2632-recursive-voxel-octree-traversal.md), and [Sparse Voxel Octree Path Tracing](2988-sparse-voxel-octree-path-tracing.md) by explicitly managing traversal state and compacting active ray lists between levels.

### [Sparse Voxel Octree Path Tracing](2988-sparse-voxel-octree-path-tracing.md)
Sparse Voxel Octree Path Tracing is a rendering technique that traverses a Sparse Voxel Octree (SVO) structure on the CPU to perform path tracing, writing results to a texture displayed via OpenGL. It differs from GPU-based SVO rendering by performing all ray marching and shading calculations on the host processor, using a linear block structure for the octree and a stack-based traversal mechanism. It builds on [CPU Sparse Voxel Octree Traversal](2455-cpu-sparse-voxel-octree-traversal.md) by integrating the traversal logic into a full path tracing loop that handles light bounces and material reflectivity.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
Not given in the sources.

**When to Use:**
Not given in the sources.

**When Not to Use:**
Not given in the sources.

## Background
**Origin:**
The method originates from the "zazen" 3D sparse voxel octree engine by Gabriel Muller [S1, unofficial]. It is described in a dissertation in Portuguese [S1, unofficial] and a presentation [S1, unofficial].

**History and Development:**
Not given in the sources.

**Where it is Used:**
Used in the "zazen" engine for rendering 3D models using sparse voxel octrees [S1, unofficial].

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
