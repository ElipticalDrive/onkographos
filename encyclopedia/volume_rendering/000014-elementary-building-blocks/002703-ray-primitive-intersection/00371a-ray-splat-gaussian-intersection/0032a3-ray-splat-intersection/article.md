# Ray-Splat Intersection

Also called: IntersectionTester, Raycaster Splat Intersection, Raycaster.castRayAtSplatTreeNode, Raycaster.intersectSplatMesh, Raycasting via BVH, Splat Picking

## Overview
Ray-Splat Intersection determines which 3D Gaussian splats in a scene are intersected by a camera ray, returning the closest hit for depth testing, picking, or alpha blending [S1, S2, unofficial]. It distinguishes itself from sibling methods by supporting both approximate spherical intersection (for performance) and true ellipsoid intersection (for geometric accuracy) within a hierarchical Bounding Volume Hierarchy (BVH) structure [S1, S2, unofficial]. Unlike [Splat Raycasting Dispatch](6924-splat-raycasting-dispatch.md), which focuses on the parallel dispatch mechanism, this method defines the geometric traversal and intersection logic itself [S1, S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Camera parameters: Origin $\mathbf{o}$, direction $\mathbf{d}$ (normalized), projection type (Perspective/Orthographic), and screen coordinates $(x, y)$ for ray generation [S1, S2, unofficial].
*   Scene data: A collection of splats, each defined by center $\mathbf{c}$, scale vector $\mathbf{s} = (s_x, s_y, s_z)$, rotation quaternion $\mathbf{q}$, and opacity $\alpha$ (stored in color channel $w$) [S1, unofficial].
*   BVH: A Bounding Volume Hierarchy constructed from axis-aligned bounding boxes (AABBs) of the splats [S1, S2, unofficial].
*   Flags: `raycastAgainstTrueSplatEllipsoid` (boolean), `splatRenderMode` (TwoD or ThreeD), and `scaleEpsilon` [S1, unofficial].

**Outputs:**
*   A sorted list of hit records, each containing the splat index, intersection distance $t$, hit point $\mathbf{p}$, and normal $\mathbf{n}$ (if required) [S1, unofficial].

### Data Structures
*   **Splat Data:**
    *   `center`: `vec3` (world space position).
    *   `scale`: `vec3` (local scale factors).
    *   `rotation`: `quat` (local orientation).
    *   `color`: `vec4` (RGB + opacity $\alpha$ in $w$) [S1, unofficial].
*   **BVH Node:**
    *   `boundingBox`: `AABB` (min/max extents in local subtree space).
    *   `indexes`: `uint32[]` (list of splat indices contained in this leaf or subtree) [S1, unofficial].
    *   `subTrees`: `Node[]` (children, if internal node) [S1, unofficial].
*   **Hit Record:**
    *   `splatIndex`: `uint32`.
    *   `distance`: `float` (from ray origin to hit point).
    *   `hitPoint`: `vec3` (world space).
    *   `normal`: `vec3` (world space, optional) [S1, unofficial].

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
        2.  Set ray origin $\mathbf{o}$ to the camera world position [S1, unofficial].
        3.  Unproject NDC coordinates to world space to get a point $\mathbf{p}_{\text{ndc}}$ [S1, unofficial].
        4.  Set ray direction $\mathbf{d} = \text{normalize}(\mathbf{p}_{\text{ndc}} - \mathbf{o})$ [S1, unofficial].
    *   **Orthographic Camera**:
        1.  Unproject NDC coordinates with $z$ derived from camera near/far planes using $z = \frac{\text{near} + \text{far}}{\text{near} - \text{far}}$ [S1, unofficial].
        2.  Set ray origin $\mathbf{o}$ to the unprojected point [S1, unofficial].
        3.  Set ray direction $\mathbf{d}$ to the camera's forward vector $(0,0,-1)$ transformed by the camera's world rotation matrix [S1, unofficial].

2.  **BVH Construction** (if not pre-built):
    1.  Iterate over all splat offsets [S2, unofficial].
    2.  Extract each splat's bounds into a boxes array [S2, unofficial].
    3.  Compute a global bounding box [S2, unofficial].
    4.  Construct a BVH from the boxes array [S2, unofficial].

3.  **Traversal**:
    1.  For each root node of the splat tree (or subtree):
        1.  Compute `fromLocal` (world matrix $\times$ scene transform if dynamic) and `toLocal` (inverse of `fromLocal`) [S1, unofficial].
        2.  Transform ray origin $\mathbf{o}$ and direction $\mathbf{d}$ to local space [S1, unofficial].
        3.  Call `castRayAtSplatTreeNode` with local ray, tree, subtree root node, and output hits array [S1, unofficial].

4.  **Node Intersection** (`castRayAtSplatTreeNode`):
    1.  Check if ray intersects node bounding box: `if (!ray.intersectBox(node.boundingBox)) return;` [S1, unofficial].
    2.  If leaf node:
        1.  Iterate over `node.data.indexes` [S1, unofficial].
        2.  For each splat global index, get scene index and check if scene is visible. If not visible, continue [S1, unofficial].
        3.  Retrieve splat color, center, scale, and rotation [S1, unofficial].
        4.  **Degeneracy Check**: If any scale component is $\le \text{scaleEpsilon}$, or if `ThreeD` mode and $s_z \le \text{scaleEpsilon}$, skip splat [S1, unofficial].
        5.  **Intersection Test**:
            *   **Approximate Sphere** (if `raycastAgainstTrueSplatEllipsoid` is false):
                1.  Compute radius: $r = \frac{s_x + s_y}{2}$ [S1, unofficial].
                2.  If `ThreeD` mode: $r = \frac{s_x + s_y + s_z}{3}$ [S1, unofficial].
                3.  Check intersection with sphere at $\mathbf{c}$ with radius $r$ [S1, unofficial].
            *   **True Ellipsoid** (if `raycastAgainstTrueSplatEllipsoid` is true):
                1.  Construct a scale matrix from the splat's scale vector [S1, unofficial].
                2.  Construct a rotation matrix from the splat's quaternion [S1, unofficial].
                3.  Calculate a uniform scale factor from the splat's opacity: $k = \log_{10}(\alpha) \cdot 2.0$ [S1, unofficial].
                4.  Construct a uniform scale matrix and combine it with rotation and scale matrices to form the transformation from sphere space to world space: $M_{\text{world}} = U \cdot R \cdot S$ [S1, unofficial].
                5.  Compute the inverse transformation matrix to map from world space to sphere space: $M_{\text{sphere}} = M_{\text{world}}^{-1}$ [S1, unofficial].
                6.  Transform the ray origin to sphere space: $\mathbf{o}_{\text{sphere}} = M_{\text{sphere}} \cdot (\mathbf{o}_{\text{local}} - \mathbf{c})$ [S1, unofficial].
                7.  Transform the ray direction to sphere space: $\mathbf{d}_{\text{sphere}} = \text{normalize}(M_{\text{sphere}} \cdot (\mathbf{o}_{\text{local}} + \mathbf{d}_{\text{local}} - \mathbf{c}) - \mathbf{o}_{\text{sphere}})$ [S1, unofficial].
                8.  Perform a ray-sphere intersection test in sphere space with radius $1.0$ [S1, unofficial].
                9.  If it intersects, transform the hit point back to world space using the forward matrix and splat center, then record the hit [S1, unofficial].
    3.  If internal node:
        1.  Recursively call `castRayAtSplatTreeNode` for each child [S1, unofficial].

5.  **Post-Processing**:
    1.  Transform hit origins and normals back to world space using `fromLocal` [S1, unofficial].
    2.  Compute distance as length of vector from ray origin to hit origin [S1, unofficial].
    3.  Sort all hits by distance ascending [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_splat_intersection(ray_origin, ray_direction, splat_tree, camera, use_true_ellipsoid=True, scale_epsilon=1e-7, render_mode="TwoD"):
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
            world_matrix = world_matrix @ subtree.scene_transform
        to_local = inverse(world_matrix)
        from_local = world_matrix
        
        # Transform ray to local space
        o_local = transform_point(to_local, ray_origin)
        d_local = normalize(transform_vector(to_local, ray_direction))
        local_ray = Ray(o_local, d_local)
        
        # Recursively cast
        cast_ray_at_node(local_ray, subtree.root_node, from_local, hits, scale_epsilon, render_mode, use_true_ellipsoid)
        
    # 3. Sort Hits
    hits.sort(key=lambda h: h.distance)
    return hits

def cast_ray_at_node(ray, node, from_local, hits, scale_epsilon, render_mode, use_true_ellipsoid):
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
            if render_mode == "ThreeD" and splat.scale.z <= scale_epsilon:
                continue
                
            hit = None
            if not use_true_ellipsoid:
                # Approximate Sphere
                radius = (splat.scale.x + splat.scale.y) / 2.0
                if render_mode == "ThreeD":
                    radius = (splat.scale.x + splat.scale.y + splat.scale.z) / 3.0
                hit = ray_sphere_intersection(ray, splat.center, radius)
            else:
                # True Ellipsoid
                # Compute opacity-based uniform scale
                opacity = splat.color.w
                uniform_scale_factor = log10(opacity) * 2.0
                
                # Construct matrices
                scale_mat = diag(splat.scale.x, splat.scale.y, splat.scale.z)
                rot_mat = quaternion_to_matrix(splat.rotation) # Standard quaternion-to-rotation matrix conversion [G]
                uniform_mat = diag(uniform_scale_factor, uniform_scale_factor, uniform_scale_factor)
                
                # World to Sphere transformation
                # M_world = U * R * S
                # M_sphere = M_world^-1
                m_world = uniform_mat @ rot_mat @ scale_mat
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
                    hit.normal = transform_vector(from_local, hit.normal)
            
            if hit:
                hits.append(hit)
    else:
        # Internal node: recurse
        for child in node.children:
            cast_ray_at_node(ray, child, from_local, hits, scale_epsilon, render_mode, use_true_ellipsoid)

def ray_sphere_intersection(ray_origin, sphere_center, radius):
    """
    Standard ray-sphere intersection.
    Returns hit record or None.
    """
    oc = ray_origin - sphere_center
    a = dot(ray.direction, ray.direction)
    b = 2.0 * dot(oc, ray.direction)
    c = dot(oc, oc) - radius * radius
    discriminant = b * b - 4.0 * a * c
    
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

# Helper definitions for self-containment
class Ray:
    def __init__(self, origin, direction):
        self.origin = origin
        self.direction = direction

class HitRecord:
    def __init__(self, distance, point, normal):
        self.distance = distance
        self.point = point
        self.normal = normal

def transform_point(matrix, point):
    # Standard 4x4 matrix-vector multiplication for homogeneous coordinates
    w = matrix[3][0]*point.x + matrix[3][1]*point.y + matrix[3][2]*point.z + matrix[3][3]
    return vec3(
        (matrix[0][0]*point.x + matrix[0][1]*point.y + matrix[0][2]*point.z + matrix[0][3]) / w,
        (matrix[1][0]*point.x + matrix[1][1]*point.y + matrix[1][2]*point.z + matrix[1][3]) / w,
        (matrix[2][0]*point.x + matrix[2][1]*point.y + matrix[2][2]*point.z + matrix[2][3]) / w
    )

def transform_vector(matrix, vector):
    # Standard 4x4 matrix-vector multiplication for direction (w=0)
    return vec3(
        matrix[0][0]*vector.x + matrix[0][1]*vector.y + matrix[0][2]*vector.z,
        matrix[1][0]*vector.x + matrix[1][1]*vector.y + matrix[1][2]*vector.z,
        matrix[2][0]*vector.x + matrix[2][1]*vector.y + matrix[2][2]*vector.z
    )

def ray_intersects_box(origin, direction, box):
    # Standard AABB-ray intersection test [G]
    # Returns True if ray intersects the axis-aligned bounding box
    pass

def diag(x, y, z):
    return [[x, 0, 0, 0], [0, y, 0, 0], [0, 0, z, 0], [0, 0, 0, 1]]

def log10(x):
    import math
    return math.log10(x)

def length(v):
    return (v.x**2 + v.y**2 + v.z**2)**0.5

def normalize(v):
    l = length(v)
    if l == 0: return vec3(0,0,0)
    return vec3(v.x/l, v.y/l, v.z/l)

def dot(v1, v2):
    return v1.x*v2.x + v1.y*v2.y + v1.z*v2.z

class vec3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z
    def __sub__(self, other): return vec3(self.x-other.x, self.y-other.y, self.z-other.z)
    def __add__(self, other): return vec3(self.x+other.x, self.y+other.y, self.z+other.z)
    def __mul__(self, scalar): return vec3(self.x*scalar, self.y*scalar, self.z*scalar)
```

### Edge Cases and Pitfalls
*   **Degenerate Splats**: Splats with scale components $\le \text{scaleEpsilon}$ ($10^{-7}$) are skipped to prevent division by zero or invalid geometry during matrix inversion [S1, unofficial].
*   **Null Data**: If `renderData` or `camera` is null, the intersection test returns null and logs an error [S2, unofficial].
*   **BVH Failure**: If BVH construction fails (`bvh === null`), the test returns null and logs an error [S2, unofficial].
*   **Vertex Count Changes**: If the number of vertices changes between calls, a warning is logged, but execution continues [S2, unofficial].
*   **Precision**: Using `log10` for opacity scaling assumes opacity is in $(0, 1]$. Opacity of $0$ results in $-\infty$, which should be handled or clamped [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Performance**: Not given in the sources.
*   **Strengths**: Supports both approximate and exact intersection; efficient culling via BVH; handles dynamic scenes via local space transformation [S1, unofficial].
*   **Limitations**: Not given in the sources.

## Background
Ray-Splat Intersection is a core component of real-time 3D Gaussian Splatting renderers, particularly those implemented in JavaScript/WebGL for browser-based viewing [S1, S2, unofficial]. It was developed to enable efficient picking and depth testing in large-scale Gaussian splat scenes [S1, S2, unofficial]. The method leverages BVH structures to accelerate ray-primitive intersection, a technique common in ray tracing but adapted here for the specific geometry of 3D Gaussians [S1, S2, unofficial].

## Sources
[S1] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
[S2] huggingface/gsplat.js, https://github.com/huggingface/gsplat.js, fetched 2026-10-27 (unofficial)
