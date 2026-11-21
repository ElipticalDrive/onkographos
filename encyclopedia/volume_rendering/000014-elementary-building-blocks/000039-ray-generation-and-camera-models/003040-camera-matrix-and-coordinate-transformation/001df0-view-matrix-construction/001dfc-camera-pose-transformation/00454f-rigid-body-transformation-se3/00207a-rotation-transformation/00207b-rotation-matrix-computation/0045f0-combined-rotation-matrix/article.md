# Combined Rotation Matrix

Also called: Euler Angle Rotation Matrix, XYZ Rotation Matrix

## Overview
Computes a $3 \times 3$ rotation matrix by composing individual rotations around the X, Y, and Z axes in Z-Y-X order [S1, unofficial]. It is specifically designed for volume rendering ray generation, where the resulting matrix components are stored as global variables representing unit steps along transformed axes [S1, unofficial]. It differs from general-purpose rotation libraries by targeting a specific fixed-point or integer-based input pipeline and storing results in a flat global variable namespace rather than a structured matrix object [S1, unofficial].

## How it works
**Inputs:** Three rotation angles in radians: $\phi$ (X-axis), $\theta$ (Y-axis), and $\psi$ (Z-axis) [S1, unofficial].
**Outputs:** A $3 \times 3$ rotation matrix $R$ [S1, unofficial].

**Data Structures:**
The method computes three intermediate $3 \times 3$ matrices ($R_x, R_y, R_z$) and combines them. The final result is a single $3 \times 3$ matrix.

**Steps:**
1. Compute the rotation matrix for the X-axis rotation $\phi$:
   $$
   R_x = \begin{bmatrix}
   1 & 0 & 0 \\
   0 & \cos(\phi) & -\sin(\phi) \\
   0 & \sin(\phi) & \cos(\phi)
   \end{bmatrix}
   $$
2. Compute the rotation matrix for the Y-axis rotation $\theta$:
   $$
   R_y = \begin{bmatrix}
   \cos(\theta) & 0 & \sin(\theta) \\
   0 & 1 & 0 \\
   -\sin(\theta) & 0 & \cos(\theta)
   \end{bmatrix}
   $$
3. Compute the rotation matrix for the Z-axis rotation $\psi$:
   $$
   R_z = \begin{bmatrix}
   \cos(\psi) & -\sin(\psi) & 0 \\
   \sin(\psi) & \cos(\psi) & 0 \\
   0 & 0 & 1
   \end{bmatrix}
   $$
4. Combine the matrices by multiplying them in Z-Y-X order:
   $$
   R = R_z \times R_y \times R_x
   $$
   where $\times$ denotes standard matrix multiplication [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def rotation_matrix_x(phi):
    c = math.cos(phi)
    s = math.sin(phi)
    return [
        [1.0, 0.0, 0.0],
        [0.0, c, -s],
        [0.0, s, c]
    ]

def rotation_matrix_y(theta):
    c = math.cos(theta)
    s = math.sin(theta)
    return [
        [c, 0.0, s],
        [0.0, 1.0, 0.0],
        [-s, 0.0, c]
    ]

def rotation_matrix_z(psi):
    c = math.cos(psi)
    s = math.sin(psi)
    return [
        [c, -s, 0.0],
        [s, c, 0.0],
        [0.0, 0.0, 1.0]
    ]

def multiply_matrices(A, B):
    """Multiply two 3x3 matrices A and B."""
    C = [[0.0, 0.0, 0.0], [0.0, 0.0, 0.0], [0.0, 0.0, 0.0]]
    for i in range(3):
        for j in range(3):
            sum_val = 0.0
            for k in range(3):
                sum_val += A[i][k] * B[k][j]
            C[i][j] = sum_val
    return C

def combined_rotation_matrix(phi, theta, psi):
    """
    Combine rotations around X, Y, and Z axes.
    :param phi: Rotation angle around the X-axis (in radians).
    :param theta: Rotation angle around the Y-axis (in radians).
    :param psi: Rotation angle around the Z-axis (in radians).
    :return: Combined rotation matrix as a 3x3 list of lists.
    """
    rx = rotation_matrix_x(phi)
    ry = rotation_matrix_y(theta)
    rz = rotation_matrix_z(psi)
    
    # Multiply in Z-Y-X order: Rz * Ry * Rx
    temp = multiply_matrices(ry, rx)
    result = multiply_matrices(rz, temp)
    return result
```

**Parameters:**
- $\phi$: X-axis rotation angle. Typical values depend on camera orientation.
- $\theta$: Y-axis rotation angle. Typical values depend on camera orientation.
- $\psi$: Z-axis rotation angle. Typical values depend on camera orientation.

**Edge Cases:**
- Angles should be in radians [S1, unofficial].
- Numerical precision issues may arise if angles are very large; normalization to $[-\pi, \pi]$ is recommended [G].

## Subtopics

### [Euler Angle Rotation Matrix Computation](20420-euler-angle-rotation-matrix-computation.md)
This method computes the $3 \times 3$ rotation matrix corresponding to a sequence of rotations about the Z, Y, and X axes (Z-Y-X Euler angles) to transform unit vectors from object space to display space. It is specifically designed for volume rendering ray generation, where the resulting matrix components are stored as global variables representing the unit steps along the transformed axes. It differs from general-purpose rotation libraries by targeting a specific fixed-point or integer-based input pipeline and storing results in a flat global variable namespace rather than a structured matrix object.

## Operating characteristics
Time complexity is $O(1)$ as it involves a fixed number of trigonometric operations and matrix multiplications. Memory complexity is $O(1)$, storing only a few temporary matrices. Performance is suitable for real-time ray generation in volume rendering. Strengths include simplicity and direct mapping to Euler angles. Limitations include potential gimbal lock if not handled carefully, though the specific implementation in [S1, unofficial] does not explicitly address this. Use when Euler angles are the natural input format for camera pose. Avoid when quaternions are preferred for interpolation or stability.

## Background
Originated in the context of direct volume rendering implementations, specifically in the `Direct_Volume_Rendering-DVR-` repository [S1, unofficial]. It is used to transform ray directions from camera space to world space or vice versa during ray casting.

## Sources
[S1] asheibanifard/Direct_Volume_Rendering-DVR-, https://github.com/asheibanifard/Direct_Volume_Rendering-DVR-, fetched 2026-10-26 (unofficial)
