# Ray Tracing and Ray Intersection Methods

Also called: Ray Intersection Methods

## Overview
Ray tracing is a surface rendering technique that determines the first object intersected by a camera ray by analytically computing intersection distances for simple geometries [S1, unofficial]. It solves the problem of finding the exact point where a ray meets a surface in a single evaluation, providing mathematically correct intersections limited only by floating-point precision [S2, unofficial]. Unlike ray marching, which steps through the scene and approximates intersections, ray tracing computes the intersection directly, making it ideal for explicit triangle meshes and simple primitives [S2, unofficial].

## How it works
**Inputs and Outputs**
- Input: Ray defined by origin $\mathbf{o}$ and normalized direction $\hat{\mathbf{d}}$ [S2, unofficial]. Scene geometry (e.g., triangle meshes) and acceleration structure (BVH or kd-tree) [S2, unofficial].
- Output: Smallest positive distance $t$ along the ray where the intersection occurs, and the intersection point $\mathbf{p}(t)$ [S2, unofficial].

**Data Structures**
- **Ray**: Defined by origin $\mathbf{o}$ and direction $\hat{\mathbf{d}}$ [G].
- **Acceleration Structure**: Bounding Volume Hierarchy (BVH) or kd-tree [S2, unofficial]. Nodes contain bounding boxes. Leaves contain lists of primitives (e.g., triangles) [S2, unofficial].
- **Triangle**: Defined by three vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$ [S2, unofficial].

**Steps**
1. **Ray Definition**: Define the parametric ray equation:
   $$ \mathbf{p}(t) = \mathbf{o} + t\hat{\mathbf{d}} $$
   where $\mathbf{o}$ is the ray origin, $\hat{\mathbf{d}}$ is the normalized ray direction, and $t$ is the distance along the ray [S2, unofficial]. The goal is to find the smallest positive $t$ where $\mathbf{p}(t)$ lies on a surface [S2, unofficial].

2. **Acceleration Structure Traversal**: Use a BVH or kd-tree to avoid testing every primitive [S2, unofficial].
   - Traverse the tree: if a ray misses a bounding box, skip every primitive inside [S2, unofficial].
   - If the ray hits a leaf node, test the ray against the primitives in that leaf [S2, unofficial].

3. **Primitive Intersection**: For each primitive in the leaf, compute the intersection distance $t$.
   - **Triangle Intersection**: Test the ray against the plane containing the triangle, then use barycentric coordinates to determine whether the intersection point lies inside the triangle edges [S2, unofficial].
   - **Sphere Intersection**: Solve the quadratic equation derived from $\|\mathbf{o} + t\hat{\mathbf{d}} - \mathbf{c}\|^2 = r^2$, where $\mathbf{c}$ is the sphere center and $r$ is the radius [S2, unofficial]. The smaller positive root is the intersection distance [S2, unofficial].

4. **Closest Hit**: Among all intersections found, select the one with the smallest positive $t$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_trace(ray_origin, ray_direction, scene):
    """
    Traces a ray against a scene with a BVH acceleration structure.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, normalized direction of the ray.
        scene: Object containing the BVH root node and primitive data.
        
    Returns:
        t_hit: Distance to the closest intersection, or infinity if no hit.
        hit_point: 3D vector, the intersection point.
    """
    t_hit = float('inf')
    hit_point = None
    
    # Traverse BVH to find closest intersection
    stack = [scene.bvh_root]
    
    while stack:
        node = stack.pop()
        
        # Check if ray intersects node's bounding box
        if not ray_intersects_bbox(ray_origin, ray_direction, node.bbox):
            continue
            
        if node.is_leaf:
            # Test ray against primitives in leaf
            for primitive in node.primitives:
                t, hit = intersect_primitive(ray_origin, ray_direction, primitive)
                if t > 0 and t < t_hit:
                    t_hit = t
                    hit_point = hit
        else:
            # Push children to stack (order may affect performance)
            stack.append(node.left_child)
            stack.append(node.right_child)
            
    return t_hit, hit_point

