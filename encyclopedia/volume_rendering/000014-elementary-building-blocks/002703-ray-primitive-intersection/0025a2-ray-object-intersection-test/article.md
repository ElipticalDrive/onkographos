# Ray-Object Intersection Test

Also called: Child Ray Intersection with Transformation, Ray-Object Intersection, raytrace

## Overview
The Ray-Object Intersection Test is the fundamental geometric primitive used to determine the closest intersection between a ray and a specific scene object, returning the intersection distance $t$ or a failure indicator if no intersection exists [S1, S3, unofficial]. It serves as the core evaluation step within a broader ray-scene traversal, where the test is executed for multiple objects to find the global minimum $t$ [S1, S2, unofficial]. What sets this method apart from sibling primitives like [Ray-box intersection](776-ray-box-intersection.md) or [Sphere Intersection](1620-sphere-intersection.md) is its role as the generic interface for arbitrary geometry; it typically handles coordinate space transformations and delegates the specific algebraic solution to the object's internal representation, rather than implementing the intersection algebra directly in the traversal loop [S4, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and a list of scene objects $\mathcal{O} = \{o_1, o_2, \dots, o_n\}$. Each object $o_i$ may possess a transformation matrix $M_i$ mapping world space to local space [S4, unofficial].
- **Output**: The minimum positive intersection distance $t_{min}$ and the index $id$ of the closest object. If no intersection is found, $t_{min}$ is infinity and $id$ is undefined [S2, unofficial].

**Data Structures**
- **Ray**: Defined by origin $\mathbf{o} \in \mathbb{R}^3$ and direction $\mathbf{d} \in \mathbb{R}^3$. The direction is typically normalized [Parent].
- **Object (GeomObj)**: An abstract base class or interface representing a geometric primitive. It must provide an intersection test function. In implementations using transformation matrices, it stores a matrix $M$ [S4, unofficial].
- **Intersection State**: A structure storing the current closest distance `dist` (initialized to infinity) and the corresponding object pointer or index `obj` [S3, unofficial].

**Numbered Steps**
1. Initialize the intersection state: set `t_min` = $\infty$ and `id` = -1 [S2, unofficial].
2. Define a minimum acceptance threshold $\epsilon$ (typically $1.0 \times 10^{-5}$ or $0.001$) to prevent self-intersection artifacts and floating-point precision errors [S3, S4, unofficial].
3. Iterate through each object $o_i$ in the scene list $\mathcal{O}$:
    a. **Transform Ray (if applicable)**: If the object uses a local coordinate system, transform the ray into that space [S4, unofficial].
        - Transform the origin: $\mathbf{o}' = M_i \cdot [\mathbf{o}, 1]^T$ (taking the xyz components).
        - Transform the direction: $\mathbf{d}' = M_i \cdot [\mathbf{d}, 0]^T$ (taking the xyz components). Note that the direction is treated as a vector (w=0) to ignore translation [S4, unofficial].
        - Use $\mathbf{o}'$ and $\mathbf{d}'$ for the subsequent test. If no transformation is needed, $\mathbf{o}' = \mathbf{o}$ and $\mathbf{d}' = \mathbf{d}$.
    b. **Execute Primitive Test**: Call the object's internal intersection routine to solve for $t_{obj}$ [S1, unofficial]. This involves solving the system of equations defined by the ray $\mathbf{p}(t) = \mathbf{o}' + t\mathbf{d}'$ and the object's surface equation [S1, unofficial].
    c. **Validate Hit**: Check if $t_{obj} > \epsilon$ [S3, S4, unofficial].
    d. **Update Closest**: If $t_{obj} < t_{min}$, update $t_{min} = t_{obj}$ and `id` = $i$ [S2, S3, unofficial].
4. After iterating through all objects, return $t_{min}$ and `id` [S2, unofficial].

**Formulas**
- Ray Equation: $\mathbf{p}(t) = \mathbf{o} + t\mathbf{d}$, where $t \in \mathbb{R}^+$ [S1, unofficial].
- Unit Sphere Surface (Example Object): $\{(x, y, z) \mid x^2 + y^2 + z^2 = 1\}$ [S1, unofficial].
- Optimization Criterion: Find object $o$ that minimizes $t$ such that $t = \text{intersection test}(\mathbf{r}, o)$ and $t > 0$ [S1, unofficial].

**Parameters**
- $\epsilon$: Minimum distance threshold to accept a hit. Typical values are $1.0 \times 10^{-5}$ [S3, unofficial] or $0.001$ [S4, unofficial]. The specific value depends on the scale of the scene and floating-point precision requirements.

