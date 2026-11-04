# Quaternion to SO3 Conversion

Also called: Quaternion to Rotation Matrix, Quaternion to Rotation Matrix Conversion, qvec2rotmat

## Overview
Quaternion to SO3 Conversion transforms a quaternion representation into a 3x3 rotation matrix $R \in SO(3)$, or vice versa, enabling the use of quaternions for compact rotation storage and matrices for linear algebra operations such as matrix multiplication and transformation composition [S1, S4, unofficial]. This method is distinct from [Quaternion Rotation](2302-quaternion-rotation.md), which applies a rotation to a vector, by providing the explicit basis transformation matrix [G]. It differs from [Quaternion Operations](3474-quaternion-operations.md) by focusing specifically on the structural conversion between the two mathematical representations rather than algebraic manipulation of quaternions themselves [G].

## How it works
The conversion can proceed in two directions: Quaternion to Rotation Matrix ($q \to R$) and Rotation Matrix to Quaternion ($R \to q$) [S4, unofficial]. Both directions require handling numerical stability and sign ambiguity [G].

### Inputs and Outputs
- **Quaternion to Matrix**:
  - Input: A quaternion $q$ represented as a vector of 4 floats. Sources use different conventions: $[x, y, z, w]$ (scalar last) in [S3, unofficial], and $[w, x, y, z]$ (scalar first) in [S2, S4, S5, unofficial].
  - Output: A rotation matrix $R$ of shape $(3, 3)$ or batch $(N, 3, 3)$ [S2, S3, unofficial].
- **Matrix to Quaternion**:
  - Input: A 3x3 rotation matrix $R$ [S3, unofficial].
  - Output: A quaternion $q$ [S3, unofficial].

### Data Structures
- **Quaternion**: 4-component vector. The implementation must adhere to the specific convention of the surrounding system [G]. The formulas below assume $q = [x, y, z, w]$ (scalar last) unless noted [G].
- **Rotation Matrix**: 3x3 orthogonal matrix with determinant +1 [G].

### Quaternion to Rotation Matrix ($q \to R$)
This direction is direct and numerically stable if the input quaternion is normalized [G].

1. **Normalization**: Ensure the input quaternion $q$ is a unit quaternion. If not, normalize it:
   $$ q_{norm} = \frac{q}{\|q\|_2} $$
   Let $q = [x, y, z, w]$ where $w$ is the scalar part [S2, S3, S5, unofficial].
2. **Compute Matrix Elements**: Calculate the 9 elements of $R$ using the following formulas [S2, S3, S5, unofficial]:
   $$
   R = \begin{bmatrix}
   1 - 2(y^2 + z^2) & 2(xy - zw) & 2(xz + yw) \\
   2(xy + zw) & 1 - 2(x^2 + z^2) & 2(yz - xw) \\
   2(xz - yw) & 2(yz + xw) & 1 - 2(x^2 + y^2)
   \end{bmatrix}
   $$
   Note: Some sources use $[q_0, q_1, q_2, q_3]$ where $q_0=w, q_1=x, q_2=y, q_3=z$ [S4, S5, unofficial]. The formula above maps $x,y,z$ to the vector components and $w$ to the scalar [G].
3. **Batch Processing**: For a batch of quaternions, compute the outer product $q \otimes q$ (element-wise multiplication of components) and apply the formulas vectorized [S3, unofficial].

### Rotation Matrix to Quaternion ($R \to q$)
This direction requires handling numerical instability near singularities (e.g., when the trace is near zero or a diagonal element is dominant) [G]. The standard approach uses the trace of $R$ to determine the largest component [S3, unofficial].

1. **Compute Trace**:
   $$ T = R_{00} + R_{11} + R_{22} $$
2. **Determine Dominant Component**:
   Identify which component ($w, x, y, z$) has the largest magnitude to avoid division by small numbers. The conditions are [S3, unofficial]:
   - If $T > 0$: $w$ is largest.
   - Else if $R_{00} > R_{11}$ and $R_{00} > R_{22}$: $x$ is largest.
   - Else if $R_{11} > R_{22}$: $y$ is largest.
   - Else: $z$ is largest.
