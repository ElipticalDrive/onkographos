# Camera Coordinate Homogenization and Rigid Transformation

Also called: Homogenize and Transform Points/Vectors, Rigid Body Transformation via Homogeneous Coordinates

## Overview
This method converts 3D points and direction vectors into homogeneous coordinates and applies a rigid body transformation to map them between coordinate systems, such as from camera space to world space [S1, unofficial]. It differs from its sibling [Scene-Related Transforms](11646-scene-related-transforms.md) by explicitly handling the distinction between points and vectors via the homogeneous fourth component, ensuring translation affects positions but not directions [G].

## How it works
**Inputs and Outputs:**
- Input: A set of 3D points $P \in \mathbb{R}^{N \times 3}$ or direction vectors $V \in \mathbb{R}^{N \times 3}$, and a rigid transformation matrix $T \in \mathbb{R}^{4 \times 4}$ representing the camera-to-world or world-to-camera pose [S1, unofficial].
- Output: Transformed homogeneous coordinates $P'_h \in \mathbb{R}^{N \times 4}$ or $V'_h \in \mathbb{R}^{N \times 4}$ [G].

**Data Structures:**
- Points and vectors are stored as $N \times 3$ tensors [S1, unofficial].
- The transformation matrix $T$ is a $4 \times 4$ tensor [S1, unofficial].

**Steps:**
1. **Homogenize Points:** Append a column of ones to the point coordinates. For a point $p = (x, y, z)$, the homogeneous representation is $p_h = (x, y, z, 1)^T$ [S1, unofficial].
2. **Homogenize Vectors:** Append a column of zeros to the vector coordinates. For a vector $v = (x, y, z)$, the homogeneous representation is $v_h = (x, y, z, 0)^T$ [S1, unofficial].
3. **Apply Transformation:** Perform matrix multiplication between the transformation matrix $T$ and the homogeneous coordinates. Using Einstein summation notation for batched operations, the transformation is computed as $result_{i} = \sum_{j} T_{ij} \cdot coord_{j}$ [S1, unofficial].

**Formulas:**
For a point $p_h$:
$$
p'_h = T \cdot p_h = \begin{bmatrix} R & t \\ 0 & 1 \end{bmatrix} \begin{bmatrix} x \\ y \\ z \\ 1 \end{bmatrix} = \begin{bmatrix} Rx + t \\ 1 \end{bmatrix}
$$
For a vector $v_h$:
$$
v'_h = T \cdot v_h = \begin{bmatrix} R & t \\ 0 & 1 \end{bmatrix} \begin{bmatrix} x \\ y \\ z \\ 0 \end{bmatrix} = \begin{bmatrix} Rx \\ 0 \end{bmatrix}
$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def homogenize_and_transform(points_or_vectors, transformation_matrix, is_point=True):
    """
    Homogenizes input points/vectors and applies a rigid transformation.
    
    Args:
        points_or_vectors: Tensor of shape (..., 3) containing 3D points or vectors.
        transformation_matrix: Tensor of shape (..., 4, 4) representing the rigid transform.
        is_point: Boolean flag; True for points (appends 1), False for vectors (appends 0).
        
    Returns:
        Transformed homogeneous coordinates of shape (..., 4).
    """
    # Homogenize: append 1 for points, 0 for vectors
    if points_or_vectors.shape[-1] == 3:
        ones = torch.ones_like(points_or_vectors[..., :1])
        zeros = torch.zeros_like(points_or_vectors[..., :1])
        homogeneous_coords = torch.cat([points_or_vectors, ones if is_point else zeros], dim=-1)
    else:
        homogeneous_coords = points_or_vectors
        
    # Apply rigid transformation via einsum: "... i j, ... j -> ... i"
    transformed = torch.einsum("... i j, ... j -> ... i", transformation_matrix, homogeneous_coords)
    return transformed
```

**Parameters:**
- $T$: The $4 \times 4$ rigid transformation matrix [S1, unofficial].
- $N$: Number of points/vectors [G].

**Edge Cases:**
- Not given in the sources.

## Subtopics

### [3D Coordinate Transformation (Translation + Rotation)](6721-3d-coordinate-transformation-translation-rotation.md)
Applies a rigid body transformation to 3D points by sequentially multiplying row vectors by $3 \times 3$ rotation matrices and adding a translation vector, explicitly avoiding $4 \times 4$ homogeneous matrices. It solves the problem of combining rotation and translation into a single linear operation while preserving explicit control over the transformation order. Choose this variant when explicit control over transformation order is required or when avoiding homogeneous coordinate overhead is necessary.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
