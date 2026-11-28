# Ray-Square Intersection

Also called: Quad Plane Intersection, Square Ray Intersection, Square::rayIntersect, castRayAgainstQuads

## Overview
Ray-square intersection computes whether a ray intersects a finite square primitive (quad) defined by a corner point and two edge vectors, or by a transformation matrix and a canonical unit square. It solves the problem of determining precise hit locations on planar square surfaces, which is essential for rendering flat geometry, light sources, and texture planes in volume rendering and path tracing. It differs from [Ray-Plane/Rectangle Intersection](6498-ray-plane-rectangle-intersection.md) by enforcing strict square boundary checks (equal edge lengths, orthogonal or specific basis constraints) and often optimizing for batched or canonical representations.

## How it works
### Inputs and Outputs
**Inputs:**
1.  Ray: Origin $\mathbf{o}$, Direction $\mathbf{d}$ (normalized).
2.  Square Definition: One of the following:
    *   **Explicit Basis:** Corner $\mathbf{c}$, Edge Vectors $\mathbf{v}_1, \mathbf{v}_2$, and Normal $\mathbf{n}$.
    *   **Transform Matrix:** World-to-local transformation matrix $\mathbf{M}_{world \to local}$ and a canonical square in local space spanning $[-0.5, 0.5] \times [-0.5, 0.5]$ at $z=0$.
3.  (Optional) Batch: Array of transformation matrices for multiple squares.

**Outputs:**
*   Boolean `hit`: True if intersection occurs within the square bounds.
*   Float `t`: Distance along the ray to the intersection point.
*   Vector2 `uv`: Local coordinates on the square (typically $[0,1]$ or $[-0.5, 0.5]$).
*   Vector3 `point`: World-space intersection point.

### Data Structures
*   **Ray:** Struct with `origin` (vec3) and `direction` (vec3).
*   **Square (Explicit):** Struct with `corner` (vec3), `v1` (vec3), `v2` (vec3), `normal` (vec3).
*   **Square (Transform):** Struct with `worldToLocal` (mat4x4).
*   **Batch:** Array of `worldToLocal` matrices.

### Steps

#### Method A: Explicit Basis (General Quad)
This method computes the intersection with the infinite plane defined by the square, then checks if the point lies within the square boundaries using barycentric-like coordinates.

1.  **Compute Plane Normal:**
    $$ \mathbf{w} = \mathbf{v}_1 \times \mathbf{v}_2 $$
    If $\|\mathbf{w}\| \approx 0$, the square is degenerate; return no hit.
    $$ \mathbf{n} = \frac{\mathbf{w}}{\|\mathbf{w}\|} $$

2.  **Ray-Plane Intersection:**
    Compute the denominator:
    $$ denom = \mathbf{n} \cdot \mathbf{d} $$
    If $|denom| < \epsilon$ (ray parallel to plane), return no hit.
    Compute distance $t$:
    $$ t = \frac{\mathbf{n} \cdot (\mathbf{c} - \mathbf{o})}{denom} $$
    If $t < \epsilon$ (intersection behind ray origin), return no hit.

3.  **Intersection Point:**
    $$ \mathbf{p} = \mathbf{o} + t \mathbf{d} $$

4.  **Boundary Check:**
    Compute vector from corner to point:
    $$ \mathbf{p}_{local} = \mathbf{p} - \mathbf{c} $$
    Compute coordinates $\alpha, \beta$ using the cross-product method described in [S1, unofficial]:
    $$ \alpha = \frac{\mathbf{w} \cdot (\mathbf{p}_{local} \times \mathbf{v}_2)}{\mathbf{w} \cdot \mathbf{w}} $$
    $$ \beta = \frac{\mathbf{w} \cdot (\mathbf{v}_1 \times \mathbf{p}_{local})}{\mathbf{w} \cdot \mathbf{w}} $$
    The point is inside the square if:
    $$ 0 < \alpha < 1 \quad \text{and} \quad 0 < \beta < 1 $$
    (Note: Strict inequality excludes edges; use $\le$ if edges are included. [S1, unofficial] uses strict inequalities).

5.  **UV Coordinates:**
    $$ u = \alpha, \quad v = \beta $$

