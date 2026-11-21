# Quaternion Operations

Also called: Quaternion Conjugate, Quaternion Multiplication, Rotate vector by quaternion

## Overview
Quaternion Operations provide the fundamental arithmetic primitives for manipulating rotations in 3D space using unit quaternions. This method implements the conjugate, multiplication, and vector rotation operations required to transform coordinates in volume rendering pipelines. It differs from [Quaternion Rotation](2302-quaternion-rotation.md) by exposing the underlying algebraic steps (conjugation and multiplication) rather than abstracting them into a single rotation call, and from [Quaternion to SO3 Conversion](2305-quaternion-to-so3-conversion.md) by operating directly on quaternion components rather than converting to rotation matrices.

## How it works
**Inputs:**
- Two quaternions $q_0 = (x_0, y_0, z_0, w_0)$ and $q_1 = (x_1, y_1, z_1, w_1)$, typically represented as 4-element vectors.
- A 3D vector $v = (v_x, v_y, v_z)$ to be rotated.

**Outputs:**
- Conjugate $q^* = (-x, -y, -z, w)$.
- Product $q = q_0 \otimes q_1$.
- Rotated vector $v'$.

**Data Structures:**
Quaternions are stored as 4-component vectors. The layout is $(x, y, z, w)$, where $x, y, z$ are the imaginary components and $w$ is the scalar component.

**Steps:**

1.  **Quaternion Conjugate:**
    Compute the conjugate of a quaternion $q = (x, y, z, w)$ by negating the vector part while keeping the scalar part unchanged:
    $$q^* = (-x, -y, -z, w)$$

2.  **Quaternion Multiplication:**
    Compute the product $q = q_0 \otimes q_1$ component-wise. The scalar component $w$ and vector components $x, y, z$ of the result are calculated as follows:
    $$
    \begin{aligned}
    w &= w_0 w_1 - x_0 x_1 - y_0 y_1 - z_0 z_1 \\
    x &= w_0 x_1 + x_0 w_1 + y_0 z_1 - z_0 y_1 \\
    y &= w_0 y_1 - x_0 z_1 + y_0 w_1 + z_0 x_1 \\
    z &= w_0 z_1 + x_0 y_1 - y_0 x_1 + z_0 w_1
    \end{aligned}
    $$

3.  **Rotate Vector by Quaternion:**
    To rotate a 3D vector $v$ by a unit quaternion $q$:
    a. Treat $v$ as a pure quaternion $q_v = (v_x, v_y, v_z, 0.0)$.
    b. Compute the conjugate $q^*$.
    c. Compute the intermediate product $q_{temp} = q_v \otimes q^*$.
    d. Compute the final result $q_{ret} = q \otimes q_{temp}$.
    e. Extract the vector part $(x, y, z)$ of $q_{ret}$ as the rotated vector. The scalar part should be zero (within floating-point precision).

**Pseudocode:**

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
    Rotates a 3D vector v by a unit quaternion q.
    v = (vx, vy, vz)
    q = (qx, qy, qz, qw)
    Returns the rotated vector.
    """
    # Treat v as a pure quaternion
    qv = (v[0], v[1], v[2], 0.0)
    
    # Compute conjugate of q
    q_conj = quaternion_conjugate(q)
    
    # Compute qv * q_conj
    q_temp = quaternion_multiply(qv, q_conj)
    
    # Compute q * (qv * q_conj)
    q_ret = quaternion_multiply(q, q_temp)
    
    # Return the vector part (x, y, z)
    return (q_ret[0], q_ret[1], q_ret[2])
```

**Parameters:**
- Quaternions should be normalized (unit length) for correct rotation behavior. If $q$ is not a unit quaternion, the rotation will include scaling. [G] Normalization is typically performed by dividing each component by the magnitude $\sqrt{x^2+y^2+z^2+w^2}$.

**Edge Cases and Pitfalls:**
- **Non-unit quaternions:** If $q$ is not normalized, the resulting rotation will be scaled.
- **Precision errors:** Floating-point arithmetic may introduce small errors in the scalar part of the rotated quaternion. It is recommended to re-normalize the result if the quaternion is to be used in subsequent operations.
- **Pure quaternion assumption:** The vector rotation method assumes $v$ is treated as a pure quaternion (scalar part 0).

## Subtopics

### [Inverse SE3 Transformation](3476-inverse-se3-transformation.md)
Computes the inverse of a rigid body transformation (Special Euclidean group SE3) defined by a rotation and a translation. It solves the problem of mapping coordinates from a transformed frame back to the original frame, which is essential for coordinate system conversions in ray generation and camera models. It sets itself apart by providing explicit implementations for both matrix and quaternion representations, ensuring numerical stability through the use of quaternion conjugates for rotation inversion.

### [Index-to-World Clipping Plane Generation](13249-index-to-world-clipping-plane-generation.md)
This method converts voxel-space index bounds (defined by minimum and maximum coordinates along the $i$, $j$, and $k$ axes) into world-space clipping planes compatible with volume rendering pipelines. It solves the problem of aligning rectangular crop regions with the physical orientation of the volume data, ensuring that clipping boundaries respect the image's spatial transformation. It differs from inverse transformation methods by explicitly constructing plane primitives from index bounds rather than transforming existing world-space geometry.

### Quaternion Möbius Transformations
Conformal transformations of 3D space formalized as quaternion Möbius transformations generated by translations, rotations, dilations, reflections, and geometric inversions [S2]. The transformation is defined as $L_H(q) = (aq+b)(cq+d)^{-1}$, where $q, a, b, c, d \in \mathbb{H}$ (quaternions) [S2]. Elementary transformations are represented by $2 \times 2$ quaternionic matrices. Types include: Identity $L(q)=q$, Dilation $L(q)=sq$ ($s \in \mathbb{R}$), Translation $L(q)=q+v$ ($v \in \mathbb{H}_0$), Inversion $L(q)=rq^{-1}$, Reflection $L(q)=-iq$, and Inversion $L(q)=q^{-1}$ [S2]. The composition of two Möbius transformations is also a Möbius transformation, with coefficients obtainable as a matrix product [S2].

## Operating characteristics
- **Time Complexity:** $O(1)$ for all operations (conjugate, multiplication, rotation) as they involve a fixed number of arithmetic operations.
- **Memory Complexity:** $O(1)$ per quaternion/vector.
- **Strengths:** Efficient, numerically stable (for unit quaternions), avoids gimbal lock, compact representation.
- **Limitations:** Requires normalization to maintain unit length; interpolation (slerp) is more complex than with matrices.
- **When to use:** When performing multiple rotations or composing rotations in 3D space.
- **When not to use:** When interfacing with APIs that require rotation matrices (e.g., OpenGL, DirectX) without conversion overhead.

## Background
Quaternion operations are fundamental in computer graphics and robotics for representing 3D rotations. The conjugate and multiplication operations are derived from Hamilton's quaternion algebra. The use of quaternions for rotation avoids the gimbal lock problem associated with Euler angles and is more compact than rotation matrices. The specific implementation details for conjugate and multiplication are standard in graphics programming [S1, unofficial]. Quaternion Möbius transformations extend this algebra to conformal geometry, allowing for more complex spatial transformations [S2].

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S2] Kernel-Reflection Sequences, https://doi.org/10.2312/stag.20161364, fetched 2026-10-26
