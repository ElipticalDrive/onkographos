# Quaternion to SO3 Conversion

Also called: Hamilton quaternion rotation matrix, Quaternion to Rotation Matrix, Quaternion to Rotation Matrix Conversion, qvec2rotmat

## Overview
This method converts a 4D unit quaternion $q = (w, x, y, z)$ into a $3 \times 3$ rotation matrix $R \in SO(3)$ [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S6, unofficial][S8, unofficial][S9, unofficial]. It solves the problem of representing a 3D rotation in a format compatible with standard linear algebra operations, such as transforming camera poses or rotating 3D points in volume rendering pipelines [S7, unofficial][S8, unofficial]. It sets itself apart from [Quaternion Operations](3474-quaternion-operations.md) by directly computing the orthogonal matrix elements without intermediate cross products [S12, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A quaternion $q$ represented as a 4D vector. The sources use two common orderings: $(w, x, y, z)$ [S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S6, unofficial][S8, unofficial][S9, unofficial] or $(x, y, z, w)$ [S6, unofficial][S7, unofficial]. The method assumes the Hamilton convention [S8, unofficial]. The input quaternion must be normalized to unit length; if not, it is normalized first [S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S6, unofficial][S9, unofficial].
- Output: A $3 \times 3$ rotation matrix $R$ with shape $(3, 3)$ for a single quaternion, or $(N, 3, 3)$ for a batch of $N$ quaternions [S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S9, unofficial].

**Data Structures:**
- Quaternion $q$: A tensor of shape $(N, 4)$ or $(4,)$ [S10, unofficial].
- Rotation Matrix $R$: A tensor of shape $(N, 3, 3)$ or $(3, 3)$ [S10, unofficial].

**Steps:**
1. **Normalization:** If the input quaternion $q$ is not guaranteed to be a unit vector, compute its L2 norm and divide $q$ by this norm [S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S6, unofficial][S9, unofficial].
   $$ \text{norm} = \sqrt{q_w^2 + q_x^2 + q_y^2 + q_z^2} $$
   $$ q \leftarrow q / \text{norm} $$
2. **Unpacking:** Extract the scalar and vector components. Let $w$ be the scalar part and $(x, y, z)$ be the vector part [S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S8, unofficial][S9, unofficial].
   - If input is $(w, x, y, z)$: $w = q[0], x = q[1], y = q[2], z = q[3]$ [S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S8, unofficial][S9, unofficial].
   - If input is $(x, y, z, w)$: $x = q[0], y = q[1], z = q[2], w = q[3]$ [S6, unofficial][S7, unofficial].
3. **Matrix Construction:** Compute the elements of $R$ using the standard Hamilton convention formulas [S12, unofficial].
   $$
   R = \begin{bmatrix}
   1 - 2(y^2 + z^2) & 2(xy - wz) & 2(xz + wy) \\
   2(xy + wz) & 1 - 2(x^2 + z^2) & 2(yz - wx) \\
   2(xz - wy) & 2(yz + wx) & 1 - 2(x^2 + y^2)
   \end{bmatrix}
   $$
   [S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S6, unofficial][S8, unofficial][S9, unofficial]
4. **Batching:** If processing a batch, stack the computed elements along the batch dimension and reshape to $(N, 3, 3)$ [S13, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def quaternion_to_rotation_matrix(q):
    """
    Converts a quaternion q to a 3x3 rotation matrix R.
    Input q shape: (..., 4) with order (w, x, y, z).
    Output R shape: (..., 3, 3).
    """
    # 1. Normalize
    norm = torch.sqrt(q[..., 0]**2 + q[..., 1]**2 + q[..., 2]**2 + q[..., 3]**2)
    q = q / norm[..., None]
    
    # 2. Unpack components (w, x, y, z)
    w = q[..., 0]
    x = q[..., 1]
    y = q[..., 2]
    z = q[..., 3]
    
    # 3. Compute matrix elements
    R = torch.empty((*q.shape[:-1], 3, 3), dtype=q.dtype, device=q.device)
    
    R[..., 0, 0] = 1 - 2 * (y**2 + z**2)
    R[..., 0, 1] = 2 * (x*y - w*z)
    R[..., 0, 2] = 2 * (x*z + w*y)
    
    R[..., 1, 0] = 2 * (x*y + w*z)
    R[..., 1, 1] = 1 - 2 * (x**2 + z**2)
    R[..., 1, 2] = 2 * (y*z - w*x)
    
    R[..., 2, 0] = 2 * (x*z - w*y)
    R[..., 2, 1] = 2 * (y*z + w*x)
    R[..., 2, 2] = 1 - 2 * (x**2 + y**2)
    
    return R
```

**Parameters:**
- None. The method is deterministic based on the input quaternion [G].

**Edge Cases and Pitfalls:**
- **Non-unit Quaternion:** If the input quaternion is not normalized, the resulting matrix will not be a pure rotation (it will include scaling) [S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S6, unofficial][S9, unofficial].
- **Sign Ambiguity:** Quaternions $q$ and $-q$ represent the same rotation [S7, unofficial].
- **Numerical Stability:** The direct formula is numerically stable for all unit quaternions [G].

## Subtopics

### SE3 to Quaternion and Translation
This variant extracts the quaternion and translation vector from a $4 \times 4$ SE3 transformation matrix [S9, unofficial]. It first extracts the top-left $3 \times 3$ submatrix as $R$ and the top-right $3 \times 1$ column as translation $t$ [S9, unofficial]. It then applies the standard Quaternion to SO3 Conversion (inverse direction) to convert $R$ to a quaternion $q$ [S9, unofficial]. The result is the pair $(q, t)$ [S9, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(1)$ for a single quaternion, $O(N)$ for a batch of $N$ quaternions [G].
- Memory: $O(1)$ auxiliary memory for a single quaternion, $O(N)$ for batch processing to store the output matrix [G].

**Measured Performance:**
- GPU implementations using tensor cores (e.g., PyTorch) can process millions of quaternions per second due to parallelization [S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S6, unofficial][S9, unofficial].

**Strengths and Limitations:**
- Strengths: Simple, fast, and numerically stable for unit quaternions [G].
- Limitations: Requires normalized input [S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S6, unofficial][S9, unofficial].

**When to Use:**
- Use when a rotation matrix is needed for linear algebra operations, such as transforming camera poses in volume rendering or Gaussian Splatting [S7, unofficial][S8, unofficial].

**When Not to Use:**
- Do not use if memory is extremely constrained and only the quaternion representation is needed for subsequent rotations (use [Quaternion Rotation](2302-quaternion-rotation.md) instead) [G].
- Do not use if the input quaternion is not normalized and normalization is not feasible [S5, unofficial].

## Background
The conversion from quaternion to rotation matrix is a fundamental operation in 3D computer graphics and robotics [G]. The Hamilton convention for quaternions is widely used in computer graphics [S8, unofficial]. The specific formula provided is standard and appears in many graphics libraries and research papers related to 3D reconstruction and Gaussian Splatting [S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S6, unofficial][S7, unofficial][S8, unofficial][S9, unofficial]. The method is attributed to the general mathematical properties of quaternions representing rotations in 3D space [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S6, unofficial][S7, unofficial][S8, unofficial][S9, unofficial].

## Sources
[S1] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S2] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S3] muskie82/MonoGS, https://github.com/muskie82/MonoGS, fetched 2026-10-27 (unofficial)
[S4] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S5] fastgs/FastGS, https://github.com/fastgs/FastGS, fetched 2026-10-27 (unofficial)
[S6] minghanqin/LangSplat, https://github.com/minghanqin/LangSplat, fetched 2026-10-27 (unofficial)
[S7] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S8] jibo27/3DGS_Hierarchical_Training, https://github.com/jibo27/3DGS_Hierarchical_Training, fetched 2026-10-27 (unofficial)
[S9] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S10] otakuxiang/gaussian-splatting-jittor, https://github.com/otakuxiang/gaussian-splatting-jittor, fetched 2026-10-27 (unofficial)
[S11] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S12] gsplat: An Open-Source Library for Gaussian Splatting, https://doi.org/10.48550/arXiv.2409.06765, fetched 2026-10-29 (unofficial)
[S13] https://github.com/WangFeng18/3d-gaussian-splatting, https://github.com/WangFeng18/3d-gaussian-splatting, fetched 2026-10-30 (unofficial)
