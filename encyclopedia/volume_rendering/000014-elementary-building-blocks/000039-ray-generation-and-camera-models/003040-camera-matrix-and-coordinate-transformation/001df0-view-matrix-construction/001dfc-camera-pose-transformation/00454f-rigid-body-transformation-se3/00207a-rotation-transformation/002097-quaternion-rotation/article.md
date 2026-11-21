# Quaternion Rotation

Also called: Quaternion Rotation of Vector, Quaternion-based vector rotation, Ray Rotation via Quaternion

## Overview
Quaternion Rotation computes the rotated position of a 3D vector $v$ given a quaternion $q$ representing the rotation [S1, S2, S3, unofficial]. It solves the problem of applying 3D rotations to vectors in volume rendering pipelines without constructing an intermediate $3 \times 3$ matrix [S1, S2, unofficial]. It differs from [Quaternion to SO3 Conversion](2305-quaternion-to-so3-conversion.md) by operating directly on the vector components rather than converting the rotation representation to a matrix format [G].

## How it works
**Inputs:**
- $q$: A quaternion tensor of shape (..., 4) representing the rotation [S1, unofficial]. Convention: scalar part $w$ followed by vector part $(x, y, z)$ [S2, S3, unofficial].
- $v$: A 3D vector tensor of shape (..., 3) to be rotated [S1, S3, unofficial].

**Outputs:**
- $v'$: The rotated 3D vector tensor of shape (..., 3) [S1, S3, unofficial].

**Data Structures:**
- Quaternion $q$: Stored as a 4-element array or tensor [S1, S3, unofficial].
- Vector $v$: Stored as a 3-element array or tensor [S1, S3, unofficial].

**Steps:**
1. **Normalization**: If $q$ is not guaranteed to be a unit quaternion, normalize it: $q_{norm} = q / \|q\|$. Let the normalized components be $w, x, y, z$ [S3, unofficial].
2. **Decomposition**: Extract the scalar part $s = w$ and the vector part $\mathbf{u} = (x, y, z)$ from $q$ [S2, unofficial].
3. **Computation**: Compute the rotated vector $v'$ using the vector algebra formula [S2, unofficial]:
   $$ v' = \mathbf{u} \cdot (2 \cdot (\mathbf{u} \cdot v)) + v \cdot (s^2 - \mathbf{u} \cdot \mathbf{u}) + (\mathbf{u} \times v) \cdot (2s) $$
   Where $\cdot$ denotes the dot product and $\times$ denotes the cross product [S2, unofficial].
   *Alternative Derivation (Hamilton Product)*: Treat $v$ as a pure quaternion by appending 0 to $v$ to form a 4D quaternion (vector-first convention) [S3, unofficial]. Compute the conjugate $q^* = (w, -x, -y, -z)$ [S3, unofficial]. Compute $v'_q = q \otimes v_q \otimes q^*$ using quaternion multiplication, then extract the first 3 components [S3, unofficial].

**Parameters:**
- No additional parameters. The rotation is fully defined by $q$ and $v$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Non-unit Quaternion**: If $q$ is not normalized, the result will be scaled incorrectly. Normalization is required [S3, unofficial].
- **Precision**: Standard floating-point arithmetic may introduce minor errors for very small rotations or near-zero vectors [G].
- **Input Format**: Ensure the quaternion convention (scalar-first vs. vector-first) matches the implementation. The sources assume scalar-first $(w, x, y, z)$ for the rotation quaternion and vector-first $(v_x, v_y, v_z, 0)$ for the pure quaternion [S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quaternion_rotate_vector(q: list[float], v: list[float]) -> list[float]:
    """
    Rotates vector v by quaternion q using the vector algebra method [S2, unofficial].
    q: [w, x, y, z]
    v: [vx, vy, vz]
    Returns: [vx', vy', vz']
    """
    # 1. Normalize q if necessary [S3, unofficial]
    w, x, y, z = q
    norm_sq = w*w + x*x + y*y + z*z
    if norm_sq != 1.0:
        inv_norm = 1.0 / (norm_sq ** 0.5)
        w *= inv_norm
        x *= inv_norm
        y *= inv_norm
        z *= inv_norm

    # 2. Decompose [S2, unofficial]
    s = w
    u_x, u_y, u_z = x, y, z
    v_x, v_y, v_z = v

    # 3. Compute terms [S2, unofficial]
    # term_a = u * (dot(u, v) * 2.0)
    dot_uv = u_x*v_x + u_y*v_y + u_z*v_z
    term_a_x = u_x * (dot_uv * 2.0)
    term_a_y = u_y * (dot_uv * 2.0)
    term_a_z = u_z * (dot_uv * 2.0)

    # term_b = v * ((s * s) - dot(u, u))
    dot_uu = u_x*u_x + u_y*u_y + u_z*u_z
    scalar_b = (s * s) - dot_uu
    term_b_x = v_x * scalar_b
    term_b_y = v_y * scalar_b
    term_b_z = v_z * scalar_b

    # term_c = cross(u, v) * (2.0 * s)
    cross_x = u_y*v_z - u_z*v_y
    cross_y = u_z*v_x - u_x*v_z
    cross_z = u_x*v_y - u_y*v_x
    scalar_c = 2.0 * s
    term_c_x = cross_x * scalar_c
    term_c_y = cross_y * scalar_c
    term_c_z = cross_z * scalar_c

    # 4. Return sum [S2, unofficial]
    return [term_a_x + term_b_x + term_c_x,
            term_a_y + term_b_y + term_c_y,
            term_a_z + term_b_z + term_c_z]
```

## Subtopics

### Axis Extraction from Quaternion
Extracting the basis axes of a rotated frame (e.g., for Gaussian Splatting) is a direct application of Quaternion Rotation [S4, unofficial]. To extract the local X-axis, apply the rotation to the vector $(1, 0, 0)$ [S4, unofficial]. To extract the local Y-axis, apply the rotation to $(0, 1, 0)$ [S4, unofficial]. To extract the local Z-axis, apply the rotation to $(0, 0, 1)$ [S4, unofficial]. This avoids constructing the full rotation matrix when only specific basis vectors are needed [S4, unofficial].

## Operating characteristics
Time complexity is $O(1)$ per vector rotation [G]. Memory complexity requires $O(1)$ auxiliary space [G]. Strengths include no gimbal lock and computational efficiency for single rotations [G]. Limitations include the requirement for normalization to maintain unit length and the need for quaternion multiplication rather than addition when composing rotations [G]. It should be used when rotating individual vectors (e.g., normals, rays, points) by a known quaternion orientation [G]. It should not be used when transforming entire coordinate systems or when matrix operations are already dominant in the pipeline, as [Rotation Matrix Computation](2280-rotation-matrix-computation.md) may be more cache-friendly for batched operations [G].

## Background
Quaternion rotation is widely used in camera pose transformations, skeletal animation, and 3D Gaussian Splatting [S3, S4, unofficial]. The historical origin and introduction date are Not given in the sources.

## Sources
[S1] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S2] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S3] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S4] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
