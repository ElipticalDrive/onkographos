# Transformation Matrix Construction
Also called: TRS Matrix Construction, TTRS Matrix Construction

## Overview
Computes a 4x4 homogeneous transformation matrix from translation, rotation (Euler angles), and scale factors for 3D geometry [S1, unofficial]. It combines the scaling and rotation logic of [Scaling and Rotation Matrix Construction](7986-scaling-and-rotation-matrix-construction.md) with translation to produce a complete model matrix. It is distinguished by its explicit construction of individual axis rotation matrices and their specific multiplication order ($R_z R_y R_x$) before applying scale [S1, unofficial].

## How it works
**Inputs:**
- `translation`: A 3D vector $(t_x, t_y, t_z)$.
- `rotation`: Three scalar angles $(x, y, z)$ representing rotations around the X, Y, and Z axes respectively, in radians.
- `scale`: A 3D vector $(s_x, s_y, s_z)$ representing scaling factors along each axis.

**Outputs:**
- `M`: A 4x4 matrix stored in row-major order (or column-major depending on implementation, but accessed as `matrix[row][col]` in the source).

**Data Structures:**
- `Mat4`: A 4x4 array of floats. In the source, it is initialized as `{{ {r0c0, r0c1, r0c2, r0c3}, ... }}`.
- `Vec3`: A 3D vector with components `.x`, `.y`, `.z`.

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
5. Multiply the rotation matrices in the order $R_z R_y R_x$. Note: The source code performs `matMul(matMul(rMatrixX, rMatrixY), rMatrixZ)`. If the matrices are stored in row-major order and vectors are row vectors, this computes $R_x R_y R_z$. However, the formula section states $M = R_z R_y R_x S$. The code snippet `rMatrix = matMul(matMul(rMatrixX, rMatrixY), rMatrixZ)` combined with `matrix = matMul(rMatrix, sMatrix)` suggests the implementation order might be $R_x R_y R_z S$ if using row-major multiplication, or the source code comments/formulas have a discrepancy. Given the formula $M = R_z R_y R_x S$ is explicit, we follow the formula for the mathematical definition, but the code implements a specific multiplication chain. We will describe the code's actual execution path as it is the implementable artifact.
   *Correction*: The source code `matMul(A, B)` typically means $A \times B$. The code does `rMatrixX * rMatrixY` then result `* rMatrixZ`. This results in $R_x R_y R_z$. The formula says $R_z R_y R_x$. These are different. The source [S1, unofficial] contains both. We must cite the discrepancy. The code is the executable part. The formula is the mathematical claim. We provide the code's logic as the primary implementation path but note the formula.
   *Re-evaluation*: In many graphics APIs (like OpenGL), matrices are column-major. If `Mat4` is column-major, `matMul(A, B)` is $A \times B$. The code `rMatrixX * rMatrixY * rMatrixZ` results in $R_x R_y R_z$. The formula $R_z R_y R_x$ is the standard intrinsic rotation order (Z-Y-X) for extrinsic rotations or X-Y-Z for intrinsic. The source explicitly writes $M = R_z R_y R_x S$. We will present the code's steps as they are, but define the final matrix according to the code's multiplication result, while noting the formula's claim.
   *Decision*: The prompt asks to implement from text. The code is explicit. The formula is explicit. They contradict. We will describe the construction of individual matrices, then state the multiplication order used in the code ($R_x R_y R_z S$) and note the formula claims $R_z R_y R_x S$ [S1, unofficial].
6. Multiply the combined rotation matrix by the scale matrix: $R_{combined} = R_{code\_order} \times S$.
7. Set the translation components in the last column (indices `[0][3]`, `[1][3]`, `[2][3]`):
   - `matrix[0][3] = translation.x`
   - `matrix[1][3] = translation.y`
   - `matrix[2][3] = translation.z`
   - `matrix[3][3] = 1` (implicit in homogeneous coordinates).

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

    # 5. Multiply Rotations
    # Source code: rMatrix = matMul(matMul(rMatrixX, rMatrixY), rMatrixZ)
    # This computes Rx * Ry * Rz
    R_temp = mat_mul(Rx, Ry)
    R_combined = mat_mul(R_temp, Rz)

    # 6. Multiply by Scale
    # Source code: matrix = matMul(rMatrix, sMatrix)
    M = mat_mul(R_combined, S)

    # 7. Set Translation
    M[0][3] = tx
    M[1][3] = ty
    M[2][3] = tz
    # M[3][3] is already 1

    return M

def mat_mul(A, B):
    # Standard 4x4 matrix multiplication
    C = [[0.0]*4 for _ in range(4)]
    for i in range(4):
        for j in range(4):
            for k in range(4):
                C[i][j] += A[i][k] * B[k][j]
    return C
```

**Edge Cases:**
- **Gimbal Lock**: If $y = \pm \pi/2$, the X and Z axes align, losing a degree of freedom. The source does not specify handling [S1, unofficial].
- **Precision**: Floating point errors in trigonometric functions may accumulate.
- **Scale Zero**: If any scale factor is 0, the matrix becomes singular.

## Subtopics

### [Pivot-based Geometry Transformation](12896-pivot-based-geometry-transformation.md)
Applies a transformation matrix to geometric primitives relative to a pivot point, scaling radii/thicknesses by the maximum scale factor.

## Operating characteristics
- **Time Complexity**: Constant time $O(1)$ for fixed 4x4 matrix construction.
- **Memory Complexity**: Constant space $O(1)$ for the 4x4 matrix.
- **Strengths**: Simple to implement; explicit control over rotation order.
- **Limitations**: Euler angles suffer from gimbal lock; the source code's multiplication order ($R_x R_y R_z$) differs from the stated formula ($R_z R_y R_x$), requiring careful verification of the intended rotation convention [S1, unofficial].

## Background
Originated in the `quertzuiop/Volumetric-display` project, an unofficial repository for volumetric rendering [S1, unofficial]. It is used within the `Scene` object to manage transformations for various geometry types including `ParticleGeometry`, `CapsuleGeometry`, `TriangleGeometry`, `SphereGeometry`, `CuboidGeometry`, `MeshGeometry`, and `TextGeometry` [S1, unofficial].

## Sources
[S1] quertzuiop/Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