def intersect_primitive(ray_origin, ray_direction, primitive):
    """
    Computes intersection with a specific primitive (e.g., triangle).
    
    Args:
        ray_origin: 3D vector.
        ray_direction: 3D vector.
        primitive: Triangle defined by vertices v0, v1, v2.
        
    Returns:
        t: Distance to intersection, or infinity if no hit.
        hit: 3D vector, intersection point.
    """
    # Example: Triangle intersection using Möller–Trumbore algorithm
    v0, v1, v2 = primitive.vertices
    edge1 = v1 - v0
    edge2 = v2 - v0
    h = cross(ray_direction, edge2)
    a = dot(edge1, h)
    
    if a < epsilon: # Ray parallel to triangle
        return float('inf'), None
        
    f = 1.0 / a
    s = ray_origin - v0
    u = f * dot(s, h)
    
    if u < 0.0 or u > 1.0:
        return float('inf'), None
        
    q = cross(s, edge1)
    v = f * dot(ray_direction, q)
    
    if v < 0.0 or u + v > 1.0:
        return float('inf'), None
        
    t = f * dot(edge2, q)
    
    if t > epsilon:
        hit = ray_origin + t * ray_direction
        return t, hit
    else:
        return float('inf'), None

def ray_intersects_bbox(ray_origin, ray_direction, bbox):
    """
    Checks if a ray intersects an axis-aligned bounding box.
    
    Args:
        ray_origin: 3D vector.
        ray_direction: 3D vector.
        bbox: Object with min and max 3D vectors.
        
    Returns:
        True if ray intersects bbox, False otherwise.
    """
    # Implementation of slab method for AABB intersection
    # Not given in the sources. [G]
    pass
