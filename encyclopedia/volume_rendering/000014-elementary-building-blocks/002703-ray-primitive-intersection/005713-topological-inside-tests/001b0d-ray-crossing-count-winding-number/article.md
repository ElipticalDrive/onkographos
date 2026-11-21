# Ray Crossing Count (Winding Number)

Also called: winding number via crossing count, Ray-Triangle Crossing Count, Crossing Count Ray Query

## Overview

The Ray Crossing Count method determines whether a 3D point lies inside or outside a closed triangle mesh by casting a ray and counting surface intersections. It solves the topological "inside/outside" ambiguity that plagues simpler depth-based tests, which can fail when a ray intersects the same triangle multiple times or hits edges/vertices, causing sign-flipping artifacts [S3, unofficial]. Unlike the Region Nesting Merge sibling, which relies on hierarchical containment logic, this method uses a direct parity or signed accumulation of ray-triangle hits [S1, S2, unofficial].

## How it works

### Inputs and Outputs
- **Input**: A query point $P = (P_x, P_y, P_z)$ and a triangle mesh $\mathcal{M}$ consisting of $N$ triangles. Each triangle $T_i$ is defined by vertices $V_{i,0}, V_{i,1}, V_{i,2}$ and an outward-facing normal $\mathbf{n}_i$.
- **Output**: A boolean indicating if $P$ is inside the mesh, or an integer/float winding number.

### Data Structures
1.  **Triangle List**: An array of $N$ triangles. Each triangle stores vertex positions and precomputed normals.
2.  **Bounding Volume Hierarchy (BVH)**: A tree structure where each node stores an Axis-Aligned Bounding Box (AABB) enclosing its child triangles. Leaves contain indices to the triangle list. This allows efficient culling of non-intersecting triangles [S2, unofficial].
3.  **2D Grid (Optional Optimization)**: For axis-aligned ray casting, a 2D grid of size $W \times H$ can store lists of triangles intersecting each grid cell, along with their intersection $z$-values. This reduces the search space from 3D volume to 2D surface projection [S1, unofficial].

### Parameters
- **Ray Direction**: Typically aligned with a principal axis (e.g., $+z$) for the 2D grid optimization, or arbitrary directions for robustness [S1, S2, unofficial].
- **Intersection Algorithm**: The Müller-Trombose algorithm is cited as a robust choice for ray-triangle intersection [S3, unofficial].
- **Grid Resolution**: Example given is $128 \times 128$ for the 2D projection optimization [S1, unofficial].

### Steps

1.  **Ray Initialization**: Define a ray origin at the query point $P$ and a direction $\mathbf{d}$. For the 2D grid method, $\mathbf{d} = (0, 0, 1)$ (positive z-axis) [S1, unofficial]. For the general winding number, $\mathbf{d}$ can be any direction, often sampled from multiple axes [S2, unofficial].

2.  **Intersection Search**:
    -   **BVH Approach**: Traverse the BVH to find all triangles whose bounding boxes intersect the ray [S2, unofficial].
    -   **2D Grid Approach**: Project $P$ to 2D coordinates $(P_x, P_y)$. Retrieve the precomputed list of triangles intersecting this 2D grid cell [S1, unofficial].

3.  **Triangle Intersection Test**: For each candidate triangle $T_i$:
    -   Compute the intersection between the ray and the triangle plane.
    -   Verify the intersection point lies within the triangle boundaries (barycentric coordinates).
    -   Check if the intersection occurs in the forward direction of the ray ($t > 0$) [S1, S3, unofficial].

