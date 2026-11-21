# Ray-Image Frame Intersection

Also called: Ray-Box Edge Intersection, Ray-Frame Intersection

## Overview
Ray-Image Frame Intersection computes the intersection of a camera-space ray with the four lines parallel to the image frame axes (the boundaries $x=0, x=1, y=0, y=1$) in normalized image coordinates [S1, S2, unofficial]. It serves as a geometric primitive to determine where a ray exits or enters the image plane boundaries, which is essential for culling rays that miss the image or for mapping ray parameters to image-space coordinates [S1, S2, unofficial]. It differs from [Ray-Volume and Bounding Box Intersection](10057-ray-volume-and-bounding-box-intersection.md) by operating specifically on the normalized image frame boundaries rather than a 3D volumetric bounding box, and from [Implicit Surface and SDF Intersection](2795-implicit-surface-and-sdf-intersection.md) by using analytic linear solutions rather than iterative root finding.

## How it works
**Inputs and Outputs**
*   **Inputs**: Ray origin $\mathbf{o} = (o_s, o_o, o_z)$, ray direction $\mathbf{d} = (d_s, d_o, d_z)$, and camera intrinsics consisting of focal lengths $f_s, f_o$ and principal points $c_s, c_o$. The dimension to intersect is specified (e.g., horizontal or vertical). A target coordinate value $v$ (typically $0.0$ or $1.0$) is provided for the boundary line.
*   **Outputs**: The ray parameter $t$, the 2D image coordinate of the intersection point $(x, y)$, and the 3D world coordinate $\mathbf{xyz}$. A validity flag indicating if the intersection is within bounds and in front of the camera.

**Data Structures**
The method operates on scalar components extracted from tensors representing ray origins, directions, and intrinsics. No complex data structures are required; computations are element-wise.

**Steps**
1.  **Extract Intrinsics and Ray Components**: Retrieve focal lengths $f_s, f_o$, principal points $c_s, c_o$, ray origin components $o_s, o_o, o_z$, and ray direction components $d_s, d_o, d_z$ [S1, S2, unofficial].
2.  **Compute Normalized Coordinate $c$**: Calculate the normalized coordinate $c$ corresponding to the target boundary value $v$ (e.g., $0.0$ or $1.0$) along the intersected dimension:
    $$ c = \frac{v - c_s}{f_s} $$
    [S1, S2, unofficial]
3.  **Compute Ray Parameter $t$**: Solve for the ray parameter $t$ where the ray intersects the plane defined by the normalized coordinate $c$. The formula is:
    $$ t = \frac{c o_z - o_s}{d_s - c d_z} $$
    [S1, S2, unofficial]
4.  **Compute Other Coordinate**: Calculate the coordinate value for the orthogonal dimension at the intersection point. Let $o_o$ and $d_o$ be the origin and direction components for the other dimension, and $c_o$ be its principal point:
    $$ \text{coordinate\_other} = c_o + \frac{f_o (o_o (c d_z - d_s) + d_o (o_s - c o_z))}{d_z o_s - d_s o_z} $$
    [S1, S2, unofficial]
5.  **Construct Intersection Points**:
    *   2D Image Point: Construct the 2D point $\mathbf{xy}$ by placing the computed `coordinate_other` in the orthogonal dimension and the target value $v$ in the intersected dimension.
    *   3D World Point: Compute the 3D point $\mathbf{xyz}$ using the ray equation:
        $$ \mathbf{xyz} = \mathbf{o} + t \mathbf{d} $$
    [S1, S2, unofficial]
