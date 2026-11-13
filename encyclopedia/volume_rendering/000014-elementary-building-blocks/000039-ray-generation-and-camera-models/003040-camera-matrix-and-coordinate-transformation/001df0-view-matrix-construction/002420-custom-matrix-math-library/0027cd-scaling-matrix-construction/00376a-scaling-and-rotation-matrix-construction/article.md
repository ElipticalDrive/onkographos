# Scaling and Rotation Matrix Construction
## Overview
Constructs a 3x3 transformation matrix by combining scaling and rotation. It creates a diagonal scaling matrix $L$ and multiplies it by a rotation matrix $R$. This method serves as the core linear transformation component for 3D geometry, specifically used in Gaussian Splatting variants to define the covariance or shape of volumetric primitives. It is distinguished from [Scaling Matrix Construction](4126-scaling-matrix-construction.md) by the inclusion of rotational degrees of freedom, and from [Transformation Matrix Construction](12879-transformation-matrix-construction.md) by the exclusion of translation.
## How it works
**Inputs:**
- `s`: A tensor of shape `(N, 3)` or `(3,)` representing scale factors for $N$ primitives. `s[:,0]` is scale along X, `s[:,1]` along Y, `s[:,2]` along Z.
- `r`: Rotation parameters. The exact format depends on the implementation variant:
  - [S1, unofficial] Uses a helper `build_rotation(r)` which typically expects Euler angles or quaternions (implementation detail of `build_rotation` is not fully specified in the provided notes, but implied to be a standard rotation construction).
  - [S2, unofficial] Explicitly derives rotation from quaternions.

**Outputs:**
- `M`: A 3x3 transformation matrix of shape `(N, 3, 3)` (or `(3, 3)` for single primitive).

**Data Structures:**
- `L`: Diagonal scaling matrix.
- `R`: Rotation matrix.

**Steps:**
1. Initialize the scaling matrix $L$. For batched inputs of size $N$, $L$ has shape $(N, 3, 3)$.
2. Set the diagonal elements of $L$:
   $$ L_{i,0,0} = s_{i,0} $$
   $$ L_{i,1,1} = s_{i,1} $$
   $$ L_{i,2,2} = s_{i,2} $$
   All off-diagonal elements are zero.
