# Ray-Splat Intersection

Also called: IntersectionTester, Raycaster Splat Intersection, Raycaster.castRayAtSplatTreeNode, Raycaster.intersectSplatMesh, Raycasting via BVH, Splat Picking

## Overview
Ray-Splat Intersection determines which 3D Gaussian splats in a scene are intersected by a camera ray, returning the closest hit for depth testing, picking, or alpha blending. It distinguishes itself from sibling methods by supporting both approximate spherical intersection (for performance) and true ellipsoid intersection (for geometric accuracy) within a hierarchical Bounding Volume Hierarchy (BVH) structure [S1, S2, unofficial]. Unlike [Splat Raycasting Dispatch](6924-splat-raycasting-dispatch.md), which focuses on the parallel dispatch mechanism, this method defines the geometric traversal and intersection logic itself [S1, S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Camera parameters: Origin $\mathbf{o}$, direction $\mathbf{d}$ (normalized), projection type (Perspective/Orthographic), and screen coordinates $(x, y)$ for ray generation.
*   Scene data: A collection of splats, each defined by center $\mathbf{c}$, scale vector $\mathbf{s} = (s_x, s_y, s_z)$, rotation quaternion $\mathbf{q}$, and opacity $\alpha$ (stored in color channel $w$).
*   BVH: A Bounding Volume Hierarchy constructed from axis-aligned bounding boxes (AABBs) of the splats.
*   Flags: `raycastAgainstTrueSplatEllipsoid` (boolean), `splatRenderMode` (TwoD or ThreeD), `scaleEpsilon`.

**Outputs:**
*   A sorted list of hit records, each containing the splat index, intersection distance $t$, hit point $\mathbf{p}$, and normal $\mathbf{n}$ (if required).

### Data Structures
*   **Splat Data:**
    *   `center`: `vec3` (world space position).
    *   `scale`: `vec3` (local scale factors).
    *   `rotation`: `quat` (local orientation).
    *   `opacity`: `float` (derived from color $w$).
*   **BVH Node:**
    *   `boundingBox`: `AABB` (min/max extents in local subtree space).
    *   `indexes`: `uint32[]` (list of splat indices contained in this leaf or subtree).
    *   `subTrees`: `Node[]` (children, if internal node).
*   **Hit Record:**
    *   `splatIndex`: `uint32`.
    *   `distance`: `float` (from ray origin to hit point).
    *   `hitPoint`: `vec3` (world space).
    *   `normal`: `vec3` (world space, optional).

### Parameters
*   `scaleEpsilon`: `0.0000001`. Threshold for ignoring degenerate splats [S1, unofficial].
*   `maxDistance`: `100.0`. Maximum ray length for approximate sampling in [S2, unofficial].
*   `resolution`: `1.0`. Step size for sampling in [S2, unofficial].
*   `raycastAgainstTrueSplatEllipsoid`: `true` for accurate ellipsoid intersection, `false` for spherical approximation [S1, unofficial].

### Steps

1.  **Ray Generation**:
    *   **Perspective Camera**:
        1.  Compute Normalized Device Coordinates (NDC):
            $$ \text{ndc}_x = \frac{x}{\text{width}} \cdot 2.0 - 1.0 $$
            $$ \text{ndc}_y = \left( \frac{\text{height} - y}{\text{height}} \right) \cdot 2.0 - 1.0 $$
        2.  Set ray origin $\mathbf{o}$ to the camera world position.
        3.  Unproject NDC coordinates to world space to get a point $\mathbf{p}_{\text{ndc}}$.
        4.  Set ray direction $\mathbf{d} = \text{normalize}(\mathbf{p}_{\text{ndc}} - \mathbf{o})$.
    *   **Orthographic Camera**:
        1.  Unproject NDC coordinates with $z$ derived from camera near/far planes.
        2.  Set ray origin $\mathbf{o}$ to the unprojected point.
        3.  Set ray direction $\mathbf{d}$ to the camera's forward vector $(0,0,-1)$ transformed by the camera's world rotation matrix.

2.  **BVH Construction** (if not pre-built):
    1.  Iterate over all splats.
    2.  Compute the AABB for each splat based on its center, scale, and rotation.
    3.  Build a BVH from these AABBs.

3.  **Traversal**:
    1.  For each root node of the splat tree (or subtree):
        1.  Compute `fromLocal` (world matrix $\times$ scene transform if dynamic) and `toLocal` (inverse of `fromLocal`).
        2.  Transform ray origin $\mathbf{o}$ and direction $\mathbf{d}$ to local space:
            $$ \mathbf{o}_{\text{local}} = \text{toLocal} \cdot (\mathbf{o}, 1) $$
            $$ \mathbf{d}_{\text{local}} = \text{normalize}(\text{toLocal} \cdot (\mathbf{o} + \mathbf{d}, 0) - \mathbf{o}_{\text{local}}) $$
        3.  Call `castRayAtSplatTreeNode` with local ray, node, and hit list.

4.  **Node Intersection** (`castRayAtSplatTreeNode`):
    1.  Check if ray intersects node bounding box:
        $$ \text{if } \neg \text{rayIntersectBox}(\mathbf{o}_{\text{local}}, \mathbf{d}_{\text{local}}, \text{node.boundingBox}) \text{ return} $$
    2.  If leaf node:
        1.  Iterate over `node.indexes`.
        2.  For each splat index:
            1.  Retrieve splat data ($\mathbf{c}, \mathbf{s}, \mathbf{q}, \alpha$).
            2.  **Degeneracy Check**: If any component of $\mathbf{s} \le \text{scaleEpsilon}$, skip. If `ThreeD` mode and $s_z \le \text{scaleEpsilon}$, skip [S1, unofficial].
            3.  **Intersection Test**:
                *   **Approximate Sphere** (if `raycastAgainstTrueSplatEllipsoid` is false):
                    1.  Compute radius:
                        $$ r = \frac{s_x + s_y}{2} $$
                        If `ThreeD` mode:
                        $$ r = \frac{s_x + s_y + s_z}{3} $$
                    2.  Test intersection of ray with sphere at $\mathbf{c}$ with radius $r$.
                *   **True Ellipsoid** (if `raycastAgainstTrueSplatEllipsoid` is true):
                    1.  Compute uniform scale factor from opacity:
                        $$ k = \log_{10}(\alpha) \cdot 2.0 $$
                    2.  Construct matrices:
                        *   Scale matrix $S = \text{diag}(s_x, s_y, s_z)$.
                        *   Rotation matrix $R$ from quaternion $\mathbf{q}$.
                        *   Uniform scale matrix $U = \text{diag}(k, k, k)$.
                    3.  Compute transformation from sphere space to world space:
                        $$ M_{\text{world}} = U \cdot R \cdot S $$
                    4.  Compute inverse transformation:
                        $$ M_{\text{sphere}} = M_{\text{world}}^{-1} $$
                    5.  Transform ray to sphere space:
                        $$ \mathbf{o}_{\text{sphere}} = M_{\text{sphere}} \cdot (\mathbf{o}_{\text{local}} - \mathbf{c}) $$
                        $$ \mathbf{d}_{\text{sphere}} = \text{normalize}(M_{\text{sphere}} \cdot (\mathbf{o}_{\text{local}} + \mathbf{d}_{\text{local}} - \mathbf{c}) - \mathbf{o}_{\text{sphere}}) $$
                    6.  Test intersection of ray $(\mathbf{o}_{\text{sphere}}, \mathbf{d}_{\text{sphere}})$ with unit sphere (radius 1.0).
                    7.  If intersecting, compute hit point $\mathbf{p}_{\text{sphere}}$, transform back to world space:
                        $$ \mathbf{p}_{\text{world}} = M_{\text{world}} \cdot \mathbf{p}_{\text{sphere}} + \mathbf{c} $$
                        Record hit.
    3.  If internal node:
        1.  Recursively call `castRayAtSplatTreeNode` for each child.

5.  **Post-Processing**:
    1.  Transform all hit points and normals back to world space using `fromLocal`.
    2.  Compute distance $t = \|\mathbf{p}_{\text{world}} - \mathbf{o}\|$.
    3.  Sort hits by distance ascending.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_splat_intersection(ray_origin, ray_direction, splat_tree, camera, use_true_ellipsoid=True, scale_epsilon=1e-7):
    """
    Intersects a ray against a splat mesh using a BVH.
    Returns a list of hits sorted by distance.
    """
    hits = []
    
    # 1. Ray Generation (Simplified for Perspective)
    # In practice, this depends on camera model. 
    # Assuming ray_origin and ray_direction are already provided in world space.
    
    # 2. Traverse Tree
    for subtree in splat_tree.sub_trees:
        # Compute local transforms
        world_matrix = subtree.world_matrix
        if subtree.dynamic_mode:
            world_matrix = world_matrix * subtree.scene_transform
        to_local = inverse(world_matrix)
        from_local = world_matrix
        
        # Transform ray to local space
        o_local = transform_point(to_local, ray_origin)
        d_local = normalize(transform_vector(to_local, ray_direction))
        local_ray = Ray(o_local, d_local)
        
        # Recursively cast
        cast_ray_at_node(local_ray, subtree.root_node, from_local, hits)
        
    # 3. Sort Hits
    hits.sort(key=lambda h: h.distance)
    return hits

def cast_ray_at_node(ray, node, from_local, hits):
    """
    Recursive BVH traversal.
    """
    # Check bounding box intersection
    if not ray_intersects_box(ray.origin, ray.direction, node.bounding_box):
        return

    if node.is_leaf:
        for splat_index in node.indexes:
            splat = get_splat_data(splat_index)
            
            # Degeneracy check
            if any(s <= scale_epsilon for s in splat.scale):
                continue
            if splat.render_mode == "ThreeD" and splat.scale.z <= scale_epsilon:
                continue
                
            hit = None
            if not use_true_ellipsoid:
                # Approximate Sphere
                radius = (splat.scale.x + splat.scale.y) / 2.0
                if splat.render_mode == "ThreeD":
                    radius = (splat.scale.x + splat.scale.y + splat.scale.z) / 3.0
                hit = ray_sphere_intersection(ray, splat.center, radius)
            else:
                # True Ellipsoid
                # Compute opacity-based uniform scale
                opacity = splat.color.w
                uniform_scale_factor = log10(opacity) * 2.0
                
                # Construct matrices
                scale_mat = diag(splat.scale.x, splat.scale.y, splat.scale.z)
                rot_mat = quaternion_to_matrix(splat.rotation)
                uniform_mat = diag(uniform_scale_factor, uniform_scale_factor, uniform_scale_factor)
                
                # World to Sphere transformation
                # M_world = U * R * S
                # M_sphere = M_world^-1
                m_world = uniform_mat * rot_mat * scale_mat
                m_sphere = inverse(m_world)
                
                # Transform ray to sphere space
                # o_sphere = M_sphere * (o_local - c)
                o_sphere = transform_point(m_sphere, ray.origin - splat.center)
                
                # d_sphere = normalize(M_sphere * (o_local + d - c) - o_sphere)
                d_end_local = ray.origin + ray.direction
                d_end_sphere = transform_point(m_sphere, d_end_local - splat.center)
                d_sphere = normalize(d_end_sphere - o_sphere)
                
                local_ray_sphere = Ray(o_sphere, d_sphere)
                hit = ray_sphere_intersection(local_ray_sphere, vec3(0,0,0), 1.0)
                
                if hit:
                    # Transform hit point back to world space
                    hit_point_sphere = hit.point
                    hit_point_world = transform_point(m_world, hit_point_sphere) + splat.center
                    hit.distance = length(hit_point_world - ray.origin)
                    hit.point = hit_point_world
                    # Normal calculation omitted for brevity, requires Jacobian of transform
            
            if hit:
                # Transform normal if needed
                if hit.normal is not None:
                    hit.normal = transform_vector(from_local, hit.normal)
                hits.append(hit)
    else:
        # Internal node: recurse
        for child in node.children:
            cast_ray_at_node(ray, child, from_local, hits)

def ray_sphere_intersection(ray_origin, sphere_center, radius):
    """
    Standard ray-sphere intersection.
    Returns hit record or None.
    """
    oc = ray_origin - sphere_center
    a = dot(ray.direction, ray.direction)
    b = 2.0 * dot(oc, ray.direction)
    c = dot(oc, oc) - radius * radius
    discriminant = b * b - 4 * a * c
    
    if discriminant < 0:
        return None
    
    sqrt_disc = sqrt(discriminant)
    t1 = (-b - sqrt_disc) / (2.0 * a)
    t2 = (-b + sqrt_disc) / (2.0 * a)
    
    t = t1 if t1 > 0 else t2
    if t < 0:
        return None
        
    point = ray_origin + ray.direction * t
    normal = normalize(point - sphere_center)
    
    return HitRecord(distance=t, point=point, normal=normal)
```

### Edge Cases and Pitfalls
*   **Degenerate Splats**: Splats with scale components $\le \text{scaleEpsilon}$ ($10^{-7}$) are skipped to prevent division by zero or invalid geometry during matrix inversion [S1, unofficial].
*   **Null Data**: If `renderData` or `camera` is null, the intersection test returns null and logs an error [S2, unofficial].
*   **BVH Failure**: If BVH construction fails (`bvh === null`), the test returns null [S2, unofficial].
*   **Vertex Count Changes**: If the number of vertices changes between calls, a warning is logged, but execution continues [S2, unofficial].
*   **Precision**: Using `log10` for opacity scaling assumes opacity is in $(0, 1]$. Opacity of 0 results in $-\infty$, which should be handled or clamped [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \log N)$ for BVH construction, $O(\log N)$ average case for ray traversal, where $N$ is the number of splats. Worst-case is $O(N)$ if the BVH is degenerate.
*   **Memory Complexity**: $O(N)$ for storing splat data and BVH nodes.
*   **Performance**: True ellipsoid intersection is more computationally expensive than spherical approximation due to matrix multiplications and inversions [S1, unofficial].
*   **Strengths**: Supports both approximate and exact intersection; efficient culling via BVH; handles dynamic scenes via local space transformation [S1, unofficial].
*   **Limitations**: BVH construction can be costly for dynamic scenes if rebuilt every frame [S2, unofficial]. True ellipsoid intersection may be too slow for real-time rendering on low-end hardware [S1, unofficial].

## Background
Ray-Splat Intersection is a core component of real-time 3D Gaussian Splatting renderers, particularly those implemented in JavaScript/WebGL for browser-based viewing [S1, S2, unofficial]. It was developed to enable efficient picking and depth testing in large-scale Gaussian splat scenes. The method leverages BVH structures to accelerate ray-primitive intersection, a technique common in ray tracing but adapted here for the specific geometry of 3D Gaussians [S1, S2, unofficial].

## Sources
[S1] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
[S2] huggingface/gsplat.js, https://github.com/huggingface/gsplat.js, fetched 2026-10-27 (unofficial)
