# Ray-Octree Traversal with Stack-Based Backtracking

Also called: TreeTraverser

## Overview
A ray casting algorithm that traverses an octree structure to locate leaf voxels intersected by a ray. It uses a stack to store parent context for backtracking when a ray exits a voxel or skips empty space, allowing it to resume traversal at a higher level. It builds on [Stack-based Octree Traversal with Direction Flipping](5007-stack-based-octree-traversal-with-direction-flipping.md) by explicitly managing the traversal state via a stack to handle arbitrary ray directions and empty space skipping.

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Octree root node.
- Maximum scale $s_{max}$ (defines the finest resolution).
- Chunk size constant `CHUNK_SIZE` and number of chunks `CHUNK_NUM` (used to derive initial size).
- Epsilon $\epsilon$ for direction clamping.

**Outputs:**
- The leaf voxel index/pointer where the ray terminates (or a list of leaves, depending on implementation; the source code breaks on the first leaf).

**Data Structures:**
- `parentStack`: Array of tuples `(Voxel*, float, glm::ivec3)` of size $s_{max} + 1$. Stores:
    - `Voxel*`: Pointer to the parent node.
    - `float`: The $t_{max}$ value at the time of push.
    - `glm::ivec3`: The `lastPos` (position) at the time of push.
- `Voxel` structure (inferred):
    - `validMask`: Bitmask indicating which children exist.
    - `leafMask`: Bitmask indicating which children are leaves.
    - `childPointers`: Array of pointers to child nodes.

**Parameters:**
- $\epsilon$: Minimum absolute value for ray direction components. Value: $2^{-s_{max}}$ (e.g., $0.015625$ for $s_{max}=6$) [S1, unofficial].

**Steps:**

1.  **Clamp Ray Direction:**
    For each component $c \in \{x, y, z\}$ of $\mathbf{d}$:
    If $|\mathbf{d}.c| < \epsilon$, set $\mathbf{d}.c = \text{copysign}(\epsilon, \mathbf{d}.c)$.
    This prevents division by zero in ray-box intersection calculations [S1, unofficial].

2.  **Initialize Traversal State:**
    - Calculate initial $t_{min}$ and $t_{max}$ for the entire octree bounding box using `findMinMax`.
    - If $t_{min} \neq 9999$ (indicating intersection):
        - Move origin to the nearest face: $\mathbf{o} \leftarrow \mathbf{o} + t_{min} \cdot \mathbf{d}$.
        - Update $t_{max} \leftarrow t_{max} - t_{min}$.
        - Set $t_{min} \leftarrow 0$.
    - Initialize `scale` $= s_{max} - 1$.
    - Initialize `size` $= \text{CHUNK\_NUM} \times \text{Chunk::CHUNK\_SIZE} / 2$.
    - Get initial `idx` and `pos` using `getIdx(origin, &pos, size)`.
    - Set `parent` to the octree root.
    - Initialize `lastPos` and `h` (current max t) appropriately (source implies `h` tracks the exit t of the current voxel).

3.  **Traversal Loop:**
    While `scale` $< s_{max}$:
    a.  **Calculate Intersection:**
        - Call `findMinMax(origin, dir, pos, size, &c_min, &c_max)` to get the entry and exit $t$ values for the current voxel.
        - Update $t_{min} \leftarrow \max(t_{min}, c_{min})$.
        - Calculate $t_{c\_max} \leftarrow \min(c_{max}, t_{max})$.

    b.  **Check Validity and Intersection:**
        - Check if the current voxel is valid and intersects the ray:
          If `(parent->validMask & (1 << idx))` AND $t_{min} \le t_{c\_max}$:
            i.  **Leaf Check:**
                If `(parent->leafMask & (1 << idx))`, break (leaf found).
            ii. **Push to Stack:**
                If $c_{max} < h$ (ray exits this voxel before the previous max exit, implying we are going deeper into a valid child that might be skipped later or we need to backtrack):
                Push `(parent, t_max, lastPos)` onto `parentStack[scale]`.
            iii. **Update State for Child:**
                - `lastPos` $\leftarrow$ `pos`.
                - $h \leftarrow c_{max}$.
                - $t_{max} \leftarrow c_{max}$.
                - `size` $\leftarrow$ `size` / 2.
                - `scale` $\leftarrow$ `scale` - 1.
            iv. **Find Child Pointer:**
                - `pointerIndex` $\leftarrow -1$.
                - For $i$ from $0$ to `idx`:
                  If `(parent->validMask & (1 << i))`, increment `pointerIndex`.
                - `parent` $\leftarrow$ `parent->childPointers[pointerIndex]`.
            v.  **Update Position for Next Level:**
                - Calculate point: $\mathbf{p} = \mathbf{o} + (c_{min} + 0.00001f) \cdot \mathbf{d}$.
                - Get new `idx` and `pos` using `getIdx(p, &pos, size)`.
            vi. Continue loop.

        - Else (voxel invalid or no intersection):
            i.  **Backtrack:**
                - Convert `idx` using `convertIndex` array.
                - Adjust `pos` based on index bits:
                  If bit 2 of `idx` is set, `pos.x` $\leftarrow$ `pos.x` - `size`.
                  If bit 4 of `idx` is set, `pos.y` $\leftarrow$ `pos.y` - `size`.
                - Pop from `parentStack` (logic implied by stack usage; source snippet cuts off explicit pop but indicates push logic and stack usage for backtracking).
                - Update `parent`, `scale`, `size`, etc., from the popped stack entry.
                - Continue loop.

**Pitfalls:**
- Division by zero in ray-box intersection if ray direction component is zero. Handled by clamping small components to epsilon with correct sign [S1, unofficial].
- Precision issues near voxel boundaries. Mitigated by adding a small offset ($0.00001f$) to $c_{min}$ when calculating the next point [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Proportional to the number of nodes visited. In sparse volumes, significantly faster than dense grid traversal as empty space is skipped.
- **Memory Complexity:** $O(s_{max})$ for the stack, where $s_{max}$ is the maximum depth of the octree.
- **Strengths:** Efficient for sparse data; handles arbitrary ray directions via direction flipping (from parent) and stack-based backtracking.
- **Limitations:** Stack operations add overhead; performance degrades if the octree is dense or the ray passes through many small voxels.
- **When to use:** Sparse voxel octrees where empty space skipping is beneficial.
- **When not to use:** Dense volumes or when hardware ray tracing (RTX) is available and preferred.

## Background
Originated in academic projects for volumetric rendering, specifically in the context of GPU-based ray casting. The implementation details are found in a senior project repository [S1, unofficial].

## Sources
[S1] Dean904/Senior-Project-Volumetric-Rendering, https://github.com/Dean904/Senior-Project-Volumetric-Rendering, fetched 2026-10-26 (unofficial)
