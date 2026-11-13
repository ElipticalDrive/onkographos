# Ray Tracing and Ray Intersection Methods

Also called: Ray Intersection Methods

## Overview
Ray tracing is a surface rendering technique that determines the first object intersected by a camera ray by analytically computing intersection distances for explicit geometry [S1, unofficial]. It solves the problem of finding the exact point where a ray meets a surface in a single evaluation, providing mathematically correct intersections limited only by floating-point precision [S3, unofficial]. Unlike [Ray Marching](8334-ray-marching.md) which advances a ray in repeated steps to approximate intersections, ray tracing solves specific mathematical equations (e.g., quadratics for spheres, barycentric tests for triangles) to find the hit point directly [S3, unofficial]. It differs from simple [Ray Casting](8334-ray-casting.md) by explicitly supporting recursive light paths for reflections, refractions, and shadows, generating more realistic images at higher computational cost [S2, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A ray defined by origin $\mathbf{o}$ and normalized direction $\hat{\mathbf{d}}$; a scene representation consisting of explicit geometry (e.g., triangle meshes) and an acceleration structure (e.g., BVH) [S3, unofficial].
- **Output**: The smallest positive distance $t$ along the ray to the first intersected primitive, and the intersection point $\mathbf{p} = \mathbf{o} + t\hat{\mathbf{d}}$ [S3, unofficial].

**Data Structures**
- **Ray**: Defined by origin $\mathbf{o}$ and direction $\hat{\mathbf{d}}$ [G].
- **Acceleration Structure**: Bounding Volume Hierarchy (BVH) or kd-tree. Nodes contain bounding boxes; leaf nodes contain lists of primitives (e.g., triangles) [S3, unofficial].
- **Primitive**: For triangles, defined by vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$ [S3, unofficial].

**Steps**
1. **Ray Definition**: Define the parametric ray equation:
   $$ \mathbf{p}(t) = \mathbf{o} + t\hat{\mathbf{d}} $$
   where $t > 0$ is the distance along the ray [S3, unofficial].
2. **Acceleration Structure Traversal**: Traverse the BVH to identify candidate primitives. If a ray misses a bounding box, skip all primitives inside it [S3, unofficial].
3. **Primitive Intersection**: For each candidate primitive, solve the intersection equation:
   - **Sphere**: For a sphere at center $\mathbf{c}$ with radius $r$, solve $\|\mathbf{o} + t\hat{\mathbf{d}} - \mathbf{c}\|^2 = r^2$ for the smallest positive root [S3, unofficial].
   - **Triangle**: Test the ray against the plane containing the triangle, then use barycentric coordinates to determine if the intersection point lies inside the triangle edges [S3, unofficial].
4. **Closest Hit Selection**: Select the primitive with the smallest positive $t$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_trace(ray_origin, ray_direction, scene):
    """
    Trace a ray against a scene to find the closest intersection.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, normalized direction of the ray.
        scene: Object containing acceleration structure and primitives.
        
    Returns:
        t_hit: Distance to closest intersection, or infinity if no hit.
        hit_point: 3D vector, intersection point.
        hit_primitive: Identifier of the hit primitive.
    """
    t_hit = float('inf')
    hit_point = None
    hit_primitive = None
    
    # Step 1: Traverse acceleration structure to find candidate primitives
    candidates = scene.bvh_traverse(ray_origin, ray_direction)
    
    # Step 2: Test each candidate primitive
    for primitive in candidates:
        t = primitive.intersect(ray_origin, ray_direction)
        
        # Step 3: Update closest hit if this intersection is closer
        if 0 < t < t_hit:
            t_hit = t
            hit_point = ray_origin + t * ray_direction
            hit_primitive = primitive
            
    return t_hit, hit_point, hit_primitive
```

**Parameters**
- **Acceleration Structure**: BVHs or kd-trees are required for large scenes to avoid testing every triangle [S3, unofficial].
- **Epsilon**: Small values to handle precision issues (e.g., $10^{-5}$ for intersection, $10^{-3}$ for ray marching step size) [S5, unofficial].

**Edge Cases and Pitfalls**
- **Complex Geometry**: Fails when object geometry is too complex for analytical intersection solutions [S1, unofficial].
- **Procedural Geometry**: Awkward to handle; requires tessellation into triangles (losing precision) or custom intersection shaders [S3, unofficial].
- **Deformation**: Moving or twisting geometry requires updating the mesh and rebuilding the acceleration structure, which is expensive [S3, unofficial].
- **Memory**: Detailed triangle meshes and acceleration structures can consume gigabytes of memory [S3, unofficial].

## Subtopics

### [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)
Binary Search IsoSurface Extraction approximates the position of an isosurface between two sample points by iteratively halving the interval based on scalar field values. It solves the problem of accurately locating the zero-crossing of a scalar field along a ray without requiring explicit polygon extraction. Unlike [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md), which computes an analytical intersection using trilinear interpolation, this method uses a fixed number of iterative bisection steps to refine the hit position. It is distinct from [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md) in that it does not rely on specific acceleration structures for traversal but rather refines a detected interval.

### [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md)
Trilinear Isosurface Intersection computes the exact intersection point between a ray and an isosurface defined by a threshold value within a single hexahedral voxel of a scalar grid. It solves the problem of finding the precise surface location without the approximation errors of linear interpolation or the iterative convergence costs of ray marching. Unlike [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) which relies on iterative subdivision, or [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md) which uses iterative root-finding, this method formulates the intersection as a cubic polynomial derived from trilinear interpolation and solves it analytically or via robust numerical methods. It builds on [Ray Tracing and Ray Intersection Methods](5617-ray-tracing-and-ray-intersection-methods.md) by providing the specific algebraic mechanism for hexahedral cells.

### [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md)
Neubauer Iso-surface Ray Casting is a visibility testing method used to determine which parts of extracted line loops or sparse surface features are occluded by the isosurface, avoiding the computational cost of extracting and rendering the entire isosurface mesh. It sets itself apart from siblings like [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) and [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) by focusing on binary occlusion checks for specific vertices rather than precise surface point extraction, and differs from [Ray Tracing](8334-ray-tracing.md) by using a coarse macro-cell traversal structure optimized for sparse feature visibility.

### [Hybrid Packetization Strategy](7606-hybrid-packetization-strategy.md)
The Hybrid Packetization Strategy is a ray-triangle intersection method designed to improve SIMD utilization on GPUs when standard packetization is inefficient due to sparse acceleration structures with high primitive density per leaf. It solves the problem of lost coherence in warp-level computations by dynamically switching between standard warp-level intersection and parallel processing of individual rays based on workload. It sets itself apart from standard packetization by distributing the work of a single ray across multiple threads when the number of intersection candidates is high, thereby maintaining throughput even when rays traverse cells with varying densities.

### [Ray Tracing](8334-ray-tracing.md)
Classic ray tracing (Whitted-style) is a recursive rendering model that captures shadows, specular reflection, refraction, and diffuse surface shading by tracing light paths from the image plane into the scene. It differs from simple ray casting by explicitly spawning secondary rays for reflection and refraction, and shadow rays for occlusion testing, thereby supporting native recursive ray propagation.

### [Multi-Sphere Ray Tracing with Diffuse Lighting](11351-multi-sphere-ray-tracing-with-diffuse-lighting.md)
This method casts a primary ray against a fixed set of spheres to find the closest intersection and computes unshadowed diffuse lighting from multiple point lights. It differs from [Ray Tracing](8334-ray-tracing.md) by omitting recursive shadow rays and reflections, and from [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) by using explicit geometric primitives rather than implicit field functions.

### [Screen-Space Ray Tracing with Depth Buffer](13547-screen-space-ray-tracing-with-depth-buffer.md)
Screen-Space Ray Tracing with Depth Buffer (SSRT) is a technique to trace rays in screen space against a depth buffer to find surface intersections, supporting up to four depth layers per pixel. It solves the problem of efficiently determining ray-surface intersections for effects like reflections, refractions, and ambient occlusion without traversing complex scene graphs or voxel structures. What sets it apart from [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) and [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) is that it operates directly on the rendered depth buffer rather than a volumetric or implicit representation, allowing it to capture any geometry already present in the frame buffer. Unlike [Ray Tracing](8334-ray-tracing.md) which traverses acceleration structures, SSRT traverses the 2D screen grid, making it highly parallelizable on GPUs but limited to visible surfaces.

### Packet Ray List Merging
Packet Ray List Merging is an optimization where lists of adjacent pixels (typically 2x2 groups) are combined into a single list. This exploits spatial coherence where adjacent pixels share common visible blocks, reducing the number of lists generated and processed [S4, unofficial]. In Phase I, the BONO tree is traversed, and lists are created for each packet of rays instead of individual pixels. If a group of pixels overlaps with the projection of the current node, the group's list is processed. The upper bound $k$ on the list size for grouped pixels is increased compared to single pixels; best performance is achieved when $k$ is around 20 for packets, compared to 12 for single pixels [S4, unofficial].

### Geometry Integrator
The Geometry Integrator is a direct illumination integrator for scene geometry that uses a hardcoded two-sided diffuse BSDF with red reflectance $[0.8, 0.2, 0.2]$ [S5, unofficial]. It performs ray intersection against the scene (`itx : SurfaceInteraction3f = scene.ray_intersect(ray)`), adjusts the normal direction to point against the ray direction (`itx.n = dr.select(dr.dot(itx.n, ray.d) < 0, itx.n, -itx.n)`), loads the BSDF, checks visibility using a shadow ray intersected against the scene (`itx_shadow : mi.SurfaceInteraction3f = scene.ray_intersect(rayL)`), and accumulates direct illumination if both primary and secondary rays are valid (`L += dr.select(valid_ray, Li, black)`) [S5, unofficial].

## Operating characteristics
- **Time Complexity**: $O(\log N)$ for BVH traversal per ray, where $N$ is the number of primitives, plus $O(1)$ for primitive intersection tests [S3, unofficial].
- **Memory Complexity**: $O(N)$ for storing the acceleration structure and geometry [S3, unofficial].
- **Measured Performance**: Modern GPUs accelerate ray-triangle intersection with dedicated hardware (RT cores), making real-time ray tracing feasible for games and interactive applications [S3, unofficial].
- **Strengths**: Exact intersections; no stepping artifacts; hardware acceleration; mesh compatibility [S3, unofficial].
- **Limitations**: Computationally more expensive than ray casting and ray marching [S2, unofficial]; procedural geometry is awkward; deformation is expensive; high memory usage for large scenes [S3, unofficial].
- **When to Use**: When exact intersections are required, and the scene consists of explicit geometry (e.g., triangle meshes) [S3, unofficial].
- **When Not to Use**: When the scene is defined by procedural functions or SDFs, or when performance is critical and approximate intersections are acceptable [S3, unofficial].

## Background
Ray tracing originated as a surface rendering technique to determine the first object intersected by a camera ray [S1, unofficial]. It was developed as a more sophisticated algorithm than ray casting, involving diffraction and reflection to generate more realistic images [S2, unofficial]. The method computes rays not only from the camera to the object, but also from the object to the light source to check for lighting, and implements reflection and diffraction [S2, unofficial]. It is computationally more expensive than ray casting and ray marching [S2, unofficial]. Modern implementations leverage acceleration structures like BVHs and kd-trees to handle large scenes efficiently [S3, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S2] http://www.hugi.scene.org/online/hugi37/hugi%2037%20-%20coding%20adok%20on%20ray%20casting,%20ray%20tracing,%20ray%20marching%20and%20the%20like.htm, http://www.hugi.scene.org/online/hugi37/hugi%2037%20-%20coding%20adok%20on%20ray%20casting,%20ray%20tracing,%20ray%20marching%20and%20the%20like.htm, fetched 2026-10-26 (unofficial)
[S3] ray marching vs ray tracing, https://tutorials.tektite.studio/ray-marching/ray-marching-vs-ray-tracing/, fetched 2026-10-27 (unofficial)
[S4] PDF Interactive High Resolution Isosurface Ray Tracing on Multi-Core ... - UMD, https://www.umiacs.umd.edu/~joseph/isosurface-paper.pdf, fetched 2026-10-26 (unofficial)
[S5] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
