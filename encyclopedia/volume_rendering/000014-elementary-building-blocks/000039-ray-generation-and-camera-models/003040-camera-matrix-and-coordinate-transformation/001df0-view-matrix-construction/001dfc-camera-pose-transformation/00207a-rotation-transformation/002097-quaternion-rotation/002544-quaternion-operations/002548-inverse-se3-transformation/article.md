# Inverse SE3 Transformation

Also called: Inverse Rigid Body Transformation, SE3 Inverse, Taichi Inverse SE3

## Overview
Computes the inverse of a rigid body transformation (Special Euclidean group SE3) defined by a rotation and a translation. It solves the problem of mapping coordinates from a transformed frame back to the original frame, which is essential for coordinate system conversions in ray generation and camera models. It sets itself apart by providing explicit implementations for both matrix and quaternion representations, ensuring numerical stability through the use of quaternion conjugates for rotation inversion [S1, unofficial].

## How it works
**Inputs:**
1.  **Matrix Form:** A $4 \times 4$ homogeneous transformation matrix $T$.
2.  **Quaternion Form:** A unit quaternion $q$ representing rotation and a 3D vector $t$ representing translation.

**Outputs:**
1.  **Matrix Form:** The inverse $4 \times 4$ matrix $T^{-1}$.
2.  **Quaternion Form:** The inverse quaternion $q_{inv}$ and inverse translation vector $t_{inv}$.

**Data Structures:**
*   **Matrix:** $4 \times 4$ floating-point array. Layout:
    $$
    T = \begin{bmatrix}
    R_{00} & R_{01} & R_{02} & t_x \\
    R_{10} & R_{11} & R_{12} & t_y \\
    R_{20} & R_{21} & R_{22} & t_z \\
    0 & 0 & 0 & 1
    \end{bmatrix}
    $$
    Where $R$ is the $3 \times 3$ rotation submatrix and $t$ is the $3 \times 1$ translation vector.
*   **Quaternion:** $q = [w, x, y, z]$ (scalar-first) or $[x, y, z, w]$ (scalar-last). The implementation assumes standard conjugate definition.
*   **Vector:** $t = [t_x, t_y, t_z]$.

**Steps:**

1.  **Extract Components (Matrix Input):**
    Extract the rotation matrix $R$ from the top-left $3 \times 3$ elements and translation $t$ from the top-right $3 \times 1$ elements of $T$.
    $$
    R = T[0:3, 0:3], \quad t = T[0:3, 3]
    $$

2.  **Compute Inverse Rotation:**
    The inverse of a rotation matrix is its transpose.
    $$
    R_{inv} = R^T
    $$
    For quaternion input, the inverse rotation is the conjugate quaternion $q^*$.
    $$
    q_{inv} = q^* = [w, -x, -y, -z] \quad (\text{if } q=[w,x,y,z])
    $$

3.  **Compute Inverse Translation:**
    The inverse translation is the negative of the original translation vector rotated by the inverse rotation.
    $$
    t_{inv} = -R_{inv} t = -R^T t
    $$
    For quaternion input, apply the inverse quaternion rotation to $t$:
    $$
    t_{inv} = -\text{rotate}(q_{inv}, t)
    $$
    Where $\text{rotate}(q, v)$ transforms vector $v$ by quaternion $q$.

4.  **Construct Output (Matrix Input):**
    Assemble $T^{-1}$ using $R_{inv}$ and $t_{inv}$.
    $$
    T^{-1} = \begin{bmatrix}
    R_{inv} & t_{inv} \\
    0 & 1
    \end{bmatrix}
    $$

**Pseudocode (Matrix Input):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_se3_matrix(T):
    # T is a 4x4 matrix
    # Extract R (3x3) and t (3x1)
    R = T[0:3, 0:3]
    t = T[0:3, 3]
    
    # Compute inverse rotation (transpose)
    R_inv = R.T
    
    # Compute inverse translation: -R_inv @ t
    t_inv = -R_inv @ t
    
    # Construct inverse matrix
    T_inv = zeros_like(T)
    T_inv[0:3, 0:3] = R_inv
    T_inv[0:3, 3] = t_inv
    T_inv[3, 3] = 1.0
    
    return T_inv
```

**Pseudocode (Quaternion Input):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_se3_quaternion(q, t):
    # q is a quaternion [w, x, y, z]
    # t is a vector [x, y, z]
    
    # Compute inverse quaternion (conjugate)
    q_inv = quaternion_conjugate(q)
    
    # Compute inverse translation: -rotate(q_inv, t)
    t_inv = -quaternion_rotate(q_inv, t)
    
    return q_inv, t_inv
```

**Parameters:**
*   None. The method is deterministic.

**Edge Cases and Pitfalls:**
*   **Non-unit Quaternion:** If $q$ is not normalized, $q^*$ is not the true inverse rotation. The input quaternion must be normalized before conjugation [G].
*   **Precision:** Floating-point errors in $R^T$ may cause slight non-orthogonality. Re-orthonormalization is not performed in the basic inverse but may be required in iterative applications [G].
*   **Homogeneous Coordinate:** The bottom row $[0, 0, 0, 1]$ must be preserved in the matrix output.

## Subtopics

### Quaternion Rotation
Applies a quaternion to a vector. Used to compute $t_{inv}$ in the quaternion form. [Quaternion Rotation](3473-quaternion-rotation.md)

### Quaternion Conjugate
Computes the conjugate of a quaternion. Used to compute $q_{inv}$. [Quaternion Operations](3474-quaternion-operations.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$. Matrix transpose and vector multiplication are constant time for fixed-size $3 \times 3$ and $3 \times 1$ structures.
*   **Memory Complexity:** $O(1)$. Requires storage for one $4 \times 4$ matrix or one quaternion and one vector.
*   **Strengths:** Numerically stable for rotations (using conjugate avoids matrix inversion). Efficient for GPU parallel execution (Taichi implementation).
*   **Limitations:** Assumes valid SE3 input (orthogonal rotation matrix or unit quaternion). Does not handle scaling or shearing.
*   **When to use:** When converting points from world/camera space to local/object space or vice versa in ray marching or rasterization pipelines.

## Background
The SE3 inverse is a fundamental operation in computer graphics and robotics for coordinate transformation. The specific implementation details, including the Taichi kernel syntax and the use of `torchImage2tiImage` for data transfer, originate from the 3D Gaussian Splatting visualization project by wanmeihuali [S1, unofficial]. This project adapts 3D Gaussian Splatting for real-time visualization using the Taichi programming language, requiring efficient GPU-accelerated coordinate transformations for camera pose updates.

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