```

**Parameters**
- **Acceleration Structure**: BVHs or kd-trees are required for large scenes to avoid testing every triangle [S2, unofficial].
- **Epsilon**: Small value to avoid self-intersection or precision issues (e.g., $10^{-5}$) [G].

**Edge Cases and Pitfalls**
- **Complex Geometry**: Fails when object geometry is too complex for analytical intersection solutions [S1, unofficial].
- **Procedural Geometry**: Procedural geometry is awkward; ray tracing must either tessellate the function into triangles (losing precision and creating enormous meshes) or write custom intersection shaders for each procedural type [S2, unofficial].
- **Deformation**: Moving, bending, or twisting geometry requires updating the mesh and rebuilding the acceleration structure [S2, unofficial].

## Subtopics

### [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)
Binary Search IsoSurface Extraction approximates the position of an isosurface between two sample points by iteratively halving the interval based on scalar field values. It solves the problem of locating the precise intersection of a ray with a scalar volume's isovalue surface without requiring analytical derivatives or complex root-finding algorithms. Unlike [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md), which assumes linear variation within a voxel, this method uses discrete sampling and bisection, making it robust for non-linear or noisy data but potentially slower due to multiple texture lookups. It serves as the geometric foundation for [Isocontour Shading with Curvature](3819-isocontour-shading-with-curvature.md) and [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md).

### [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md)
Trilinear Isosurface Intersection computes the exact intersection point between a ray and the isosurface defined by a threshold value within a single hexahedral voxel of a scalar grid. It solves the cubic polynomial derived from the trilinear interpolation of the eight corner values to find the intersection distance $t$. Unlike ray marching methods that step through the volume, this method provides a direct closed-form or iterative solution, offering superior performance for silhouette and secondary rays where precision is critical. It builds on [Ray Tracing and Ray Intersection Methods](5617-ray-tracing-and-ray-intersection-methods.md) by providing the specific algebraic formulation for the trilinear interpolant.

### [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md)
Neubauer Iso-surface Ray Casting is a single-kernel iso-surface ray casting method that uses a min/max octree built from macro-cells to accelerate ray traversal. It sets itself apart from its siblings by projecting macro-cell boundaries onto the image plane to create hexagonal footprints, allowing rays to skip entire subtrees of the octree based on 2D rasterization rather than 3D bounding box tests.

### [Hybrid Packetization Strategy](7606-hybrid-packetization-strategy.md)
The Hybrid Packetization Strategy is a ray-triangle intersection method designed to improve SIMD utilization on GPUs when standard packetization is inefficient due to sparse acceleration structures with high primitive density per leaf. It solves the problem of lost coherence in warp-level computations by dynamically switching between standard warp-level intersection and parallel processing of individual rays based on workload. It sets itself apart from standard packetization by distributing the work of a single ray across multiple threads when the number of intersection candidates is high, thereby maintaining throughput even when rays traverse cells with varying densities.

### [Ray Tracing](8334-ray-tracing.md)
Classic ray tracing (Whitted-style) is a recursive rendering model that captures shadows, specular reflection, refraction, and diffuse surface shading by tracing light paths from the image plane into the scene. It differs from simple ray casting by explicitly spawning secondary rays for reflection and refraction, and shadow rays for occlusion testing, thereby supporting native recursive ray propagation.

### Packet Ray List Merging
Packet Ray List Merging is an optimization where lists of adjacent pixels (typically 2x2 groups) are combined into a single list. This exploits spatial coherence where adjacent pixels share common visible blocks, reducing the number of lists generated and processed [S3, unofficial]. In Phase I, the BONO tree is traversed, and lists are created for each packet of rays (2x2 adjacent pixels) instead of individual pixels. If a group of pixels overlaps with the projection of the current node, the group's list is processed [S3, unofficial]. The upper bound $k$ on the list size for grouped pixels is increased compared to single pixels. Best performance is achieved when $k$ is around 20 for packets, compared to 12 for single pixels [S3, unofficial].

### Geometry Integrator
The Geometry Integrator is a direct illumination integrator for scene geometry that uses a hardcoded two-sided diffuse BSDF with red reflectance [0.8, 0.2, 0.2] [S4, unofficial]. It performs ray intersection against the scene (`itx : SurfaceInteraction3f = scene.ray_intersect(ray)`), adjusts the normal direction to point against the ray direction (`itx.n = dr.select(dr.dot(itx.n, ray.d) < 0, itx.n, -itx.n)`), loads a hardcoded two-sided diffuse BSDF, checks visibility using a shadow ray intersected against the scene (`itx_shadow : mi.SurfaceInteraction3f = scene.ray_intersect(rayL)`), and accumulates direct illumination if both primary and secondary rays are valid (`L += dr.select(valid_ray, Li, black)`) [S4, unofficial].

## Operating characteristics
**Time and Memory Complexity**
- **Time Complexity**: $O(\log N)$ for BVH traversal on average, where $N$ is the number of primitives [S2, unofficial]. However, worst-case can be $O(N)$ for poorly constructed BVHs or specific ray directions [G].
- **Memory Complexity**: $O(N)$ for storing the acceleration structure and primitives [S2, unofficial].

**Measured Performance**
- Modern GPUs accelerate both tree traversal and triangle intersection with dedicated ray tracing hardware (RT cores), making real-time ray tracing feasible for games and interactive applications [S2, unofficial].
- Exact intersections are limited only by floating-point precision [S2, unofficial].

**Scaling**
- Scales well with hardware acceleration (RT cores) [S2, unofficial].
- Memory for large scenes can be a bottleneck; detailed triangle meshes and their acceleration structures can consume gigabytes of memory [S2, unofficial].

**Strengths and Limitations**
- **Strengths**: Exact intersections, hardware acceleration, mesh compatibility [S2, unofficial].
- **Limitations**: Procedural geometry is awkward, smooth blending is difficult, deformation is expensive, memory for large scenes [S2, unofficial].

**When to Use**
- Use when exact intersections are required, and the scene is composed of explicit geometry (e.g., triangle meshes) [S2, unofficial].
- Use when hardware acceleration is available [S2, unofficial].

**When Not to Use**
- Do not use when the scene is defined by procedural functions or SDFs, as ray marching is more suitable [S2, unofficial].
- Do not use when deformation is frequent and expensive to update in the acceleration structure [S2, unofficial].

## Background
Ray tracing was introduced as a surface rendering technique to determine the first object intersected by a camera ray [S1, unofficial]. It has evolved with hardware acceleration, particularly with the introduction of RT cores on NVIDIA GPUs, making real-time ray tracing feasible [S2, unofficial]. It is widely used in offline rendering and increasingly in real-time applications [S2, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S2] ray marching vs ray tracing, https://tutorials.tektite.studio/ray-marching/ray-marching-vs-ray-tracing/, fetched 2026-10-27 (unofficial)
[S3] PDF Interactive High Resolution Isosurface Ray Tracing on Multi-Core ... - UMD, https://www.umiacs.umd.edu/~joseph/isosurface-paper.pdf, fetched 2026-10-26 (unofficial)
[S4] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
