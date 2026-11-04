# Quaternion Rotation
Also called: Quaternion Rotation of Vector, Quaternion-based vector rotation, Ray Rotation via Quaternion
## Overview
Quaternion Rotation applies a 3D rotation to a vector using quaternion algebra [S1, unofficial][S2, unofficial][S3, unofficial]. It sets itself apart from [SE2 Lie Group Implementation](3144-se2-lie-group-implementation.md) and [2D Rotation](3357-2d-rotation.md) by operating in 3D space with unit quaternions, and from [Quaternion to SO3 Conversion](2305-quaternion-to-so3-conversion.md) by directly yielding a rotated vector rather than an intermediate rotation matrix [G].
## How it works
**Inputs:**
- $q$: A quaternion tensor of shape $(..., 4)$ representing the rotation, typically in $(w, x, y, z)$ layout [S1, unofficial][S2, unofficial].
- $v$: A 3D vector tensor of shape $(..., 3)$ to be rotated [S1, unofficial].

**Outputs:**
- $v'$: The rotated 3D vector tensor of shape $(..., 3)$ [S1, unofficial].

**Data Structures:**
- Quaternions are stored as 4-element vectors/tensors [S1, unofficial].
- Vectors are stored as 3-element vectors/tensors [S1, unofficial].

**Steps:**
1. **Normalize $q$**: Ensure $q$ is a unit quaternion. Compute $q_{norm} = q / \|q\|$ [S2, unofficial][S3, unofficial].
2. **Extract Components**: Let $w = q[0]$ be the scalar part and $q_v = (x, y, z)$ be the vector part [S1, unofficial][S2, unofficial].
3. **Compute Intermediate Vector $t$**: Calculate the cross product of $q_v$ and $v$, scaled by 2: $t = 2 (q_v \times v)$ [S1, unofficial][S2, unofficial].
4. **Compute Second Cross Product**: Calculate the cross product of $q_v$ and $t$: $u = q_v \times t$ [S1, unofficial][S2, unofficial].
5. **Combine Terms**: The rotated vector is the sum of $u$, the scaled scalar part of $q$ times $t$, and the original vector $v$: $v' = u + (w \cdot t) + v$ [S1, unofficial][S2, unofficial].

**Alternative Formulation (Hamilton Product):**
1. Normalize $q$ [S3, unofficial].
2. Treat $v$ as a pure quaternion $v_q = (0, v_x, v_y, v_z)$ [S3, unofficial].
3. Compute the conjugate $q^* = (w, -x, -y, -z)$ [S3, unofficial].
4. Compute the Hamilton product $p = q \otimes v_q$ [S3, unofficial].
5. Compute the Hamilton product $result = p \otimes q^*$ [S3, unofficial].
6. Return the vector part $(result_x, result_y, result_z)$ of $result$ [S3, unofficial].

**Pseudocode (Optimized Cross-Product Form):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quaternion_rotate_vector(q, v):
    """
    Rotates vector v by quaternion q.
    q: [w, x, y, z]
    v: [vx, vy, vz]
    """
    # 1. Normalize q
    norm_q = (q[0]**2 + q[1]**2 + q[2]**2 + q[3]**2)**0.5
    if norm_q > 0:
        q = [q[0]/norm_q, q[1]/norm_q, q[2]/norm_q, q[3]/norm_q]
    
    w, x, y, z = q
    vx, vy, vz = v
    
    # 2. Extract vector part of q
    # qv = [x, y, z]
    
    # 3. Compute t = 2 * cross(qv, v)
    tx = 2 * (y * vz - z * vy)
    ty = 2 * (z * vx - x * vz)
    tz = 2 * (x * vy - y * vx)
    
    # 4. Compute u = cross(qv, t)
    ux = y * tz - z * ty
    uy = z * tx - x * tz
    uz = x * ty - y * tx
    
    # 5. v' = u + w*t + v
    rx = ux + w * tx + vx
    ry = uy + w * ty + vy
    rz = uz + w * tz + vz
    
    return [rx, ry, rz]
```

**Edge Cases and Pitfalls:**
- **Non-unit Quaternion**: If $q$ is not normalized, the rotation will include scaling. Normalization is required [S2, unofficial][S3, unofficial].
- **Zero Vector**: Rotating a zero vector yields a zero vector [G].
- **Precision**: Floating-point errors can accumulate if quaternions are not re-normalized periodically in iterative applications [G].

## Subtopics

### [Quaternion to SO3 Conversion](2305-quaternion-to-so3-conversion.md)
Quaternion to SO3 Conversion transforms a quaternion representation into a 3x3 rotation matrix $R \in SO(3)$, or vice versa, enabling the use of quaternions for compact rotation storage and matrices for linear algebra operations such as matrix multiplication and transformation composition. This method is distinct from [Quaternion Rotation](2302-quaternion-rotation.md), which applies a rotation to a vector, by providing the explicit basis transformation matrix [G]. It differs from [Quaternion Operations](3474-quaternion-operations.md) by focusing specifically on the structural conversion between the two mathematical representations rather than algebraic manipulation of quaternions themselves [G].

### [Quaternion Operations](3474-quaternion-operations.md)
Defines conjugation, multiplication, and vector rotation for quaternions in $(x, y, z, w)$ layout. These operations form the computational core for [Quaternion Rotation](2302-quaternion-rotation.md) in volume rendering pipelines.

## Operating characteristics
- **Time Complexity**: $O(1)$ per vector [G].
- **Memory Complexity**: $O(1)$ auxiliary space [G].
- **Strengths**: Compact storage (4 floats), efficient for composition [G].
- **Limitations**: Less intuitive for visualization than matrices; interpolation requires spherical linear interpolation (slerp) which is more complex than linear interpolation of matrices [G].
- **When to use**: Preferred for rotating many vectors in volume rendering pipelines [S1, unofficial][S3, unofficial].
- **When not to use**: When interfacing with APIs that strictly require $3 \times 3$ or $4 \times 4$ matrices [G].

## Background
The method is used in volume rendering pipelines for camera pose transformation and vector rotation [S1, unofficial][S2, unofficial][S3, unofficial]. The optimized cross-product formulation is widely implemented in GPU shaders and deep learning frameworks for its efficiency [G].

## Sources
[S1] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S2] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S3] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
