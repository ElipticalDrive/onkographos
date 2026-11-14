# Quaternion Rotation

Also called: Quaternion Rotation of Vector, Quaternion-based vector rotation, Ray Rotation via Quaternion

## Overview
Quaternion Rotation applies a 3D rotation defined by a unit quaternion $q$ to a 3D vector $v$ [S1, S2, S3, unofficial]. It solves the problem of efficiently rotating rays or points in camera space without the computational overhead of $4 \times 4$ matrices or the singularities of Euler angles [S3, S4, unofficial]. It sets itself apart from [Quaternion to SO3 Conversion](2305-quaternion-to-so3-conversion.md) by operating directly on vectors rather than producing a basis matrix, and from [Quaternion Operations](3474-quaternion-operations.md) by providing the specific application logic for vector transformation rather than just algebraic primitives [G].

## How it works
**Inputs:**
- $q$: A quaternion $(w, x, y, z)$ representing the rotation. Must be a unit quaternion ($|q|=1$) [S3, unofficial].
- $v$: A 3D vector $(v_x, v_y, v_z)$ to be rotated [S1, S3, unofficial].

**Outputs:**
- $v'$: The rotated 3D vector [S3, unofficial].

**Data Structures:**
- Quaternion $q$: 4 components $(w, x, y, z)$, typically stored as `float` or `f32` [S2, unofficial].
- Vector $v$: 3 components $(x, y, z)$, typically stored as `float` or `f32` [S1, unofficial].

**Steps:**
1. **Normalize** $q$ to ensure it is a unit quaternion. If $q$ is not guaranteed to be normalized, compute $q_{norm} = q / \|q\|$ [S3, unofficial].
2. **Extract** the scalar part $s = q.w$ and the vector part $\mathbf{u} = (q.x, q.y, q.z)$ [S2, unofficial].
3. **Compute** the rotated vector $v'$ using the expanded Hamilton product formula, which avoids explicit quaternion multiplication overhead [S2, unofficial]:
   $$ v' = \mathbf{u} \cdot (2 \cdot (\mathbf{u} \cdot v)) + v \cdot (s^2 - \|\mathbf{u}\|^2) + \mathbf{u} \times v \cdot (2s) $$
   Where:
   - $\cdot$ is the dot product.
   - $\times$ is the cross product.
   - $\|\mathbf{u}\|^2$ is the squared magnitude of the vector part $\mathbf{u}$.
   - $s^2 - \|\mathbf{u}\|^2$ is equivalent to $w^2 - (x^2 + y^2 + z^2)$ [S2, unofficial].

