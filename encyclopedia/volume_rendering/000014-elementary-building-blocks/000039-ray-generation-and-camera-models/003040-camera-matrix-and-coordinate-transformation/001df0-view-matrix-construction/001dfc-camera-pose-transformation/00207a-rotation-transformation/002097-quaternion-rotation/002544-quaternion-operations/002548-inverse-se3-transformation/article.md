# Inverse SE3 Transformation
Also called: Inverse Rigid Body Transformation, SE3 Inverse, Taichi Inverse SE3

## Overview
Computes the inverse of an SE3 transformation represented by a unit quaternion $q$ and a translation vector $t$, or equivalently by a $4 \times 4$ homogeneous transformation matrix [S1, unofficial]. The inverse rotation is the conjugate quaternion $q^*$, and the inverse translation is $-q^* \otimes t$ [S1, unofficial].

## How it works
### Inputs and Outputs
- **Input (Quaternion form):** A unit quaternion $q = [w, x, y, z]$ representing rotation, and a 3D translation vector $t = [t_x, t_y, t_z]$ [S1, unofficial].
- **Input (Matrix form):** A $4 \times 4$ homogeneous transformation matrix $T$ [S1, unofficial].
- **Output (Quaternion form):** Inverse quaternion $q_{inv}$ and inverse translation $t_{inv}$ [S1, unofficial].
- **Output (Matrix form):** Inverse $4 \times 4$ matrix $T^{-1}$ [S1, unofficial].

### Data Structures
- **Quaternion:** 4 floating-point values $[w, x, y, z]$ [S1, unofficial].
- **Vector:** 3 floating-point values $[x, y, z]$ [S1, unofficial].
- **Matrix:** $4 \times 4$ grid of floating-point values [S1, unofficial].

### Formulas
For quaternion representation:
1. **Inverse Rotation:** The inverse of a rotation represented by unit quaternion $q$ is its conjugate $q^*$ [S1, unofficial].
    $$ q_{inv} = q^* = [w, -x, -y, -z] $$
2. **Inverse Translation:** The inverse translation is the negative of the original translation vector rotated by the inverse rotation [S1, unofficial].
    $$ t_{inv} = -\text{rotate}(q^*, t) $$
    Where $\text{rotate}(q, v)$ applies the rotation $q$ to vector $v$ [S1, unofficial].

For matrix representation:
Given $T = \begin{bmatrix} R & t \\ 0 & 1 \end{bmatrix}$, where $R$ is the $3 \times 3$ rotation submatrix and $t$ is the $3 \times 1$ translation column:
1. **Inverse Rotation:** $R^T$ (transpose of $R$) [S1, unofficial].
2. **Inverse Translation:** $-R^T t$ [S1, unofficial].
    $$ T^{-1} = \begin{bmatrix} R^T & -R^T t \\ 0 & 1 \end{bmatrix} $$

### Pseudocode (Quaternion Form)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from typing import List, Tuple

def inverse_se3_quaternion(q: List[float], t: List[float]) -> Tuple[List[float], List[float]]:
    """
    Computes the inverse SE3 transformation from quaternion q and translation t.
    q = [w, x, y, z], t = [tx, ty, tz]
    """
    # Step 1: Compute conjugate quaternion
    q_inv = [q[0], -q[1], -q[2], -q[3]]
    
    # Step 2: Rotate the original translation vector by the inverse rotation
    t_rotated = quaternion_rotate_vector(q_inv, t)
    
    # Step 3: Negate the rotated translation
    t_inv = [-t_rotated[0], -t_rotated[1], -t_rotated[2]]
    
    return q_inv, t_inv

def quaternion_rotate_vector(q: List[float], v: List[float]) -> List[float]:
    """
    Rotates vector v by quaternion q.
    Assumes q is normalized.
    Formula: v' = v + 2 * cross(q.xyz, cross(q.xyz, v) + q.w * v)
    """
    qx, qy, qz, qw = q[1], q[2], q[3], q[0]
    vx, vy, vz = v[0], v[1], v[2]
    
    # Cross product of q.xyz and v
    cx = qy * vz - qz * vy
    cy = qz * vx - qx * vz
    cz = qx * vy - qy * vx
    
    # Cross product of q.xyz and the result above
    ux = qy * cz - qz * cy
    uy = qz * cx - qx * cz
    uz = qx * cy - qy * cx
    
    # Inner term: cross(q.xyz, v) + qw * v
    ix = cx + qw * vx
    iy = cy + qw * vy
    iz = cz + qw * vz
    
    # Final rotation: v + 2 * cross(q.xyz, inner)
    fx = qy * iz - qz * iy
    fy = qz * ix - qx * iz
    fz = qx * iy - qy * ix
    
    return [vx + 2 * fx, vy + 2 * fy, vz + 2 * fz]
```

### Pseudocode (Matrix Form)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from typing import List

def inverse_se3_matrix(T: List[List[float]]) -> List[List[float]]:
    """
    Computes the inverse of a 4x4 SE3 transformation matrix.
    T is assumed to be in the form:
    [ R00 R01 R02 Tx ]
    [ R10 R11 R12 Ty ]
    [ R20 R21 R22 Tz ]
    [  0   0   0   1 ]
    """
    # Extract R_T (transpose of upper-left 3x3)
    R_T = [
        [T[0][0], T[1][0], T[2][0]],
        [T[0][1], T[1][1], T[2][1]],
        [T[0][2], T[1][2], T[2][2]]
    ]
    
    # Extract translation vector t from T (first 3 elements of 4th column)
    t = [T[0][3], T[1][3], T[2][3]]
    
    # Compute inverse translation: -R_T @ t
    inv_t = [
        -(R_T[0][0]*t[0] + R_T[0][1]*t[1] + R_T[0][2]*t[2]),
        -(R_T[1][0]*t[0] + R_T[1][1]*t[1] + R_T[1][2]*t[2]),
        -(R_T[2][0]*t[0] + R_T[2][1]*t[1] + R_T[2][2]*t[2])
    ]
    
    # Construct the inverse matrix
    return [
        [R_T[0][0], R_T[0][1], R_T[0][2], inv_t[0]],
        [R_T[1][0], R_T[1][1], R_T[1][2], inv_t[1]],
        [R_T[2][0], R_T[2][1], R_T[2][2], inv_t[2]],
        [0.0, 0.0, 0.0, 1.0]
    ]
```

### Parameters and Edge Cases
- **Normalization:** The input quaternion $q$ must be normalized for correct rotation [G].
- **Precision:** Floating-point errors can accumulate over repeated operations [G].
- **Degenerate Input:** If $T$ is not a valid rigid body transformation (e.g., $R$ is not orthogonal), the result is geometrically incorrect [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ for both forms [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space [G].
- **Performance:** Highly parallelizable in GPU kernels (e.g., Taichi) with low latency [S1, unofficial].
- **Strengths:** Numerically stable for rigid bodies; computationally cheap [G].
- **Limitations:** Only applicable to rigid transformations (SE3); does not handle scaling or shearing [G].

## Background
The SE3 group represents rigid body motions in 3D space [G]. The inverse operation is fundamental in robotics, computer graphics, and computer vision [G]. In the context of 3D Gaussian Splatting and volume rendering, it transforms points between world and camera coordinate systems [S1, unofficial]. The specific implementation details are from the Taichi 3D Gaussian Splatting project [S1, unofficial].

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