#### Method B: Transform Matrix (Canonical Square)
This method transforms the ray into the local space of the square, where the square is a unit square centered at the origin in the $xy$-plane ($z=0$), spanning $[-0.5, 0.5]$. This is efficient for batched rendering [S2, unofficial].

1.  **Transform Ray:**
    Transform ray origin and direction to local space using $\mathbf{M}_{world \to local}$:
    $$ \mathbf{o}_{local} = \mathbf{M}_{world \to local} \cdot [\mathbf{o}, 1]^T $$
    $$ \mathbf{d}_{local} = \text{normalize}(\mathbf{M}_{world \to local} \cdot [\mathbf{d}, 0]^T) $$
    Note: The $w$ component of the origin becomes the local $z$ coordinate. The direction is transformed as a vector (ignoring translation) and normalized [S2, unofficial].

2.  **Ray-Plane Intersection (Local):**
    The plane is at $z=0$.
    $$ t = \frac{-\mathbf{o}_{local}.z}{\mathbf{d}_{local}.z} $$
    Check validity:
    *   $\mathbf{d}_{local}.z > \epsilon$ (ray points towards positive $z$, assuming square faces $+z$? Or check against specific orientation). [S2, unofficial] checks `localStart.z < 0` and `localDirec.z > 0`, implying the ray starts behind the plane ($z<0$) and moves towards it ($z>0$).
    *   $t > \epsilon$ (intersection in front of ray).

3.  **Hit Position:**
    $$ \mathbf{p}_{local} = \mathbf{o}_{local} + t \mathbf{d}_{local} $$

4.  **Boundary Check:**
    Check if $\mathbf{p}_{local}.x$ and $\mathbf{p}_{local}.y$ are within $[-0.5, 0.5]$:
    $$ -0.5 \le \mathbf{p}_{local}.x \le 0.5 \quad \text{and} \quad -0.5 \le \mathbf{p}_{local}.y \le 0.5 $$

5.  **UV Coordinates:**
    $$ u = \mathbf{p}_{local}.x + 0.5 $$
    $$ v = \mathbf{p}_{local}.y + 0.5 $$
    (Mapping $[-0.5, 0.5]$ to $[0, 1]$).

6.  **Back-Transform (Optional):**
    If world-space point is needed:
    $$ \mathbf{p}_{world} = \mathbf{M}_{local \to world} \cdot [\mathbf{p}_{local}.x, \mathbf{p}_{local}.y, 0, 1]^T $$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_square_intersect_explicit(ray_o, ray_d, corner, v1, v2):
    """
    Intersect ray with square defined by corner, v1, v2.
    Returns (hit, t, u, v, point)
    """
    # 1. Compute normal
    w = cross(v1, v2)
    w_dot_w = dot(w, w)
    if w_dot_w < 1e-9:
        return False, 0, 0, 0, None  # Degenerate square

    n = w / sqrt(w_dot_w)

    # 2. Ray-plane intersection
    denom = dot(n, ray_d)
    if abs(denom) < 1e-9:
        return False, 0, 0, 0, None  # Parallel

    t = dot(n, corner - ray_o) / denom
    if t < 1e-9:
        return False, 0, 0, 0, None  # Behind ray

    # 3. Intersection point
    p = ray_o + t * ray_d

    # 4. Boundary check
    p_local = p - corner
    alpha = dot(w, cross(p_local, v2)) / w_dot_w
    beta = dot(w, cross(v1, p_local)) / w_dot_w

    # Check bounds (strictly inside as per [S1, unofficial])
    if alpha > 0.0 and alpha < 1.0 and beta > 0.0 and beta < 1.0:
        return True, t, alpha, beta, p
    else:
        return False, 0, 0, 0, None

