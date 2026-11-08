# Ray-box intersection
Also called: Slab Test, Slab Method, Smits' method, AABB Slab Method, Slab intersection, AABB Ray Intersection, Ray-AABB Intersection, Ray-BBox Intersection, Kay and Kajiya slab test, Smits' Ray-Box Intersection, Ray-Bounding Box Intersection, Ray-Axis-Aligned Bounding Box Intersection

## Overview
The ray-box intersection method determines whether a ray intersects an axis-aligned bounding box (AABB) by treating the box as the intersection of three pairs of parallel planes, known as slabs [S1, S3, unofficial]. It computes the entry ($t_{near}$) and exit ($t_{far}$) distances for each slab pair and checks for overlap across all three axes [S1, S3, unofficial]. Unlike general primitive intersection methods, this approach exploits the axis-aligned nature of the box to reduce the problem to simple scalar comparisons and vector operations, making it the dominant technique for bounding volume hierarchy traversal in ray tracing [S2, S3, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   **Ray:** Defined by origin $\mathbf{o} = (o_x, o_y, o_z)$ and direction $\mathbf{v} = (v_x, v_y, v_z)$, parameterized as $\mathbf{p}(t) = \mathbf{o} + t\mathbf{v}$ [S1, S3, unofficial].
*   **AABB:** Defined by minimum corner $\mathbf{p}_0 = (x_0, y_0, z_0)$ and maximum corner $\mathbf{p}_1 = (x_1, y_1, z_1)$ [S1, S2, S3, unofficial].
*   **Ray Interval (Optional):** A valid hit distance interval $[t_{min}, t_{max}]$ representing the range of $t$-values on the ray that count as a hit [S3, S4, unofficial].

**Outputs:**
*   Boolean indicating intersection.
*   If intersecting, the entry distance $t_{near}$ and exit distance $t_{far}$ [S1, S4, unofficial].

### Data Structures
The method operates on scalar values or 3D vectors [S2, S3, unofficial].
*   **Inverse Ray Direction:** $\mathbf{invV} = (1/v_x, 1/v_y, 1/v_z)$ [S3, S4, unofficial].
*   **Interval Vectors:** Vectors storing the computed $t$-values for the lower and upper bounds of each axis [S3, S4, unofficial].

### Algorithm Steps
The method can be implemented in two primary styles: the iterative scalar approach (Kay and Kajiya original) and the compact vector approach (widely used in modern kernels) [S2, S3, unofficial].

#### 1. Iterative Scalar Approach
This approach processes each axis (X, Y, Z) sequentially, updating global $t_{near}$ and $t_{far}$ values [S1, unofficial].

1. Initialize $t_{near} = -\infty$ and $t_{far} = \infty$ [S1, unofficial].
2. For each axis $i \in \{x, y, z\}$:
    a. Let $o_i$ be the ray origin component, $v_i$ be the ray direction component, $min_i$ be the box minimum coordinate, and $max_i$ be the box maximum coordinate [S1, S3, unofficial].
    b. Check for parallelism: If $v_i = 0$:
        i. If $o_i < min_i$ or $o_i > max_i$, the ray is outside the slab. Return `false` (miss) [S1, unofficial].
        ii. Otherwise, the ray is inside the slab along this axis; continue to the next axis [S1, unofficial].
    c. Compute intersection distances with the near and far planes of the slab:
        $$t_1 = \frac{min_i - o_i}{v_i}$$
        $$t_2 = \frac{max_i - o_i}{v_i}$$
    d. Ensure $t_1$ is the smaller value (entry) and $t_2$ is the larger value (exit). If $t_1 > t_2$, swap $t_1$ and $t_2$ [S1, unofficial].
    e. Update the global intersection interval:
        $$t_{near} = \max(t_{near}, t_1)$$
        $$t_{far} = \min(t_{far}, t_2)$$
    f. Check for miss: If $t_{near} > t_{far}$, the intervals do not overlap. Return `false` [S1, unofficial].
    g. Check if the box is behind the ray: If $t_{far} < 0$, the entire box is behind the ray origin. Return `false` [S1, unofficial].
3. If the loop completes without returning `false`, the ray intersects the box. Return `true` [S1, unofficial].

#### 2. Compact Vector Approach
This approach computes all intersections in parallel using vector operations, often branchless [S2, S3, unofficial].

1. Compute the inverse ray direction: $\mathbf{invV} = (1/v_x, 1/v_y, 1/v_z)$ [S3, S4, unofficial].
2. Compute the $t$-values for the lower and upper bounds of the box:
    $$\mathbf{t}_{lower} = (\mathbf{p}_0 - \mathbf{o}) \odot \mathbf{invV}$$
    $$\mathbf{t}_{upper} = (\mathbf{p}_1 - \mathbf{o}) \odot \mathbf{invV}$$
    where $\odot$ denotes component-wise multiplication [S3, S4, unofficial].
3. Determine the entry and exit $t$-values for each slab by taking the component-wise minimum and maximum of $\mathbf{t}_{lower}$ and $\mathbf{t}_{upper}$:
    $$\mathbf{t}_{min\_slabs} = \min(\mathbf{t}_{lower}, \mathbf{t}_{upper})$$
    $$\mathbf{t}_{max\_slabs} = \max(\mathbf{t}_{lower}, \mathbf{t}_{upper})$$
4. Incorporate the ray's valid hit interval $[t_{min}, t_{max}]$ as a fourth dimension (or separate scalar check) [S3, S4, unofficial].
    $$t_{box\_min} = \max(\mathbf{t}_{min\_slabs}.x, \mathbf{t}_{min\_slabs}.y, \mathbf{t}_{min\_slabs}.z, t_{min})$$
    $$t_{box\_max} = \min(\mathbf{t}_{max\_slabs}.x, \mathbf{t}_{max\_slabs}.y, \mathbf{t}_{max\_slabs}.z, t_{max})$$
5. The ray intersects the box if $t_{box\_min} \le t_{box\_max}$ [S3, S4, unofficial].

### Pseudocode (Compact Vector Style)
The following Python-like pseudocode implements the robust compact vector approach, handling zero direction components and negative zeros correctly per IEEE 754 standards [S6, unofficial][S14][S16, unofficial].

```python
def slabs_box_test(p0, p1, ray_origin, ray_dir, ray_t_min, ray_t_max):
    """
    Determines if a ray intersects an AABB.
    Handles zero direction components and negative zeros robustly.
    """
    t_near = float('-inf')
    t_far = float('inf')
    
    for i in range(3):
        if ray_dir[i] == 0.0:
            if ray_origin[i] < p0[i] or ray_origin[i] > p1[i]:
                return False, None, None
            continue
        
        inv_d = 1.0 / ray_dir[i]
        t1 = (p0[i] - ray_origin[i]) * inv_d
        t2 = (p1[i] - ray_origin[i]) * inv_d
        
        if inv_d >= 0.0:
            t_near = max(t_near, t1)
            t_far = min(t_far, t2)
        else:
            t_near = max(t_near, t2)
            t_far = min(t_far, t1)
            
        if t_near > t_far:
            return False, None, None
        if t_far < 0.0:
            return False, None, None
            
    return True, max(t_near, ray_t_min), min(t_far, ray_t_max)
```

### Parameters and Typical Values
*   **Ray Direction:** Must be normalized in many applications, but the algorithm works with unnormalized directions as long as `inv_ray_dir` is computed correctly [S3, S4, unofficial].
*   **Box Corners:** $\mathbf{p}_0$ and $\mathbf{p}_1$ must satisfy $p_{0,i} \le p_{1,i}$ for all $i \in \{x,y,z\}$ [S1, S2, S3, unofficial].
*   **Ray Interval:** The interval $[t_{min}, t_{max}]$ represents the valid hit distances on the ray [S3, S4, unofficial]. Typical values are not given in the sources [G].

### Edge Cases and Pitfalls
*   **Zero Direction Components:** The algorithm produces correct results for rays with zero direction components (parallel to axes) provided the hardware is IEEE floating-point compliant [S3, S4, unofficial]. The division by zero in `inv_ray_dir` results in infinity, which is handled correctly by the min/max logic [S3, S4, unofficial]. Explicit checks for $v_i = 0$ prevent `NaN` generation when the ray origin lies exactly on a box boundary [S6, unofficial][S14].
*   **Negative Zero Direction:** If a direction component is `-0.0`, testing the sign of the direction itself fails due to IEEE 754 rules. The robust implementation tests the sign of the reciprocal direction component instead [S14][S16, unofficial].
*   **Ray Origin on Box Face:** If the ray origin lies exactly on the box surface and the direction points along the face, naive implementations may result in `NaN` (Not a Number), causing the test to return `false` [S3, S4, unofficial]. This behavior can be modified if specific handling is required [S3, S4, unofficial].
*   **Branchless Optimization:** Removing early exit checks (like `if t_near > t_far`) creates a branchless implementation. While this performs more arithmetic operations, it significantly increases performance on modern hardware by avoiding branch mispredictions [S3, unofficial].

## Subtopics

### Annulus Box Bounds Conversion
Converts an inner bounding box to a list of bounds relative to an enclosing box, adding entries for sides where the inner box is strictly contained [S3, unofficial]. It initializes a counter, loops over dimensions to count boundaries, allocates an array of `ANNorthHalfSpace` structs, and populates them with dimension index, cutting value, and sign (+1 or -1) based on whether the inner box's lower or upper bound is strictly inside the enclosing box [S3, unofficial].

### Ray-Box Intersection via OpenGL Geometry Shaders
Computes entry and exit points of rays intersecting a bounding box by rendering the box geometry to textures using OpenGL, storing hit positions in 16-bit floating point textures [S17, unofficial]. It creates a Framebuffer Object (FBO) with `GL_RGBA16F` color attachments and a `GL_DEPTH_COMPONENT16` depth renderbuffer, enables face culling and depth testing, and draws the cube geometry using a hit shader program passing the MVP matrix as a uniform [S17, unofficial]. The depth buffer must be cleared along with the color buffer, and correct depth writing from the geometry shader is required [S17, unofficial].

## Operating characteristics
*   **Time Complexity:** The algorithm performs a fixed number of arithmetic operations and comparisons regardless of scene complexity [S3, S4, unofficial].
*   **Performance:** The slabs test is the dominant method for AABB intersection due to its simplicity and efficiency [S3, S4, unofficial]. Branchless implementations are preferred in high-performance ray tracing kernels (e.g., Embree) to maximize throughput on SIMD units and GPUs [S3, unofficial].
*   **Strengths:** Fast, robust, easy to implement, works well with SIMD/vectorization [S3, S4, unofficial].
*   **Limitations:** Only applicable to axis-aligned boxes [S3, S4, unofficial].

## Background
*   **Origin:** The method was introduced by T. L. Kay and J. T. Kajiya in their 1986 paper "Ray tracing complex scenes" presented at SIGGRAPH [S2, S3, unofficial].
*   **Development:** The algorithm has become the standard for AABB intersection in ray tracing. It is widely used in modern ray tracing kernels such as Embree and in astronomical simulation tools like raytheia [S3, S4, unofficial].
*   **Usage:** It is used for primary ray-scene intersection testing, shadow ray occlusion testing, frustum culling, and spatial traversal of octree and grid acceleration structures [S5, unofficial].

## Sources
[S1] Ray‐Box Intersection Using the Slab Method | Every Algorithm, https://every-algorithm.github.io/2024/11/09/slab_method.html, fetched 2026-10-29 (unofficial)
[S2] Ray - Box Intersection - ACM SIGGRAPH, https://education.siggraph.org/static/HyperGraph/raytrace/rtinter3.htm, fetched 2026-10-29 (unofficial)
[S3] (PDF) Ray Axis-Aligned Bounding Box Intersection - ResearchGate, https://www.researchgate.net/publication/354064987_Ray_Axis-Aligned_Bounding_Box_Intersection, fetched 2026-10-30 (unofficial)
[S4] RAY AXIS-ALIGNED BOUNDING BOX INTERSECTION, https://link.springer.com/content/pdf/10.1007/978-1-4842-7185-8_2.pdf, fetched 2026-10-29 (unofficial)
[S5] intersect: AABB Ray Intersection Algorithm - glre.dev, https://glre.dev/addons/geometry/aabb/intersect/, fetched 2026-10-29 (unofficial)
[S6] Fast, Branchless Ray/Bounding Box Intersections - tavianator.com, https://tavianator.com/2011/ray_box.html, fetched 2026-10-29 (unofficial)
[S14] PDF An Efficient and Robust Ray-Box Intersection Algorithm, https://people.csail.mit.edu/amy/papers/box-jgt.pdf, fetched 2026-10-29
[S16] (PDF) An Efficient and Robust Ray-Box Intersection Algorithm, https://www.researchgate.net/publication/220494140_An_Efficient_and_Robust_Ray-Box_Intersection_Algorithm, fetched 2026-11-01 (unofficial)
[S17] franjaviersans/VRCUDA, https://github.com/franjaviersans/VRCUDA, fetched 2026-10-27 (unofficial)