3. **Compute Quaternion Components**:
   Depending on the dominant component, use the corresponding case [S3, unofficial]:
   - **Case 1: $T > 0$ (w is largest)**
     $$ S = \frac{0.5}{\sqrt{1 + T}} $$
     $$ w = 0.25 / S $$
     $$ x = (R_{21} - R_{12}) \cdot S $$
     $$ y = (R_{02} - R_{20}) \cdot S $$
     $$ z = (R_{10} - R_{01}) \cdot S $$
   - **Case 2: $R_{00}$ is max (x is largest)**
     $$ S = 2 \sqrt{1 + R_{00} - R_{11} - R_{22}} $$
     $$ x = 0.25 \cdot S $$
     $$ y = (R_{01} + R_{10}) / S $$
     $$ z = (R_{02} + R_{20}) / S $$
     $$ w = (R_{21} - R_{12}) / S $$
   - **Case 3: $R_{11}$ is max (y is largest)**
     $$ S = 2 \sqrt{1 + R_{11} - R_{00} - R_{22}} $$
     $$ x = (R_{01} + R_{10}) / S $$
     $$ y = 0.25 \cdot S $$
     $$ z = (R_{12} + R_{21}) / S $$
     $$ w = (R_{02} - R_{20}) / S $$
   - **Case 4: $R_{22}$ is max (z is largest)**
     $$ S = 2 \sqrt{1 + R_{22} - R_{00} - R_{11}} $$
     $$ x = (R_{02} + R_{20}) / S $$
     $$ y = (R_{12} + R_{21}) / S $$
     $$ z = 0.25 \cdot S $$
     $$ w = (R_{10} - R_{01}) / S $$
4. **Sign Ambiguity Handling**: Quaternions $q$ and $-q$ represent the same rotation. Implementations may enforce a canonical sign convention, such as ensuring $w \ge 0$. If the resulting $w < 0$, negate the entire quaternion $q = -q$ [S4, unofficial].

### Pseudocode (Quaternion to Matrix)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quaternion_to_rotation_matrix(q):
    """
    Converts a quaternion [x, y, z, w] to a 3x3 rotation matrix.
    Assumes q is normalized.
    q: list/tuple of 4 floats
    Returns R: list of 3 lists, each with 3 floats
    """
    x, y, z, w = q
    x2, y2, z2, w2 = x*x, y*y, z*z, w*w
    xy, xz, xw = x*y, x*z, x*w
    yz, yw, zw = y*z, y*w, z*w
    return [
        [1.0 - 2.0*(y2 + z2), 2.0*(xy - zw), 2.0*(xz + yw)],
        [2.0*(xy + zw), 1.0 - 2.0*(x2 + z2), 2.0*(yz - xw)],
        [2.0*(xz - yw), 2.0*(yz + xw), 1.0 - 2.0*(x2 + y2)]
    ]
```

### Pseudocode (Matrix to Quaternion)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rotation_matrix_to_quaternion(R):
    """
    Converts a 3x3 rotation matrix R to a quaternion [x, y, z, w].
    R: list of 3 lists, each with 3 floats
    Returns q: list of 4 floats
    """
    r00, r01, r02 = R[0]
    r10, r11, r12 = R[1]
    r20, r21, r22 = R[2]
    
    trace = r00 + r11 + r22
    
    if trace > 0:
        S = 0.5 / (1.0 + trace)**0.5
        w = 0.25 / S
        x = (r21 - r12) * S
        y = (r02 - r20) * S
        z = (r10 - r01) * S
    elif r00 > r11 and r00 > r22:
        S = 2.0 * (1.0 + r00 - r11 - r22)**0.5
        x = 0.25 * S
        y = (r01 + r10) / S
        z = (r02 + r20) / S
        w = (r21 - r12) / S
    elif r11 > r22:
        S = 2.0 * (1.0 + r11 - r00 - r22)**0.5
        x = (r01 + r10) / S
        y = 0.25 * S
        z = (r12 + r21) / S
        w = (r02 - r20) / S
    else:
        S = 2.0 * (1.0 + r22 - r00 - r11)**0.5
        x = (r02 + r20) / S
        y = (r12 + r21) / S
        z = 0.25 * S
        w = (r10 - r01) / S
        
    # Handle sign ambiguity: ensure w >= 0
    if w < 0:
        x, y, z, w = -x, -y, -z, -w
        
    return [x, y, z, w]
```

