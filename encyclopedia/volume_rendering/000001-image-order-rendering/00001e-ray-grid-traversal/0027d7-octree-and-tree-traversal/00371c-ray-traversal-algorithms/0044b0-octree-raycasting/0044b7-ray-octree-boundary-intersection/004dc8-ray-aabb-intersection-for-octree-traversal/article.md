# Ray-AABB Intersection for Octree Traversal

Also called: Ray-Parameter Function, Revelles, Urena and Lastra Octree Traversal

## Overview
This algorithm determines whether a ray intersects an axis-aligned bounding box (AABB) representing an octree node and prepares the necessary parameters for traversing its subtrees. It distinguishes itself by normalizing the ray direction to be strictly positive through coordinate flipping, which simplifies the subsequent intersection logic and traversal order determination. This method serves as the core intersection test within the broader [Ray-Octree Boundary Intersection](11318-ray-octree-boundary-intersection.md) framework.

## How it works
**Inputs:**
- Ray origin $o$ and direction $d$ (3D vectors).
- Octree node bounds: minimum corner $\_min$, maximum corner $\_max$, and node size $\_size$.
- Note: $\_size$ is typically derived from bounds. Sources indicate ambiguity: $\_size = \_max - \_min$ or $\_size = (\_max - \_min) / 2$. The algorithm assumes $\_size$ corresponds to the extent used in the origin flip formula [S1, unofficial].

**Outputs:**
- Boolean indicating intersection.
- If intersecting, calls `processSubtree` with entry/exit distances and a bitmask.

**Data Structures:**
- `a`: Unsigned integer bitmask (3 bits) indicating which axes were flipped.
- `t0`: Vector of entry distances per axis.
- `t1`: Vector of exit distances per axis.
- `t`: Scalar tracking the maximum entry distance (closest entry point).

**Steps:**
1. Initialize bitmask $a = 0$.
2. Copy ray origin $o$ and direction $d$.
3. For each axis $i \in \{0, 1, 2\}$:
   - If $d[i] < 0$:
     - Flip origin: $o[i] = \_size[i] - o[i]$. [G: This assumes the local coordinate system origin is at $\_min$ and $\_size$ represents the full extent. If $\_size$ is half-extent, this formula is incorrect. Standard AABB traversal usually flips relative to the box bounds directly, but this specific implementation uses $\_size$] [S1, unofficial].
     - Negate direction: $d[i] = -d[i]$.
     - Set bit $i$ in $a$: $a \ |= \ (1 \ll (2-i))$. Specifically: bit 2 (value 4) for axis 0, bit 1 (value 2) for axis 1, bit 0 (value 1) for axis 2 [S1, unofficial].
4. Compute entry distances: $t0 = (\_min - o) / d$.
5. Compute exit distances: $t1 = (\_max - o) / d$.
6. Initialize $t = \infty$ (maximum double value).
7. Check intersection: If $\max(t0) < \min(t1)$, the ray intersects the box. Call `processSubtree(t0, t1, r, t, sd, a)`. Otherwise, return false.

**Pitfalls:**
- Incorrect traversal logic or misinterpretation of $\_size$ can cause rendering artifacts, such as only the upper-right part of the image being rendered while the rest is rejected [S1, unofficial].
- The formula $o[i] = \_size[i] - o[i]$ is specific to the implementation cited. Standard implementations often use $o[i] = \_max[i] - o[i]$ or similar depending on coordinate space. Implementers must verify the definition of $\_size$ in their specific octree structure [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_aabb_intersect(oct_node, ray):
    # Inputs: oct_node with attributes _min, _max, _size
    # ray with attributes origin (o), direction (d)
    
    a = 0
    o = copy(ray.origin)
    d = copy(ray.direction)
    
    # Normalize ray direction to be positive
    if d[0] < 0:
        o[0] = oct_node._size[0] - o[0]
        d[0] = -d[0]
        a |= 4  # Bit 2
    
    if d[1] < 0:
        o[1] = oct_node._size[1] - o[1]
        d[1] = -d[1]
        a |= 2  # Bit 1
        
    if d[2] < 0:
        o[2] = oct_node._size[2] - o[2]
        d[2] = -d[2]
        a |= 1  # Bit 0
        
    # Compute intersection distances
    t0 = (oct_node._min - o) / d
    t1 = (oct_node._max - o) / d
    
    t = float('inf')
    
    # Check for intersection
    if max(t0) < min(t1):
        # Call subtree processing with prepared parameters
        return process_subtree(t0, t1, ray, t, surface_data, a)
    else:
        return False
```

## Operating characteristics
- **Time Complexity:** $O(1)$ per node visit. The intersection test involves a constant number of arithmetic operations and comparisons.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Simplifies traversal logic by ensuring positive ray directions, allowing consistent ordering of child node visits based on the bitmask.
- **Limitations:** Relies on correct interpretation of node size and bounds. Incorrect implementation can lead to significant rendering errors (e.g., missing image regions) [S1, unofficial].
- **When to use:** When traversing an octree where node bounds are axis-aligned and the traversal order needs to be determined by the ray's entry/exit points relative to the node's children.

## Background
This method is based on the "Efficient Parametric Algorithm for Octree Traversal" by Revelles, Urena, and Lastra. The specific implementation details and pitfalls discussed in the sources stem from a community discussion on Stack Overflow regarding an implementation of this algorithm [S1, unofficial]. The algorithm is a standard technique in volume rendering and ray tracing for efficient spatial subdivision traversal.

## Sources
[S1] Only upper right part of image rendered when using octree, https://stackoverflow.com/questions/20808334/only-upper-right-part-of-image-rendered-when-using-octree, fetched 2026-10-24 (unofficial)
