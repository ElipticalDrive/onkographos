# Ray-Octree Traversal with Stack-Based Backtracking

Also called: TreeTraverser

## Overview
A ray casting algorithm that traverses an octree structure to locate leaf voxels intersected by a ray [S1, unofficial]. It builds on [Stack-based Octree Traversal with Direction Flipping](5007-stack-based-octree-traversal-with-direction-flipping.md) by using a stack to store parent context for backtracking when the ray exits a voxel or skips empty space [S1, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1, unofficial]. Octree root node [S1, unofficial]. Maximum scale $s_{max}$ [S1, unofficial]. Chunk size constant `CHUNK_SIZE` and number of chunks `CHUNK_NUM` [S1, unofficial].

**Outputs:** The leaf voxel index/pointer where the ray terminates [S1, unofficial].

**Data Structures:** `parentStack`: Array of tuples `(Voxel*, float, glm::ivec3)` of size $s_{max} + 1$ storing parent pointer, $t_{max}$ value, and `lastPos` for backtracking [S1, unofficial]. `Voxel` structure contains `validMask` (bitmask for existing children), `leafMask` (bitmask for leaf children), and `childPointers` (array of child node pointers) [S1, unofficial].

**Parameters:** $\epsilon$: Minimum absolute value for ray direction components to avoid division by zero. Value: $2^{-s_{max}}$ (e.g., $0.015625$ for $s_{max}=6$) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def traverse_ray_octree(origin, dir, root, s_max, CHUNK_NUM, CHUNK_SIZE, epsilon):
    # Clamp direction to avoid division by zero
    for axis in ('x', 'y', 'z'):
        if abs(dir[axis]) < epsilon:
            dir[axis] = math.copysign(epsilon, dir[axis])

    # Initial bounding box intersection
    t_min, t_max = findMinMax(origin, dir, root.bounds)
    if t_min != 9999:
        origin = origin + t_min * dir
        t_max -= t_min
        t_min = 0.0

    scale = s_max - 1
    size = (CHUNK_NUM * CHUNK_SIZE) / 2
    idx, pos = getIdx(origin, size)
    parent = root
    lastPos = pos
    h = t_max
    parentStack = [None] * (s_max + 1)

    while scale < s_max:
        c_min, c_max = findMinMax(origin, dir, pos, size)
        t_min = max(t_min, c_min)
        tc_max = min(c_max, t_max)

        if (parent.validMask & (1 << idx)) and t_min <= tc_max:
            if parent.leafMask & (1 << idx):
                return idx
            if c_max < h:
                parentStack[scale] = (parent, t_max, lastPos)
            lastPos = pos
            h = c_max
            t_max = c_max
            size /= 2
            scale -= 1

            pointerIndex = -1
            for i in range(idx + 1):
                if parent.validMask & (1 << i):
                    pointerIndex += 1
            parent = parent.childPointers[pointerIndex]

            point = origin + (c_min + 0.00001) * dir
            idx, pos = getIdx(point, size)
            continue
        else:
            # Backtrack
            idx = convertIndex[idx]
            if idx & 2: pos.x -= size
            if idx & 4: pos.y -= size
            # Source snippet cuts off explicit pop logic; standard stack pop restores parent, t_max, lastPos, scale, and size [S1, unofficial].
            if parentStack[scale] is not None:
                parent, t_max, lastPos = parentStack[scale]
                scale += 1
                size *= 2
                parentStack[scale] = None
            else:
                break
    return None
```

**Pitfalls:** Division by zero in ray-box intersection if ray direction component is zero. Handled by clamping small components to epsilon with correct sign [S1, unofficial]. Precision issues near voxel boundaries. Mitigated by adding a small offset ($0.00001$) to $c_{min}$ when calculating the next point [S1, unofficial].

## Operating characteristics
Time complexity is proportional to the number of visited nodes, significantly faster than dense grid traversal for sparse volumes [G]. Memory complexity is $O(s_{max})$ for the stack [S1, unofficial]. Strengths include efficient empty space skipping and handling arbitrary ray directions via the parent method [G]. Limitations include stack operation overhead and performance degradation on dense volumes [G]. Use for sparse voxel octrees; avoid for dense volumes or when hardware ray tracing is preferred [G].

## Background
Originated in academic projects for GPU-based volumetric ray casting [S1, unofficial].

## Sources
[S1] Dean904/Senior-Project-Volumetric-Rendering, https://github.com/Dean904/Senior-Project-Volumetric-Rendering, fetched 2026-10-26 (unofficial)