3. Compute the rotation matrix $R$ from the rotation parameters `r`.
   - If using quaternions (as in [S2, unofficial]), convert the quaternion to a 3x3 rotation matrix.
   - If using Euler angles (as implied by [S1, unofficial]'s context in InstantSplat), construct $R$ from the angles. Note that [S1, unofficial] mentions $R_z R_y R_x$ order in the child article, but the specific multiplication order for `build_rotation` is not explicitly detailed in the provided snippets for this method itself.
4. Compute the final transformation matrix $M$ by matrix multiplication:
   $$ M = L \times R $$
   In code, this is `L @ R`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def build_scaling_rotation(s, r):
    """
    Constructs a 3x3 transformation matrix by combining scaling and rotation.
    
    Args:
        s: Tensor of shape (N, 3) or (3,) containing scale factors [sx, sy, sz].
        r: Rotation parameters. Format depends on the specific build_rotation implementation.
           For quaternion-based (S2), r should be a quaternion (w, x, y, z) or (x, y, z, w).
           For Euler-based (S1), r should be Euler angles.
           
    Returns:
        M: Tensor of shape (N, 3, 3) containing the combined scaling and rotation matrix.
    """
    # Handle batch dimension
    if s.dim() == 1:
        s = s.unsqueeze(0)
        
    N = s.shape[0]
    
    # Step 1 & 2: Initialize and fill diagonal scaling matrix L
    # L shape: (N, 3, 3)
    L = torch.zeros(N, 3, 3, device=s.device, dtype=s.dtype)
    
    # Set diagonal elements
    L[:, 0, 0] = s[:, 0]
    L[:, 1, 1] = s[:, 1]
    L[:, 2, 2] = s[:, 2]
    
    # Step 3: Compute rotation matrix R
    # Note: The exact implementation of build_rotation depends on the source.
    # [S2, unofficial] uses quaternions. [S1, unofficial] uses a helper that likely handles Euler angles or quaternions.
    # We assume a generic build_rotation function exists that takes r and returns R.
    R = build_rotation(r)
    
    # Step 4: Multiply L and R
    # M = L @ R
    M = torch.bmm(L, R)
    
    return M

def build_rotation(r):
    """
    Placeholder for rotation matrix construction.
    In [S2, unofficial], this converts quaternions to rotation matrices.
    In [S1, unofficial], this may handle Euler angles.
    Implementation details are not fully provided in the notes for this specific method.
    [G] Standard quaternion to rotation matrix conversion:
    If r is (w, x, y, z):
    x, y, z, w = r
    return torch.tensor([
        [1-2*(y**2 + z**2), 2*(x*y - z*w), 2*(x*z + y*w)],
        [2*(x*y + z*w), 1-2*(x**2 + z**2), 2*(y*z - x*w)],
        [2*(x*z - y*w), 2*(y*z + x*w), 1-2*(x**2 + y**2)]
    ])
    """
    # This function's implementation is not fully detailed in the provided notes.
    # It is assumed to be available in the context of the calling code.
    raise NotImplementedError("build_rotation implementation depends on specific source (S1 or S2)")
```

**Edge Cases and Pitfalls:**
- **Zero Scale:** If any scale factor is zero, the matrix becomes singular. This may cause issues in inverse transformations or covariance calculations.
- **Rotation Parameter Format:** The input `r` must match the expected format of `build_rotation`. Mixing Euler angles and quaternions without conversion will lead to incorrect results.
- **Batch Dimension:** Ensure `s` and `r` have compatible batch dimensions. If `s` is `(N, 3)`, `r` must also support batched operations or be broadcastable.
- **Precision:** Floating-point precision issues may arise in the rotation matrix construction, especially for small angles or near-degenerate quaternions. Normalization of quaternions is recommended before conversion.

## Subtopics

### [Transformation Matrix Construction](12879-transformation-matrix-construction.md)
Computes a 4x4 homogeneous transformation matrix from translation, rotation (Euler angles), and scale factors for 3D geometry. It combines the scaling and rotation logic of [Scaling and Rotation Matrix Construction](7986-scaling-and-rotation-matrix-construction.md) with translation to produce a complete model matrix. It is distinguished by its explicit construction of individual axis rotation matrices and their specific multiplication order ($R_z R_y R_x$) before applying scale.

## Operating characteristics
- **Time Complexity:** $O(N)$ for constructing the diagonal matrix $L$, where $N$ is the number of primitives. The rotation matrix construction $R$ depends on the rotation parameter format (e.g., $O(1)$ per primitive for quaternion to matrix conversion). The matrix multiplication $L \times R$ is $O(N)$ since $L$ is diagonal.
- **Memory Complexity:** $O(N)$ for storing the output matrix $M$ of shape $(N, 3, 3)$.
- **Performance:** Efficient for batched operations on GPU, as matrix operations are highly parallelizable.
- **Strengths:** Simple and efficient construction of linear transformations. Separates scaling and rotation, allowing for independent control.
- **Limitations:** Does not include translation. Requires a separate method for full model matrix construction.
- **When to Use:** When only linear transformation (scaling and rotation) is needed for 3D primitives, such as in Gaussian Splatting covariance computation.
- **When Not to Use:** When translation is required; use [Transformation Matrix Construction](12879-transformation-matrix-construction.md) instead.

## Background
This method is commonly used in 3D computer graphics, particularly in Gaussian Splatting variants like InstantSplat [S1, unofficial] and 4D Gaussian Splatting [S2, unofficial]. It provides a fundamental building block for defining the orientation and scale of volumetric primitives in 3D space. The separation of scaling and rotation allows for intuitive parameterization of object transformations.

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