6.  **Validate Intersection**: The intersection is valid if:
    *   The 2D point $\mathbf{xy}$ is within bounds $[-\epsilon, 1+\epsilon]$ for both dimensions.
    *   The 3D point $\mathbf{xyz}$ is in front of the camera ($z > -\epsilon$).
    *   The ray parameter $t > -\epsilon$.
    [S1, S2, unofficial]

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_image_frame_intersection(o_s, o_o, o_z, d_s, d_o, d_z, f_s, f_o, c_s, c_o, v, dim):
    """
    Computes intersection of a ray with an image frame boundary.
    
    Args:
        o_s, o_o, o_z: Ray origin components.
        d_s, d_o, d_z: Ray direction components.
        f_s, f_o: Focal lengths.
        c_s, c_o: Principal points.
        v: Target coordinate value (0.0 or 1.0).
        dim: 's' for horizontal (x), 'o' for vertical (y).
        
    Returns:
        t: Ray parameter.
        xy: 2D image coordinates.
        xyz: 3D world coordinates.
        valid: Boolean validity flag.
    """
    epsilon = 1e-6
    
    # Step 1 & 2: Compute normalized coordinate c
    c = (v - c_s) / f_s
    
    # Step 3: Compute ray parameter t
    # Note: Division by zero if d_s - c * d_z is zero, handled by validity check
    denominator_t = d_s - c * d_z
    if abs(denominator_t) < epsilon:
        return None, None, None, False
        
    t = (c * o_z - o_s) / denominator_t
    
    # Step 4: Compute other coordinate
    # The formula uses specific variable names from sources:
    # coordinate_other = co + (fo * (oo * (c * dz - ds) + do * (os - c * oz))) / (dz * os - ds * oz)
    numerator_other = f_o * (o_o * (c * d_z - d_s) + d_o * (o_s - c * o_z))
    denominator_other = d_z * o_s - d_s * o_z
    
    if abs(denominator_other) < epsilon:
        return None, None, None, False
        
    coordinate_other = c_o + numerator_other / denominator_other
    
    # Step 5: Construct points
    if dim == 's':
        # Intersecting horizontal boundary (x = v)
        x = v
        y = coordinate_other
    else:
        # Intersecting vertical boundary (y = v)
        x = coordinate_other
        y = v
        
    xy = (x, y)
    xyz = (o_s + t * d_s, o_o + t * d_o, o_z + t * d_z)
    
    # Step 6: Validate
    # Check bounds [-epsilon, 1+epsilon]
    in_bounds = (-epsilon <= x <= 1 + epsilon) and (-epsilon <= y <= 1 + epsilon)
    # Check in front of camera
    in_front = xyz[2] > -epsilon
    # Check t is positive (or near zero)
    t_valid = t > -epsilon
    
    valid = in_bounds and in_front and t_valid
    
    return t, xy, xyz, valid
```

**Parameters**
*   $\epsilon$: Epsilon for floating-point comparisons. Typical value: $10^{-6}$ [S1, S2, unofficial].
*   $v$: Target coordinate. Values are typically $0.0$ or $1.0$ [S1, S2, unofficial].

**Edge Cases and Pitfalls**
*   **Division by Zero**: If $d_s - c d_z = 0$ or $d_z o_s - d_s o_z = 0$, the ray is parallel to the intersection plane or the geometry is degenerate. The method should return invalid [S1, S2, unofficial].
*   **Precision**: Floating-point errors may cause points slightly outside $[0, 1]$ to be considered valid if $\epsilon$ is not used. Conversely, strict bounds may reject valid intersections due to precision loss [S1, S2, unofficial].
*   **Degenerate Input**: If the ray origin is on the image plane or the direction is zero, the intersection is undefined [G].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per ray per boundary. The computation involves a fixed number of arithmetic operations.
*   **Memory Complexity**: $O(1)$ auxiliary space.
*   **Performance**: Highly efficient for parallel execution on GPUs due to lack of branching and simple arithmetic [G].
*   **Strengths**: Exact analytic solution; no iteration required; suitable for culling and boundary checks.
*   **Limitations**: Only computes intersections with the image frame boundaries, not with arbitrary geometry. Requires valid camera intrinsics [S1, S2, unofficial].

## Background
The method is used in sparse voxel and volume rendering pipelines, particularly in implementations like VolSplat and VoLoDSplat, for efficient ray-culling and coordinate mapping [S1, S2, unofficial]. It is a fundamental building block in ray-primitive intersection families for image-space based rendering.

## Sources
[S1] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S2] pulangk97/VoLoDSplat, https://github.com/pulangk97/VoLoDSplat, fetched 2026-10-27 (unofficial)
