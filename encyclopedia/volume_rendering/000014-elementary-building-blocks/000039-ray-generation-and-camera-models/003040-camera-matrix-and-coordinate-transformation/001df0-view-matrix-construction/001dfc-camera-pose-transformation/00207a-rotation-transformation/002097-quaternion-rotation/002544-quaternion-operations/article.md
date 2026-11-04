# Quaternion Operations

Also called: Quaternion Conjugate, Quaternion Multiplication, Rotate vector by quaternion

## Overview
Defines conjugation, multiplication, and vector rotation for quaternions in $(x, y, z, w)$ layout [S1, unofficial]. These operations form the computational core for [Quaternion Rotation](2302-quaternion-rotation.md) in volume rendering pipelines [G].

## How it works

### Inputs and Outputs
*   **Input**: Two quaternions $q_0 = (x_0, y_0, z_0, w_0)$ and $q_1 = (x_1, y_1, z_1, w_1)$, and optionally a 3D vector $v = (v_x, v_y, v_z)$ for rotation [S1, unofficial].
*   **Output**: A resulting quaternion $q_{res}$ or a rotated 3D vector $v'$ [S1, unofficial].

### Data Structures
Quaternions are stored as 4-component vectors in $(x, y, z, w)$ layout, where $x, y, z$ form the vector part and $w$ is the scalar part [S1, unofficial].

### Steps
1.  Compute the conjugate of a quaternion by negating the vector components while keeping the scalar component unchanged.
2.  Multiply two quaternions component-wise using the standard Hamilton product rules.
3.  Rotate a 3D vector by treating it as a pure quaternion, computing $q \otimes (q_v \otimes q^*)$, and extracting the resulting vector part.

### Formulas

**1. Quaternion Conjugate**
The conjugate of a quaternion $q = (x, y, z, w)$ is defined as [S1, unofficial]:
$$q^* = (-x, -y, -z, w)$$

**2. Quaternion Multiplication**
The product $q = q_0 \otimes q_1$ is computed component-wise [S1, unofficial]:
$$
\begin{aligned}
x &= w_0 x_1 + x_0 w_1 + y_0 z_1 - z_0 y_1 \\
y &= w_0 y_1 - x_0 z_1 + y_0 w_1 + z_0 x_1 \\
z &= w_0 z_1 + x_0 y_1 - y_0 x_1 + z_0 w_1 \\
w &= w_0 w_1 - x_0 x_1 - y_0 y_1 - z_0 z_1
\end{aligned}
$$

**3. Rotate Vector by Quaternion**
To rotate a 3D vector $v$ by a unit quaternion $q$:
1.  Treat $v$ as a pure quaternion $q_v = (v_x, v_y, v_z, 0.0)$ [S1, unofficial].
2.  Compute the intermediate quaternion: $q_{temp} = q \otimes (q_v \otimes q^*)$ [S1, unofficial].
3.  Extract the vector part $(x, y, z)$ of $q_{temp}$ as the result $v'$ [S1, unofficial].

### Parameters and Edge Cases
*   **Normalization**: The rotation formula assumes $q$ is a unit quaternion ($|q|=1$); if $q$ is not normalized, the result will be scaled incorrectly [G].
*   **Precision**: Floating-point errors can accumulate during repeated multiplications, requiring periodic re-normalization [G].
*   **Order of Multiplication**: Quaternion multiplication is non-commutative ($q_0 \otimes q_1 \neq q_1 \otimes q_0$), so the operand order in the formula must be preserved [G].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quaternion_conjugate(q):
    """
    Computes the conjugate of a quaternion q = (x, y, z, w).
    Returns (-x, -y, -z, w).
    """
    x, y, z, w = q
    return (-x, -y, -z, w)

def quaternion_multiply(q0, q1):
    """
    Multiplies two quaternions q0 and q1.
    q0 = (x0, y0, z0, w0), q1 = (x1, y1, z1, w1)
    Returns q0 * q1.
    """
    x0, y0, z0, w0 = q0
    x1, y1, z1, w1 = q1
    
    x = w0 * x1 + x0 * w1 + y0 * z1 - z0 * y1
    y = w0 * y1 - x0 * z1 + y0 * w1 + z0 * x1
    z = w0 * z1 + x0 * y1 - y0 * x1 + z0 * w1
    w = w0 * w1 - x0 * x1 - y0 * y1 - z0 * z1
    
    return (x, y, z, w)

def rotate_vector_by_quaternion(v, q):
    """
    Rotates vector v by quaternion q.
    v = (vx, vy, vz)
    q = (qx, qy, qz, qw)
    Returns rotated vector v'.
    """
    # Treat v as pure quaternion
    qv = (v[0], v[1], v[2], 0.0)
    
    # Compute conjugate of q
    q_conj = quaternion_conjugate(q)
    
    # Compute q * (qv * q_conj)
    q_temp = quaternion_multiply(q, quaternion_multiply(qv, q_conj))
    
    # Return vector part
    return (q_temp[0], q_temp[1], q_temp[2])
```

## Subtopics

### [Inverse SE3 Transformation](3476-inverse-se3-transformation.md)
Computes the inverse of a rigid body transformation (Special Euclidean group SE3) defined by a rotation quaternion $q$ and a translation vector $t$, or equivalently by a $4 \times 4$ homogeneous transformation matrix. This operation is essential for converting camera poses from world-to-camera coordinates to camera-to-world coordinates, or vice versa, during ray generation and view matrix construction in volume rendering and 3D Gaussian Splatting. It is distinguished from general matrix inversion by exploiting the orthogonality of the rotation component, reducing computational cost from $O(n^3)$ to $O(1)$ with simple vector operations.

## Operating characteristics
*   **Time Complexity**: $O(1)$ for all operations, requiring 16 multiplications and 12 additions per multiplication [S1, unofficial].
*   **Memory Complexity**: $O(1)$ for storing the 4-component quaternion and intermediates [S1, unofficial].
*   **Measured Performance**: Not given in the sources [S1, unofficial].
*   **Scaling**: Not given in the sources.
*   **Strengths**: Computationally efficient compared to matrix operations. Avoids gimbal lock. Compact representation (4 floats vs 9 or 16) [G].
*   **Limitations**: Interpolation requires additional logic. Non-commutativity requires careful ordering [G]. Not as intuitive for visualizing rotation axes as Euler angles [G].
*   **When to Use**: When composing multiple rotations or rotating many vectors efficiently [G]. Essential for [Quaternion Rotation](2302-quaternion-rotation.md) [G].
*   **When Not to Use**: When explicit rotation matrices are required for compatibility with other graphics APIs or when interpolating between rotations (use Slerp instead) [G].

## Background
Not given in the sources.

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