def ray_square_intersect_transform(ray_o, ray_d, world_to_local):
    """
    Intersect ray with canonical square in local space [-0.5, 0.5]x[-0.5, 0.5] at z=0.
    Uses transform matrix. Returns (hit, t, u, v, point)
    """
    # 1. Transform ray to local space
    # Transform origin (w=1)
    o_local_x = dot(world_to_local[0], vec4(ray_o, 1.0))
    o_local_y = dot(world_to_local[1], vec4(ray_o, 1.0))
    o_local_z = dot(world_to_local[2], vec4(ray_o, 1.0))
    o_local = vec3(o_local_x, o_local_y, o_local_z)

    # Transform direction (w=0)
    d_local_x = dot(world_to_local[0], vec4(ray_d, 0.0))
    d_local_y = dot(world_to_local[1], vec4(ray_d, 0.0))
    d_local_z = dot(world_to_local[2], vec4(ray_d, 0.0))
    d_local = normalize(vec3(d_local_x, d_local_y, d_local_z))

    # 2. Ray-plane intersection at z=0
    # Check if ray points towards plane (z>0) and starts behind (z<0)
    if d_local.z <= 1e-9 or o_local.z >= -1e-9:
        return False, 0, 0, 0, None

    t = -o_local.z / d_local.z
    if t < 1e-9:
        return False, 0, 0, 0, None

    # 3. Hit position in local space
    p_local_x = o_local.x + t * d_local.x
    p_local_y = o_local.y + t * d_local.y
    # p_local_z is 0

    # 4. Boundary check
    if -0.5 <= p_local_x <= 0.5 and -0.5 <= p_local_y <= 0.5:
        u = p_local_x + 0.5
        v = p_local_y + 0.5
        # Transform point back to world space if needed
        # p_world = local_to_world * vec4(p_local, 0, 1)
        return True, t, u, v, None # Return local point or transform back
    else:
        return False, 0, 0, 0, None
```

### Parameters
*   $\epsilon$: Epsilon for floating-point comparisons. Typical value: $10^{-9}$ to $10^{-7}$ [G].
*   `leastT`: In batched scenarios, the minimum $t$ found so far. Initialized to infinity or a large value (e.g., 1000) [S2, unofficial].

### Edge Cases and Pitfalls
1.  **Degenerate Squares:** If $\mathbf{v}_1$ and $\mathbf{v}_2$ are parallel or zero, the normal is undefined. Check $\|\mathbf{v}_1 \times \mathbf{v}_2\| < \epsilon$ [S1, unofficial].
2.  **Parallel Ray:** If $\mathbf{n} \cdot \mathbf{d} \approx 0$, the ray is parallel to the plane. No intersection [S1, unofficial].
3.  **Back-face Culling:** The sign of $\mathbf{n} \cdot \mathbf{d}$ determines if the ray hits the front or back face. Depending on the application, back faces may be ignored. [S2, unofficial] assumes a specific orientation ($z>0$ direction).
4.  **Precision:** When checking boundaries, use a small epsilon to avoid missing hits due to floating-point errors at the edges [G].
5.  **Transform Normalization:** When transforming the ray direction, it must be normalized after transformation because the transformation matrix may include non-uniform scaling [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per square. For $N$ squares, $O(N)$ if tested individually.
*   **Memory Complexity:** $O(1)$ auxiliary space.
*   **Strengths:**
    *   Analytic solution is exact and fast.
    *   Transform-based method allows efficient batching and GPU-friendly code [S2, unofficial].
    *   Explicit method handles arbitrary oriented squares without matrix inversion.
*   **Limitations:**
    *   Only works for planar squares.
    *   Transform method requires pre-computed world-to-local matrices.
    *   Precision issues near edges or parallel rays.
*   **When to Use:**
    *   Rendering flat geometry, light sources, or skyboxes composed of quads.
    *   When high performance is required for large numbers of small planar primitives.
*   **When Not to Use:**
    *   For curved surfaces (use sphere, cylinder, etc.).
    *   For non-planar or deformed quads (use triangle decomposition or higher-order primitives).

## Background
Ray-square intersection is a fundamental primitive test in computer graphics, particularly in ray tracing and path tracing. It is often implemented as part of a broader ray-primitive intersection module. The explicit basis method is derived from standard computational geometry techniques for point-in-convex-polygon tests. The transform-based method is common in modern rendering engines and GPU shaders, where objects are often defined in local space and transformed to world space. The specific implementation details in [S1, unofficial] and [S2, unofficial] reflect common practices in volumetric path tracing and utility math libraries.

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] GitHub - zalo/MathUtilities: A collection of some of the neat math and physics tricks that I've collected over the last few y..., https://github.com/zalo/MathUtilities, fetched 2026-10-30 (unofficial)
