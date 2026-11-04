# Ray Crossing Count (Winding Number)

Also called: Ray-Triangle Crossing Count, winding number via crossing count, Crossing Count Ray Query

## Overview
The Ray Crossing Count method determines whether a query point lies inside, outside, or on the boundary of a closed triangle mesh by casting rays and counting signed intersections with the mesh geometry. It solves the point-in-mesh containment problem, which is required for correct interior testing in volume rendering algorithms like Marching Cubes. Unlike simple binary hit tests, this method computes a generalized winding number: it returns exactly 1 for points inside a closed, outward-oriented mesh, 0 for points outside, and a continuous value between 0 and 1 for points near holes or in non-manifold configurations [S1, unofficial]. It distinguishes itself from sibling intersection tests by aggregating signed parity across multiple ray segments rather than returning a single boolean hit or distance [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A query point $P$, a ray direction $\mathbf{d}$ (typically axis-aligned for efficiency), and a triangle mesh represented by a bounding volume hierarchy (BVH).
- **Output**: An integer or float `crossingCount` representing the winding number.

**Data Structures**
- **BVH Node**: Contains an axis-aligned bounding box (AABB) and child pointers (or triangle indices for leaves).
- **Triangle**: Defined by three vertices and a precomputed normal vector $\mathbf{n}$.

**Algorithm Steps**
1. Initialize `crossingCount = 0` and `totalCount = 0`.
2. Traverse the BVH to identify all triangles whose bounding boxes intersect the query ray originating at $P$ with direction $\mathbf{d}$.
3. For each intersecting triangle $T$:
   a. Increment `totalCount`.
   b. Compute the dot product $D = \mathbf{n}_T \cdot \mathbf{d}$.
   c. If $D > 0$, the ray exits the volume (back-face hit); increment `crossingCount` by +1.
   d. If $D \le 0$, the ray enters the volume (front-face hit); decrement `crossingCount` by 1 (add -1).
4. Return `crossingCount`.

**Formula**
For a closed, outward-oriented surface mesh, the net crossing count $W$ for a ray from an outside point is 0, and for a ray from an inside point is +1 [S1, unofficial]. The contribution of each triangle $i$ is:
$$ W = \sum_{i} \text{sign}(\mathbf{n}_i \cdot \mathbf{d}) $$
where $\text{sign}(x) = +1$ if $x > 0$ and $-1$ otherwise [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_crossing_count(query_point, ray_direction, bvh):
    """
    Computes the signed ray crossing count for a query point against a mesh in a BVH.
    
    Args:
        query_point: 3D vector (x, y, z)
        ray_direction: 3D vector (x, y, z), assumed normalized
        bvh: Root node of a BVH containing triangles with normals
    
    Returns:
        int: The winding number (crossing count)
    """
    crossing_count = 0
    total_count = 0
    
    # Stack for iterative BVH traversal
    stack = [bvh]
    
    while stack:
        node = stack.pop()
        
        # Check if node AABB intersects the ray
        # Ray: P(t) = query_point + t * ray_direction, t > 0
        # AABB: [min, max]
        if not ray_aabb_intersect(query_point, ray_direction, node.aabb_min, node.aabb_max):
            continue
            
        if node.is_leaf:
            for triangle in node.triangles:
                # Perform precise ray-triangle intersection
                if ray_intersects_triangle(query_point, ray_direction, triangle):
                    total_count += 1
                    # Determine orientation: +1 for exit (back-face), -1 for entry (front-face)
                    dot_product = dot(triangle.normal, ray_direction)
                    if dot_product > 0.0:
                        crossing_count += 1
                    else:
                        crossing_count -= 1
        else:
            # Push children to stack
            stack.append(node.left_child)
            stack.append(node.right_child)
            
    return crossing_count

def ray_aabb_intersect(origin, direction, aabb_min, aabb_max):
    """
    Standard slab method for ray-AABB intersection.
    Returns True if the ray intersects the box.
    """
    t_min = -float('inf')
    t_max = float('inf')
    
    for i in range(3):
        if direction[i] == 0:
            if origin[i] < aabb_min[i] or origin[i] > aabb_max[i]:
                return False
        else:
            t1 = (aabb_min[i] - origin[i]) / direction[i]
            t2 = (aabb_max[i] - origin[i]) / direction[i]
            if t1 > t2:
                t1, t2 = t2, t1
            t_min = max(t_min, t1)
            t_max = min(t_max, t2)
            
    return t_max >= t_min

def ray_intersects_triangle(origin, direction, triangle):
    """
    Möller–Trumbore intersection algorithm.
    Returns True if the ray intersects the triangle.
    """
    v0, v1, v2 = triangle.vertices
    e1 = v1 - v0
    e2 = v2 - v0
    h = cross(direction, e2)
    a = dot(e1, h)
    
    if a <= 0.0000001: # Parallel or degenerate
        return False
        
    f = 1.0 / a
    s = origin - v0
    u = f * dot(s, h)
    
    if u < 0.0 or u > 1.0:
        return False
        
    q = cross(s, e1)
    v = f * dot(direction, q)
    
    if v < 0.0 or u + v > 1.0:
        return False
        
    t = f * dot(e2, q)
    
    # Ensure intersection is in front of the ray origin
    if t > 0.0000001:
        return True
        
    return False
```

**Parameters**
- **Ray Direction**: Often chosen as axis-aligned (e.g., $\pm x, \pm y, \pm z$) to simplify BVH traversal and intersection tests [S1, unofficial].
- **Threshold**: A small epsilon is used in intersection tests to handle numerical precision [G].

**Edge Cases and Pitfalls**
- **Degenerate Intersections**: Rays hitting edges or vertices may double-count triangles, making the theoretical crossing count unreliable for single rays [S1, unofficial].
- **Numerical Stability**: For query rays hitting edges, vertices, or numerically fancy configurations, the theory may not match practice [S1, unofficial].
- **Mesh Validity**: This method is meaningful for closed meshes only [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ in the worst case where $N$ is the number of triangles, but typically $O(\log N)$ or better with a BVH due to spatial culling.
- **Memory Complexity**: $O(N)$ to store the mesh and BVH.
- **Strengths**: Robust for closed manifolds; provides a continuous winding number for non-manifold inputs.
- **Limitations**: Sensitive to numerical precision at boundaries; requires a closed mesh for binary interpretation.

## Background
The method is referenced in the implementation of Vega et al., "A Fast and Memory-Saving Marching Cubes 33 Implementation with the Correct Interior Test" (JCGT 2019) [S1, unofficial]. It is used to ensure correct interior testing in volume extraction algorithms.

## Sources
[S1] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