**Alternative Implementation (Conjugation):**
1. Normalize $q$ [S3, unofficial].
2. Treat $v$ as a pure quaternion $v_q = (0, v_x, v_y, v_z)$ [S3, unofficial].
3. Compute the conjugate $q^* = (w, -x, -y, -z)$ [S3, unofficial].
4. Compute $v'_{q} = q \otimes v_q \otimes q^*$ using quaternion multiplication [S3, unofficial].
5. Return the vector part $(v'_x, v'_y, v'_z)$ of the result [S3, unofficial].

**Pseudocode (Expanded Formula):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quaternion_rotate_vector(q, v):
    """
    Rotates vector v by quaternion q using the expanded Hamilton product formula.
    q: tuple or list of 4 floats (w, x, y, z)
    v: tuple or list of 3 floats (x, y, z)
    Returns: tuple or list of 3 floats (x', y', z')
    """
    # Extract components
    s = q[0]
    u = (q[1], q[2], q[3])
    
    # Normalize q if necessary (assuming input might not be normalized)
    norm_q = (s**2 + u[0]**2 + u[1]**2 + u[2]**2)**0.5
    if norm_q > 0:
        s = s / norm_q
        u = (u[0]/norm_q, u[1]/norm_q, u[2]/norm_q)
        
    # Compute dot product of u and v
    dot_uv = u[0]*v[0] + u[1]*v[1] + u[2]*v[2]
    
    # Compute cross product of u and v
    cross_uv = (
        u[1]*v[2] - u[2]*v[1],
        u[2]*v[0] - u[0]*v[2],
        u[0]*v[1] - u[1]*v[0]
    )
    
    # Term a: u * (2 * dot(u, v))
    term_a = (
        u[0] * (2.0 * dot_uv),
        u[1] * (2.0 * dot_uv),
        u[2] * (2.0 * dot_uv)
    )
    
    # Term b: v * (s^2 - dot(u, u))
    dot_uu = u[0]**2 + u[1]**2 + u[2]**2
    scalar_b = s**2 - dot_uu
    term_b = (
        v[0] * scalar_b,
        v[1] * scalar_b,
        v[2] * scalar_b
    )
    
    # Term c: cross(u, v) * (2 * s)
    term_c = (
        cross_uv[0] * (2.0 * s),
        cross_uv[1] * (2.0 * s),
        cross_uv[2] * (2.0 * s)
    )
    
    # Result: a + b + c
    result = (
        term_a[0] + term_b[0] + term_c[0],
        term_a[1] + term_b[1] + term_c[1],
        term_a[2] + term_b[2] + term_c[2]
    )
    
    return result
```

**Parameters:**
- $q$: Quaternion tensor of shape $(..., 4)$ [S1, unofficial].
- $v$: 3D vector tensor of shape $(..., 3)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Non-unit Quaternion:** If $q$ is not normalized, the rotation will include scaling. Normalization is required for pure rotation [S3, unofficial].
- **Precision:** Floating-point errors can accumulate if quaternions are not re-normalized periodically in iterative applications [G].
- **Degenerate Input:** If $q$ is $(0,0,0,0)$, normalization fails. Guard against zero-magnitude quaternions [G].

## Subtopics

### [Quaternion to SO3 Conversion](2305-quaternion-to-so3-conversion.md)
Transforms a quaternion representation into a $3 \times 3$ rotation matrix $R \in SO(3)$, or vice versa, enabling the use of quaternions for compact rotation storage and matrices for linear algebra operations such as matrix multiplication and transformation composition. This method is distinct from [Quaternion Rotation](2302-quaternion-rotation.md), which applies a rotation to a vector, by providing the explicit basis transformation matrix [G]. It differs from [Quaternion Operations](3474-quaternion-operations.md) by focusing specifically on the structural conversion between the two mathematical representations rather than algebraic manipulation of quaternions themselves [G].

### [Quaternion Operations](3474-quaternion-operations.md)
Defines the fundamental arithmetic operations for unit quaternions used in 3D graphics: conjugation, multiplication, and vector rotation. It solves the problem of efficiently representing and composing 3D rotations without the singularities of Euler angles or the computational overhead of $4 \times 4$ matrices. It sets itself apart from [Quaternion Rotation](2302-quaternion-rotation.md) by providing the underlying algebraic primitives (multiplication and conjugation) required to implement the higher-level rotation logic, and from [Quaternion to SO3 Conversion](2305-quaternion-to-so3-conversion.md) by focusing on quaternion-native arithmetic rather than matrix conversion.

### Axis Extraction from Quaternion
To extract the X-axis of a Gsplat, apply the quaternion rotation to the local X-axis vector $(1,0,0)$ using `quatVec(quaternion, vec3(1.0, 0.0, 0.0))` [S4, unofficial]. To extract the Y-axis, apply the rotation to $(0,1,0)$ using `quatVec(quaternion, vec3(0.0, 1.0, 0.0))` [S4, unofficial]. To extract the Z-axis, apply the rotation to $(0,0,1)$ using `quatVec(quaternion, vec3(0.0, 0.0, 1.0))` [S4, unofficial].

## Operating characteristics
**Time Complexity:** $O(1)$ per vector rotation. The expanded formula requires a constant number of multiplications and additions (approx. 15-20 FLOPs) [S2, unofficial]. The conjugation method requires quaternion multiplications, which are slightly more expensive but still $O(1)$ [S3, unofficial].

**Memory Complexity:** $O(1)$ auxiliary space. No additional data structures are needed beyond the input and output vectors [S1, S2, unofficial].

**Measured Performance:**
- In GPU-based implementations (e.g., PyTorch tensors), vectorized operations on batches of quaternions and vectors are highly efficient [S1, S3, unofficial].
- CPU implementations using the expanded formula avoid the overhead of quaternion class instantiation and multiplication functions [S2, unofficial].

**Strengths:**
- Computational efficiency compared to matrix multiplication [S3, unofficial].
- No gimbal lock issues [S3, unofficial].
- Compact storage (4 floats vs 9 for a matrix) [S3, unofficial].

**Limitations:**
- Requires normalization to ensure pure rotation [S3, unofficial].
- Less intuitive for composition of multiple rotations compared to matrix multiplication (though quaternion multiplication is efficient) [G].

**When to Use:**
- When rotating individual vectors or rays efficiently.
- When memory bandwidth is a constraint and compact rotation storage is needed.

**When Not to Use:**
- When composing many rotations and intermediate results need to be stored as matrices for further linear algebra operations (use [Quaternion to SO3 Conversion](2305-quaternion-to-so3-conversion.md) instead) [G].
- When the rotation is already represented as a matrix and matrix multiplication is more convenient [G].

## Background
Quaternion rotation was introduced as a method to represent 3D rotations without the singularities of Euler angles. It is widely used in computer graphics, robotics, and physics simulations. The method is based on Hamilton's quaternion algebra [S3, unofficial]. In the context of volume rendering and 3D Gaussian Splatting, it is used to rotate the covariance matrices or the viewing rays efficiently [S3, S4, unofficial].

## Sources
[S1] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S2] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S3] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S4] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
