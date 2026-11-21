# Ray-Object Intersection Test

Also called: Child Ray Intersection with Transformation, Ray Intersection Update, Ray-Object Intersection, raytrace

## Overview
The Ray-Object Intersection Test determines if and where a ray intersects a specific scene object, returning the closest valid intersection distance or a failure indicator [S1, unofficial]. It serves as the core decision unit in ray tracing and volume rendering pipelines, filtering out non-intersections and self-intersections via epsilon thresholds to ensure robustness [S2, unofficial][S4, unofficial]. Unlike sibling primitives that solve for specific analytic shapes (e.g., [Sphere Intersection](1620-sphere-intersection.md)), this test is a generic interface that may involve coordinate space transformations to handle arbitrary object orientations and positions [S5, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and an object (geometry definition, optional transformation matrix $\mathbf{M}$) [G].
*   **Output:** Distance $t$ to the closest intersection point, hit status, object index, and surface normal [S1, unofficial][S2, unofficial][S3, unofficial].

**Data Structures**
The ray state is maintained in a structure containing:
*   `origin`: Vector3D $(x, y, z)$
*   `direction`: Vector3D $(x, y, z)$, typically normalized [G].
*   `normal`: Vector3D, the surface normal at the hit point [S2, unofficial].
*   `t`: Float, the current closest hit distance [S2, unofficial].
*   `hit`: Boolean, indicating if a valid hit has been recorded [S2, unofficial].

**Steps**
1.  **Transformation (Optional):** If the object has a transformation matrix $\mathbf{M}$, transform the ray into the object's local coordinate space [S5, unofficial].
    *   New origin: $\mathbf{o}' = \mathbf{M} \cdot [\mathbf{o}, 1]^T$
    *   New direction: $\mathbf{d}' = \mathbf{M} \cdot [\mathbf{d}, 0]^T$
2.  **Intersection Solve:** Solve the system of equations representing the ray and the object's surface geometry to find potential intersection parameters $t_{obj}$ [S1, unofficial].
3.  **Filtering:** Check if the computed $t_{obj}$ is valid:
    *   $t_{obj} > \epsilon$ (where $\epsilon$ is $1.0 \times 10^{-5}$ [S2, unofficial][S4, unofficial] or $0.001$ [S5, unofficial]) to prevent self-intersection or near-origin artifacts [S4, unofficial].
    *   $t_{obj} < t_{current}$ (if tracking the closest hit among multiple objects) [S3, unofficial].
4.  **Update:** If valid, update the ray's hit state:
    *   Set `hit` = true.
    *   Set `t` = $t_{obj}$.
    *   Store the object index [S3, unofficial].
    *   Transform the normal back to world space using the matrix transpose: $\mathbf{n}_{world} = \text{normalize}(\mathbf{M}^\top \cdot [\mathbf{n}_{local}, 1]^T)$ [S5, unofficial].
5.  **Return:** Return the boolean hit status and the distance $t$ [S1, unofficial][S2, unofficial].

**Formulas**
*   Ray Equation: $\mathbf{p}(t) = \mathbf{o} + t\mathbf{d}, \quad t \in \mathbb{R}^+$ [S1, unofficial].
*   Unit Sphere Surface: $\{(x, y, z) \mid x^2 + y^2 + z^2 = 1\}$ [S1, unofficial].
*   Optimization Criterion: Find object $o$ that minimizes $t$ such that $t = \text{intersection test}(\mathbf{r}, o)$ and $t > 0$ [S1, unofficial].

**Parameters**
*   $\epsilon$ (SMALLEST_DIST): Minimum distance threshold to accept a hit. Typical values are $1.0 \times 10^{-5}$ [S2, unofficial][S4, unofficial] or $0.001$ [S5, unofficial].

**Edge Cases and Pitfalls**
*   **Self-Intersection:** Intersections very close to the ray origin or previous hit point ($t \le \epsilon$) must be discarded to avoid artifacts [S4, unofficial].
*   **Degenerate Rays:** If $\mathbf{d}$ is not normalized, $t$ does not represent Euclidean distance [G].
*   **Transformation Precision:** Normal transformation uses the matrix transpose, not the inverse-transpose, to map local normals to world space per the provided implementation [S5, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_object_intersection(ray, objects, epsilon=1e-5):
    """
    Finds the closest intersection between a ray and a list of objects.
    """
    closest_t = float('inf')
    hit_index = -1
    
    for index, obj in enumerate(objects):
        # 1. Transform ray to object's local space if matrix M exists
        if hasattr(obj, 'transform'):
            M = obj.transform
            local_o = M.transform_point(ray.origin)
            local_d = M.transform_vector(ray.direction)
        else:
            local_o = ray.origin
            local_d = ray.direction
            
        # 2. Solve system of equations between the ray and the object geometry
        # Returns distance t_obj or None if no intersection
        t_obj = solve_intersection(local_o, local_d, obj.geometry)
        
        # 3. Filter: must be in front of ray origin and closer than current best
        if t_obj is not None and t_obj > epsilon and t_obj < closest_t:
            closest_t = t_obj
            hit_index = index
            
    # 4. Update ray state if a valid hit was found
    if hit_index != -1:
        ray.t = closest_t
        ray.hit = True
        ray.obj_index = hit_index
        
        # Transform normal back to world space using matrix transpose
        if hasattr(objects[hit_index], 'transform'):
            M = objects[hit_index].transform
            local_n = compute_normal(objects[hit_index].geometry, closest_t)
            world_n = M.transpose().transform_vector(local_n)
            ray.normal = normalize(world_n)
        else:
            local_n = compute_normal(objects[hit_index].geometry, closest_t)
            ray.normal = normalize(local_n)
            
    return ray.hit, ray.t

def solve_intersection(o, d, geometry):
    # Solves the parametric ray equation p(t) = o + t*d against the object's implicit/surface equation.
    # The sources do not give the specific solver implementation for arbitrary geometry.
    return None

def compute_normal(geometry, t):
    # Computes the surface normal at the intersection point.
    # The sources do not give the specific normal computation implementation.
    return Vector3(0, 0, 1)
```

## Subtopics

### [Bounding Box Node Intersection](3958-bounding-box-node-intersection.md)
A recursive ray traversal algorithm for a binary bounding box tree (BBNode) that determines the closest surface intersection by testing the node's bounding box, then recursing into left and right children, and finally comparing hit distances. It solves the problem of efficiently finding the nearest primitive in a hierarchical scene structure by pruning branches whose bounding volumes do not intersect the ray. Unlike [Ray-Object Intersection Test](3604-ray-object-intersection-test.md) which tests a single object, this method navigates a tree structure to find the global minimum intersection distance among all contained primitives.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per object for analytic primitives (spheres, planes, triangles) [S1, unofficial]. $O(N)$ for a naive test against $N$ objects [S3, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space per test [S2, unofficial].
*   **Strengths:** Core decision unit for ray tracing and volume rendering; handles coordinate transformations; robust with epsilon filtering [S1, unofficial][S2, unofficial][S5, unofficial].
*   **Limitations:** Linear scan does not scale to large scenes [S3, unofficial].
*   **When to use:** For small scenes, debugging, or as the leaf node test in acceleration structures [G].
*   **When not to use:** For large-scale rendering without acceleration; for complex CSG operations without decomposition [G].

## Background
The concept originates from early ray tracing algorithms, where the intersection of a ray with primitive shapes (spheres, planes) was solved analytically [S1, unofficial]. The term "raytrace" in this context refers to the iterative process of casting rays and updating intersection states [S4, unofficial]. The use of epsilon thresholds to prevent self-intersection is a standard practice in computer graphics to handle floating-point precision errors [S4, unofficial][S5, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
[S3] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S4] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-27 (unofficial)
[S5] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