### Parameters
- **Normalization**: Input quaternions should be unit quaternions. If not, normalization is required [S2, S3, S5, unofficial].
- **Convention**: The order of components $[x, y, z, w]$ vs $[w, x, y, z]$ must be consistent. The formulas above assume $[x, y, z, w]$ (scalar last) [S2, S3, S5, unofficial]. If using $[w, x, y, z]$, adjust indices accordingly [S4, unofficial].

### Edge Cases and Pitfalls
- **Numerical Instability**: When the trace is near zero or a diagonal element is near zero, direct division can cause instability. The case-based approach using the largest component avoids this [S3, unofficial].
- **Sign Ambiguity**: $q$ and $-q$ represent the same rotation. Implementations may enforce a canonical sign (e.g., $w \ge 0$) to ensure uniqueness [S4, unofficial].
- **Non-Rotation Matrices**: If the input matrix is not a valid rotation matrix (e.g., due to numerical error or non-orthogonality), the output quaternion may not be a unit quaternion. Normalization of the output quaternion is recommended [S3, unofficial].
- **Batch Processing**: Ensure broadcasting rules are handled correctly for batched inputs [S2, S3, S5, unofficial].

## Subtopics

### SE3 to Quaternion and Translation
Extracts the quaternion and translation vector from a 4x4 SE3 transformation matrix.
- **Definition**: Decomposes an SE3 matrix into its rotational (quaternion) and translational (vector) components [S3, unofficial].
- **Steps**:
  1. Extract the top-left 3x3 submatrix $R$ and the top-right 3x1 column vector $t$ from the 4x4 SE3 matrix [S3, unofficial].
  2. Convert $R$ to a quaternion using the `rotation_matrix_to_quaternion` method described above [S3, unofficial].
  3. Return the quaternion $q$ and translation vector $t$ [S3, unofficial].

## Operating characteristics
- **Time Complexity**:
  - $q \to R$: $O(1)$ for a single quaternion, $O(N)$ for a batch of size $N$. Involves a fixed number of multiplications and additions [G].
  - $R \to q$: $O(1)$ for a single matrix, $O(N)$ for a batch. Involves square roots and conditional branching [G].
- **Memory Complexity**: $O(1)$ for single conversions, $O(N)$ for batched conversions to store the output matrix or quaternion [G].
- **Measured Performance**: Not given in the sources.
- **Strengths**:
  - $q \to R$ is computationally cheap and numerically stable [G].
  - $R \to q$ using the trace-based method is robust against numerical instability near singularities [S3, unofficial].
- **Limitations**:
  - Sign ambiguity requires additional handling if a canonical representation is needed [S4, unofficial].
- **When to use**: Use $q \to R$ when matrix operations (e.g., composition with other transformations) are required. Use $R \to q$ when compact storage or interpolation (e.g., SLERP) is needed [G].

## Background
- **Usage**: Commonly used in camera pose estimation, 3D reconstruction, and robotics for converting between rotation representations [S3, S4, unofficial]. The implementation details in the sources reflect modern deep learning frameworks (PyTorch) used in recent 3D Gaussian Splatting research [S3, S5, unofficial].

## Sources
[S1] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S2] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S3] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S4] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S5] https://github.com/WangFeng18/3d-gaussian-splatting, https://github.com/WangFeng18/3d-gaussian-splatting, fetched 2026-10-30 (unofficial)