4.  **Counting Logic**:
    -   **Parity Method (Simple Inside/Outside)**:
        -   Initialize `count = 0`.
        -   For each valid intersection, increment `count`.
        -   If `count` is odd, $P$ is inside; if even, $P$ is outside [S1, S3, unofficial].
    -   **Signed Winding Number Method (Robust)**:
        -   Initialize `crossingCount = 0` and `totalCount = 0` [S2, unofficial].
        -   For each valid intersection:
            -   Increment `totalCount`.
            -   Calculate the dot product between the triangle normal $\mathbf{n}_i$ and the ray direction $\mathbf{d}$: $dot = \mathbf{n}_i \cdot \mathbf{d}$.
            -   If $dot > 0$, the ray exits the volume (or enters depending on orientation convention); add $+1$ to `crossingCount`.
            -   If $dot \le 0$, the ray enters the volume; add $-1$ to `crossingCount`.
            -   *Note*: The source code example adds $+1$ if `dot > 0` and implicitly $-1$ otherwise (or does not add if equal, but standard winding numbers usually handle grazing cases carefully) [S2, unofficial].
        -   For a closed, outward-oriented mesh, a point outside should yield a net count of $0$, and a point inside should yield $+1$ (exiting once more than entering) [S2, unofficial].

5.  **Multi-Ray Voting (Optional Robustness)**: To handle degenerate cases where a single ray hits an edge or vertex, cast rays in six directions (positive/negative x, y, z) and take a majority vote of the inside/outside results [S2, unofficial].

