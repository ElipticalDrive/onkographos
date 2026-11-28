# Transformation Matrix Construction
Also called: TRS Matrix Construction, TTRS Matrix Construction

## Overview
Computes a 4x4 homogeneous transformation matrix from translation, rotation (Euler angles), and scale factors for 3D geometry [S1, unofficial]. It extends [Scaling and Rotation Matrix Construction](7986-scaling-and-rotation-matrix-construction.md) by adding translation to produce a complete model matrix [S1, unofficial].

## How it works
**Inputs:**
- `translation`: A 3D vector $(t_x, t_y, t_z)$ [S1, unofficial].
- `rotation`: Three scalar angles $(x, y, z)$ representing rotations around the X, Y, and Z axes respectively, in radians [S1, unofficial].
- `scale`: A 3D vector $(s_x, s_y, s_z)$ representing scaling factors along each axis [S1, unofficial].

**Outputs:**
- `M`: A 4x4 transformation matrix [S1, unofficial].

**Data Structures:**
- `Mat4`: A 4x4 array of floats, initialized as `{{ {r0c0, r0c1, r0c2, r0c3}, ... }}` [S1, unofficial].
- `Vec3`: A 3D vector with components `.x`, `.y`, `.z` [S1, unofficial].

**Steps:**
1. Construct the X-axis rotation matrix $R_x$ using angle $x$:
   $$
   R_x = \begin{bmatrix}
   1 & 0 & 0 & 0 \\
   0 & \cos x & -\sin x & 0 \\
   0 & \sin x & \cos x & 0 \\
   0 & 0 & 0 & 1
   \end{bmatrix}
   $$
2. Construct the Y-axis rotation matrix $R_y$ using angle $y$:
   $$
   R_y = \begin{bmatrix}
   \cos y & 0 & \sin y & 0 \\
   0 & 1 & 0 & 0 \\
   -\sin y & 0 & \cos y & 0 \\
   0 & 0 & 0 & 1
   \end{bmatrix}
   $$
3. Construct the Z-axis rotation matrix $R_z$ using angle $z$:
   $$
   R_z = \begin{bmatrix}
   \cos z & -\sin z & 0 & 0 \\
   \sin z & \cos z & 0 & 0 \\
   0 & 0 & 1 & 0 \\
   0 & 0 & 0 & 1
   \end{bmatrix}
   $$
4. Construct the scale matrix $S$ using scale factors $s_x, s_y, s_z$:
   $$
   S = \begin{bmatrix}
   s_x & 0 & 0 & 0 \\
   0 & s_y & 0 & 0 \\
   0 & 0 & s_z & 0 \\
   0 & 0 & 0 & 1
   \end{bmatrix}
   $$
5. Multiply the rotation matrices: `rMatrix = matMul(matMul(rMatrixX, rMatrixY), rMatrixZ)` [S1, unofficial]. The source code computes $R_x R_y R_z$, while the formula specifies $M = R_z R_y R_x S$ [S1, unofficial]. The implementation follows the code's multiplication chain [S1, unofficial].
6. Multiply the combined rotation matrix by the scale matrix: `matrix = matMul(rMatrix, sMatrix)` [S1, unofficial].
7. Set the translation components in the last column: `matrix[0][3] = translation.x`, `matrix[1][3] = translation.y`, `matrix[2][3] = translation.z` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def construct_transformation_matrix(translation, rotation, scale):
    tx, ty, tz = translation
    rx, ry, rz = rotation
    sx, sy, sz = scale

    # 1. Construct Rx
    cos_rx = math.cos(rx)
    sin_rx = math.sin(rx)
    Rx = [
        [1, 0, 0, 0],
        [0, cos_rx, -sin_rx, 0],
        [0, sin_rx, cos_rx, 0],
        [0, 0, 0, 1]
    ]

    # 2. Construct Ry
    cos_ry = math.cos(ry)
    sin_ry = math.sin(ry)
    Ry = [
        [cos_ry, 0, sin_ry, 0],
        [0, 1, 0, 0],
        [-sin_ry, 0, cos_ry, 0],
        [0, 0, 0, 1]
    ]

    # 3. Construct Rz
    cos_rz = math.cos(rz)
    sin_rz = math.sin(rz)
    Rz = [
        [cos_rz, -sin_rz, 0, 0],
        [sin_rz, cos_rz, 0, 0],
        [0, 0, 1, 0],
        [0, 0, 0, 1]
    ]

    # 4. Construct S
    S = [
        [sx, 0, 0, 0],
        [0, sy, 0, 0],
        [0, 0, sz, 0],
        [0, 0, 0, 1]
    ]

    # 5. Multiply Rotations (code order: Rx * Ry * Rz)
    R_temp = mat_mul(Rx, Ry)
    R_combined = mat_mul(R_temp, Rz)

    # 6. Multiply by Scale
    M = mat_mul(R_combined, S)

    # 7. Set Translation
    M[0][3] = tx
    M[1][3] = ty
    M[2][3] = tz

    return M

def mat_mul(A, B):
    C = [[0.0] * 4 for _ in range(4)]
    for i in range(4):
        for j in range(4):
            for k in range(4):
                C[i][j] += A[i][k] * B[k][j]
    return C
```

**Edge Cases:**
- Gimbal lock occurs when $y = \pm \pi/2$, aligning the X and Z axes and losing a degree of freedom [G].
- Floating-point errors in trigonometric functions may accumulate during multiplication [G].
- The matrix becomes singular if any scale factor is zero [G].

## Subtopics

### Model Matrix Construction with Coordinate Flip
It creates a translation matrix with $t_x, t_y$ in the last column (Z translation is 0) [S2, unofficial]. It generates a rotation matrix from a quaternion and combines them as `model = (trans @ rot).astype(np.float32)` [S2, unofficial]. If `flip_coords` is True, it post-multiplies by a diagonal matrix with `[-1.0, -1.0, -1.0, 1.0]` to reflect the volume through its center [S2, unofficial]. This flip ensures that the box overlay, axis overlay, and ray-marching all see the same consistently flipped transform, eliminating misalignment [S2, unofficial].

### Rotation Matrix Generation
It generates a 4x4 rotation matrix from an Euler angle around a specified axis [S3, unofficial]. It initializes a 4x4 identity matrix and checks the scipy version [S3, unofficial]. If the version is $\ge$ 1.4.0, it uses `Rotation.from_euler(axis, angle).as_matrix()` [S3, unofficial]. Otherwise (scipy < 1.4.0), it uses `Rotation.from_euler(axis, angle).as_dcm()` [S3, unofficial].

## Operating characteristics
Time complexity is $O(1)$ for fixed 4x4 matrix construction [G]. Memory complexity is $O(1)$ for the 4x4 matrix [G]. Strengths include simple implementation and explicit control over rotation order [G]. Limitations include gimbal lock inherent to Euler angles [G].

## Background
It originated in the `quertzuiop/Volumetric-display` project, an unofficial repository for volumetric rendering [S1, unofficial]. It is used within the `Scene` object to manage transformations for various geometry types including `ParticleGeometry`, `CapsuleGeometry`, `TriangleGeometry`, `SphereGeometry`, `CuboidGeometry`, `MeshGeometry`, and `TextGeometry` [S1, unofficial].

## Sources
[S1] quertzuiop/Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
[S2] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S3] mesh_to_sdf, https://github.com/marian42/mesh_to_sdf, fetched 2026-10-28 (unofficial)