**Edge Cases and Pitfalls**
- **Self-Intersection**: Intersections with $t < \epsilon$ are ignored to prevent the ray from intersecting the same object it just exited or originated from [S3, S4, unofficial].
- **Transformation Precision**: When transforming the ray direction, ensure the w-component is 0 to avoid translation affecting the direction vector [S4, unofficial].
- **No Intersection**: If the loop completes without finding a valid $t_{obj}$, the result is $\infty$ [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_object_intersection_test(ray_origin, ray_direction, objects, epsilon=1e-5):
    """
    Finds the closest intersection between a ray and a list of objects.
    
    Args:
        ray_origin: Vector3, origin of the ray.
        ray_direction: Vector3, direction of the ray (assumed normalized).
        objects: List of GeomObj, each having a transform matrix and intersect method.
        epsilon: float, minimum distance to accept a hit.
        
    Returns:
        tuple: (t_min, object_index) or (inf, -1) if no intersection.
    """
    t_min = float('inf')
    id = -1
    
    for i, obj in enumerate(objects):
        # Step 3a: Transform ray to object's local space if needed
        # Assuming obj has a 'matrix' attribute and 'intersect' method
        # If the object is in world space, matrix is identity
        
        # Transform origin (w=1)
        local_origin = obj.matrix.transform_point(ray_origin)
        
        # Transform direction (w=0)
        local_direction = obj.matrix.transform_vector(ray_direction)
        
        # Step 3b: Execute primitive test
        # The object's intersect method solves the system of equations
        # Returns the distance t or -1 if no intersection
        t_obj = obj.intersect(local_origin, local_direction)
        
        # Step 3c: Validate hit
        if t_obj > epsilon:
            # Step 3d: Update closest
            if t_obj < t_min:
                t_min = t_obj
                id = i
                
    return t_min, id
```

## Subtopics

### Bounding Box Node Intersection
A recursive ray traversal algorithm for a binary bounding box tree (BBNode) that determines the closest surface intersection by testing the node's bounding box, then recursing into left and right children, and finally comparing hit distances [S4, unofficial]. It is chosen when scene geometry is organized into a hierarchical structure for acceleration. It differs from the flat list iteration by pruning empty space.
- **Definition**: Recursively tests a BBNode. If the left child exists, it calls `intersect` on it (casting to BBNode if it is a node, or `child_intersect` if it is a leaf). Same for the right child. The hit distance from the left child is compared to the right child, with floating-point precision handled by adding epsilon to the left hit distance before comparison [S4, unofficial].
- **Layout**: `BBNode` contains `min` (Vec3), `max` (Vec3), `child_l` (SceneObject*), `child_r` (SceneObject*), and `getType()` returning NODE or leaf type [S4, unofficial].
- **Parameter**: $\epsilon = 0.001$ is used to handle floating-point precision issues in hit distance comparison [S4, unofficial].
- **Pitfall**: Floating-point precision in hit distance comparison requires adding epsilon to the left hit distance before comparing to ensure consistent ordering [S4, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ for a flat list of $N$ objects, as each object must be tested [S2, unofficial]. With acceleration structures like the Bounding Box Node, complexity becomes $O(\log N)$ on average for sparse scenes, but worst-case remains $O(N)$ [S4, unofficial].
- **Memory Complexity**: $O(1)$ auxiliary space for the flat list approach, storing only the current minimum distance and index [S2, unofficial]. The BBNode approach requires $O(N)$ space to store the tree structure [S4, unofficial].
- **Strengths**: Simple to implement for flat lists; generic interface allows any geometry type to be tested via transformation [S4, unofficial].
- **Limitations**: Linear scan is inefficient for large scenes without acceleration structures; transformation overhead per object can be significant [S4, unofficial].
- **When to use**: Small scenes, debugging, or as the leaf node test in an acceleration structure [S4, unofficial].
- **When not to use**: Large scenes with many objects without an acceleration structure, where $O(N)$ per ray is prohibitive.

## Background
The Ray-Object Intersection Test is a foundational concept in ray tracing, originating from the early days of computer graphics. The mathematical formulation of finding the minimum $t$ for a ray and object is standard in ray tracing literature [S1, unofficial]. The specific implementation details involving transformation matrices and epsilon thresholds are documented in various educational and open-source ray tracers, such as those found in academic theses and GitHub repositories [S1, S3, S4, unofficial]. The use of bounding box trees for recursive intersection testing represents an optimization over the flat list approach, introduced to handle complex scenes efficiently [S4, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S3] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-27 (unofficial)
[S4] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