6.  **Generalized Winding Number**: For meshes with holes or "soups" (non-closed), the signed crossing count provides a continuous value between $0$ and $1$, representing the probability or density of being inside [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_crossing_count(query_point, triangles, bvh, use_signed=True):
    """
    Determines if query_point is inside the mesh defined by triangles.
    
    Args:
        query_point: Tuple (x, y, z)
        triangles: List of Triangle objects, each with vertices and normal
        bvh: Bounding Volume Hierarchy for efficient culling
        use_signed: If True, uses winding number; if False, uses parity
        
    Returns:
        Boolean (inside/outside) or Float (winding number)
    """
    ray_origin = query_point
    # Use positive Z axis for simplicity, or arbitrary direction
    ray_direction = (0.0, 0.0, 1.0) 
    
    crossing_count = 0
    total_count = 0
    
    # 1. Find candidate triangles using BVH
    candidate_indices = bvh.intersect_ray(ray_origin, ray_direction)
    
    for idx in candidate_indices:
        tri = triangles[idx]
        
        # 2. Ray-Triangle Intersection
        # Returns t (distance) and hit_point if intersected, else None
        hit = ray_triangle_intersect(ray_origin, ray_direction, tri)
        
        if hit is not None:
            t, hit_point = hit
            
            # Only count intersections in the forward direction
            if t > 0:
                total_count += 1
                
                if use_signed:
                    # 3. Signed Winding Number
                    # Dot product of triangle normal and ray direction
                    dot_product = dot(tri.normal, ray_direction)
                    
                    # Convention: +1 if exiting (normal points same direction as ray)
                    # -1 if entering (normal points opposite to ray)
                    if dot_product > 0.0:
                        crossing_count += 1
                    else:
                        crossing_count -= 1
                else:
                    # 4. Parity Method
                    crossing_count += 1
    
    if use_signed:
        # For closed outward-oriented mesh:
        # Outside -> 0, Inside -> 1
        # Graceful transition for open meshes
        return crossing_count
    else:
        # Parity: Odd -> Inside, Even -> Outside
        return (crossing_count % 2) == 1

def ray_triangle_intersect(origin, direction, triangle):
    """
    Müller-Trombose or standard Möller-Trumbore intersection.
    Returns (t, point) if hit, else None.
    """
    # Implementation of ray-triangle intersection
    # 1. Compute plane normal from triangle vertices
    v0 = triangle.v0
    v1 = triangle.v1
    v2 = triangle.v2
    
    edge1 = v1 - v0
    edge2 = v2 - v0
    normal = cross(edge1, edge2)
    
    # 2. Check if ray is parallel to plane
    denom = dot(normal, direction)
    if abs(denom) < 1e-8:
        return None
        
    # 3. Compute t
    t = dot(normal, v0 - origin) / denom
    
    if t <= 0:
        return None
        
    # 4. Compute hit point
    hit_point = origin + direction * t
    
    # 5. Barycentric coordinates to check if point is inside triangle
    # Using edge normals or area method
    # ... (Standard barycentric check) ...
    
    # Simplified check:
    # If hit_point is within triangle bounds
    if is_point_in_triangle(hit_point, v0, v1, v2):
        return (t, hit_point)
    else:
        return None

def is_point_in_triangle(p, v0, v1, v2):
    """Check if point p is inside triangle v0-v1-v2 using barycentric coords."""
    # Compute vectors
    v0v1 = v1 - v0
    v0v2 = v2 - v0
    v0p = p - v0
    
    # Compute dot products
    d00 = dot(v0v0, v0v0) # Not needed if using cross product method
    # Standard barycentric calculation
    # ...
    return True # Placeholder for actual barycentric logic
```

### Edge Cases and Pitfalls
-   **Edge/Vertex Hits**: Rays hitting triangle edges or vertices may double-count or miscount intersections, making the theoretical crossing count unreliable for single rays [S2, unofficial]. This is mitigated by using signed winding numbers (which cancel out grazing hits if normals are consistent) or multi-ray voting [S2, unofficial].
-   **Open Meshes**: The parity method is only meaningful for closed meshes. For open meshes or "soup" of triangles, the signed winding number provides a generalized result, but the binary inside/outside interpretation is invalid [S2, unofficial].
-   **Numerical Precision**: Floating-point errors near the surface can cause inconsistent results. Using a small epsilon for intersection checks is recommended [G].
-   **Performance**: Brute-force checking all triangles is $O(N)$. Using a BVH reduces this to $O(\log N)$ on average [S2, unofficial]. The 2D grid optimization reduces the problem from checking $128^3$ points to $128^2$ points by pre-calculating intersections [S1, unofficial].

## Operating characteristics

-   **Time Complexity**: $O(N)$ for brute-force ray-triangle intersection. $O(\log N)$ with BVH acceleration. The 2D grid pre-calculation is $O(N \cdot H \cdot W)$ but reduces query time to $O(1)$ per grid cell [S1, unofficial].
-   **Memory Complexity**: $O(N)$ for storing triangles. $O(H \cdot W)$ for the 2D grid optimization [S1, unofficial].
-   **Measured Performance**: The overhead of the raycast step is small when performed simultaneously with Signed Distance Field (SDF) calculation [S3, unofficial].
-   **Strengths**: Robust against sign-flipping artifacts caused by depth-based tests [S3, unofficial]. Provides generalized winding numbers for non-closed meshes [S2, unofficial].
-   **Limitations**: Sensitive to degenerate ray hits (edges/vertices) without mitigation [S2, unofficial]. Requires closed meshes for simple parity interpretation [S2, unofficial].
-   **When to Use**: When topological correctness is critical, especially in Marching Cubes implementations where incorrect inside/outside tests cause artifacts [S2, unofficial]. When dealing with meshes that may have holes or non-manifold geometry, the signed winding number is preferred [S2, unofficial].

## Background

The Ray Crossing Count method is a fundamental technique in computational geometry for point-in-polyhedron tests. It is widely used in volume rendering, particularly in Marching Cubes implementations, to correctly determine the interior of a mesh. The specific implementation using signed crossing counts to fix artifacts in Marching Cubes 33 was detailed by Vega et al. in "A Fast and Memory-Saving Marching Cubes 33 Implementation with the Correct Interior Test" (JCGT 2019) [S2, unofficial]. The 2D grid optimization approach is discussed in community forums as a way to reduce computational complexity for voxel-based rendering [S1, unofficial].

## Sources

[S1] Checking if point is inside the mesh via raycasting - where to cast rays?, https://gamedev.stackexchange.com/questions/120869/checking-if-point-is-inside-the-mesh-via-raycasting-where-to-cast-rays, fetched 2026-10-26 (unofficial)
[S2] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
[S3] Signed Distance Field Rendering Journey pt.1 – kosmonaut's blog, https://kosmonautblog.wordpress.com/2017/05/01/signed-distance-field-rendering-journey-pt-1/, fetched 2026-10-29 (unofficial)
