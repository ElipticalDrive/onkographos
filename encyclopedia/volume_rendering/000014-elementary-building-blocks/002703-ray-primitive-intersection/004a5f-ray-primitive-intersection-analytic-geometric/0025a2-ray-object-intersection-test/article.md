# Ray-Object Intersection Test

Also called: Child Ray Intersection with Transformation, Ray-Object Intersection, raytrace

## Overview
The Ray-Object Intersection Test determines the closest intersection between a ray and a specific scene object, returning the intersection distance $t$ or a failure indicator if no intersection exists [S1, S3, unofficial]. It serves as the core evaluation step within a broader ray-scene traversal, where the test is executed for multiple objects to find the global minimum $t$ [S1, S2, unofficial]. What sets this method apart from sibling primitives like [Ray-box intersection](776-ray-box-intersection.md) or [Sphere Intersection](1620-sphere-intersection.md) is its role as the generic interface for arbitrary geometry; it typically handles coordinate space transformations and delegates the specific algebraic solution to the object's internal representation, rather than implementing the intersection algebra directly in the traversal loop [S4, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and a list of scene objects $\mathcal{O} = \{o_1, o_2, \dots, o_n\}$ [S2, S3, unofficial]. Each object $o_i$ may possess a transformation matrix $M_i$ mapping world space to local space [S4, unofficial].
- **Output**: The minimum positive intersection distance $t_{min}$ and the index $id$ of the closest object. If no intersection is found, $t_{min}$ is infinity and $id$ is undefined [S2, unofficial].

**Data Structures**
- **Ray**: Defined by origin $\mathbf{o} \in \mathbb{R}^3$ and direction $\mathbf{d} \in \mathbb{R}^3$ [G]. The direction is typically normalized [G].
- **Object (GeomObj)**: An abstract base class or interface representing a geometric primitive [S4, unofficial]. It must provide an intersection test function and store a transformation matrix $M$ [S4, unofficial].
- **Intersection State**: A structure storing the current closest distance `dist` (initialized to infinity) and the corresponding object pointer or index `obj` [S3, unofficial].

**Numbered Steps**
1. Initialize the intersection state: set `t_min` = $\infty$ and `id` = -1 [S2, unofficial].
2. Define a minimum acceptance threshold $\epsilon$ (typically $1.0 \times 10^{-5}$ [S3, unofficial] or $0.001$ [S4, unofficial]) to prevent self-intersection artifacts and floating-point precision errors [S3, S4, unofficial].
3. Iterate through each object $o_i$ in the scene list $\mathcal{O}$:
    a. **Transform Ray (if applicable)**: If the object uses a local coordinate system, transform the ray into that space [S4, unofficial].
        - Transform the origin: $\mathbf{o}' = M_i \cdot [\mathbf{o}, 1]^T$ (taking the xyz components) [S4, unofficial].
        - Transform the direction: $\mathbf{d}' = M_i \cdot [\mathbf{d}, 0]^T$ (taking the xyz components) [S4, unofficial].
        - Use $\mathbf{o}'$ and $\mathbf{d}'$ for the subsequent test [S4, unofficial]. If no transformation is needed, $\mathbf{o}' = \mathbf{o}$ and $\mathbf{d}' = \mathbf{d}$ [G].
    b. **Execute Primitive Test**: Call the object's internal intersection routine to solve for $t_{obj}$ [S1, unofficial]. This involves solving the system of equations defined by the ray $\mathbf{p}(t) = \mathbf{o}' + t\mathbf{d}'$ and the object's surface equation [S1, unofficial].
    c. **Validate Hit**: Check if $t_{obj} > \epsilon$ [S3, S4, unofficial].
    d. **Update Closest**: If $t_{obj} < t_{min}$, update $t_{min} = t_{obj}$ and `id` = $i$ [S2, S3, unofficial].
4. After iterating through all objects, return $t_{min}$ and `id` [S2, unofficial].

**Formulas**
- Ray Equation: $\mathbf{p}(t) = \mathbf{o} + t\mathbf{d}$, where $t \in \mathbb{R}^+$ [S1, unofficial].
- Unit Sphere Surface (Example Object): $\{(x, y, z) \mid x^2 + y^2 + z^2 = 1\}$ [S1, unofficial].
- Optimization Criterion: Find object $o$ that minimizes $t$ such that $t = \text{intersection test}(\mathbf{r}, o)$ and $t > 0$ [S1, unofficial].

**Parameters**
- $\epsilon$: Minimum distance threshold to accept a hit. Typical values are $1.0 \times 10^{-5}$ [S3, unofficial] or $0.001$ [S4, unofficial]. Not given in the sources.

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
        ray_origin: Vec3, origin of the ray.
        ray_direction: Vec3, direction of the ray (assumed normalized).
        objects: List of GeomObj, each having a transform matrix and intersect method.
        epsilon: float, minimum distance to accept a hit.
        
    Returns:
        tuple: (t_min, object_index) or (inf, -1) if no intersection.
    """
    t_min = float('inf')
    id = -1
    
    for i, obj in enumerate(objects):
        # Step 3a: Transform ray to object's local space if needed
        # Transform origin (w=1) and direction (w=0) per evidence [S4, unofficial]
        # Assumes obj.matrix is a 4x4 matrix and Vec4 constructs a 4D vector
        local_origin = obj.matrix * Vec4(ray_origin.x, ray_origin.y, ray_origin.z, 1.0)
        local_direction = obj.matrix * Vec4(ray_direction.x, ray_direction.y, ray_direction.z, 0.0)
        
        # Extract xyz components for the 3D primitive test
        o_prime = Vec3(local_origin.x, local_origin.y, local_origin.z)
        d_prime = Vec3(local_direction.x, local_direction.y, local_direction.z)
        
        # Step 3b: Execute primitive test
        # The object's intersect method solves the system of equations
        # Returns the distance t or -1 if no intersection
        t_obj = obj.intersect(o_prime, d_prime)
        
        # Step 3c & 3d: Validate hit and update closest
        if t_obj > epsilon and t_obj < t_min:
            t_min = t_obj
            id = i
            
    return t_min, id
```

## Subtopics

### Bounding Box Node Intersection
A recursive ray traversal algorithm for a binary bounding box tree that determines the closest surface intersection by testing the node's bounding box and recursing into children. It differs from the flat list iteration by pruning empty space and is chosen when scene geometry is organized into a hierarchical structure for acceleration. [Bounding Box Node Intersection](8208-bounding-box-node-intersection.md)

## Operating characteristics
Not given in the sources.

## Background
The method is documented in academic theses and open-source ray tracers, such as those found in [S1, S3, S4, unofficial]. The mathematical formulation of finding the minimum $t$ for a ray and object is standard in ray tracing literature [S1, unofficial]. The specific implementation details involving transformation matrices and epsilon thresholds are documented in various educational and open-source ray tracers [S1, S3, S4, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S3] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-27 (unofficial)
[S4] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
