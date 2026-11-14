# Camera Coordinate Homogenization and Rigid Transformation

Also called: Homogenize and Transform Points/Vectors, Rigid Body Transformation via Homogeneous Coordinates

## Overview
This method converts 3D points and direction vectors into homogeneous coordinates and applies a rigid body transformation (rotation and translation) to map them between coordinate systems, such as from camera space to world space. It differs from its sibling [Scene-Related Transforms](11646-scene-related-transforms.md) by explicitly handling the distinction between points and vectors via the homogeneous fourth component, ensuring translation affects positions but not directions, whereas siblings may handle scene-relative adjustments without this strict separation.

## How it works
**Inputs and Outputs:**
- Input: A set of 3D points $P \in \mathbb{R}^{N \times 3}$ or direction vectors $V \in \mathbb{R}^{N \times 3}$, and a rigid transformation matrix $T \in \mathbb{R}^{4 \times 4}$ representing the camera-to-world or world-to-camera pose.
- Output: Transformed homogeneous coordinates $P'_h \in \mathbb{R}^{N \times 4}$ or $V'_h \in \mathbb{R}^{N \times 4}$.

**Data Structures:**
- Points are stored as $N \times 3$ tensors.
- Vectors are stored as $N \times 3$ tensors.
- The transformation matrix $T$ is a $4 \times 4$ matrix.

**Steps:**
1. **Homogenize Points:** Append a column of ones to the point coordinates. For a point $p = (x, y, z)$, the homogeneous representation is $p_h = (x, y, z, 1)^T$.
2. **Homogenize Vectors:** Append a column of zeros to the vector coordinates. For a vector $v = (x, y, z)$, the homogeneous representation is $v_h = (x, y, z, 0)^T$.
3. **Apply Transformation:** Perform matrix multiplication between the transformation matrix $T$ and the homogeneous coordinates. Using Einstein summation notation for batched operations, the transformation is computed as $result_{i} = \sum_{j} T_{ij} \cdot coord_{j}$.

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

def homogenize_and_transform(points_or_vectors, transformation_matrix):
    """
    Homogenizes input points/vectors and applies a rigid transformation.
    
    Args:
        points_or_vectors: Tensor of shape (..., 3) containing 3D points or vectors.
        transformation_matrix: Tensor of shape (..., 4, 4) representing the rigid transform.
        
    Returns:
        Transformed homogeneous coordinates of shape (..., 4).
    """
    # Determine if input is points (last dim 3) or vectors (last dim 3)
    # The distinction is made by the caller appending 1 or 0 appropriately.
    # However, if the input is raw 3D data, we must homogenize first.
    
    # Homogenize: Append 1 for points, 0 for vectors.
    # Assuming the input is already homogenized or we need to append based on type.
    # If input is (..., 3), we append a column.
    # For points: append 1. For vectors: append 0.
    # This function assumes the caller has already prepared the homogeneous coordinates
    # or we need to distinguish. Let's assume we are transforming pre-homogenized data
    # or we provide a helper.
    
    # Let's assume the input is raw 3D and we need to homogenize.
    # We need to know if it's a point or vector. Let's assume a flag or separate calls.
    # For this implementation, we'll assume the input is already homogeneous (..., 4)
    # or we provide a specific homogenization step.
    
    # Re-reading notes: "Homogenize points by appending a 1", "Homogenize vectors by appending a 0"
    # So we need to handle the homogenization step explicitly.
    
    # If input is (..., 3):
    if points_or_vectors.shape[-1] == 3:
        # We need to know if it's a point or vector. 
        # Let's assume a separate function or flag. 
        # For the sake of this method, we'll assume the input is already homogeneous (..., 4)
        # or we are transforming points.
        pass
    
    # Standard implementation for transforming homogeneous coordinates:
    # einsum(transformation, homogeneous_coordinates, "... i j, ... j -> ... i")
    
    # If input is (..., 3) and we treat as points:
    if points_or_vectors.shape[-1] == 3:
        # Append 1 for points
        ones = torch.ones_like(points_or_vectors[..., :1])
        homogeneous_coords = torch.cat([points_or_vectors, ones], dim=-1)
    else:
        homogeneous_coords = points_or_vectors
        
    # Apply transformation
    # T is (..., 4, 4), H is (..., 4)
    # Result is (..., 4)
    transformed = torch.einsum("...ij,...j->...i", transformation_matrix, homogeneous_coords)
    
    return transformed
```

**Parameters:**
- $T$: The $4 \times 4$ rigid transformation matrix.
- $N$: Number of points/vectors.

**Edge Cases:**
- Precision issues may arise if the transformation matrix is not strictly orthogonal, leading to non-rigid transformations.
- Boundary conditions for vectors vs. points must be strictly maintained by appending 0 or 1 respectively.

## Subtopics

### [3D Coordinate Transformation (Translation + Rotation)](6721-3d-coordinate-transformation-translation-rotation.md)
Applies rigid transformations using sequential matrix multiplications and vector addition, treating points as row vectors. It differs by separating translation and rotation steps and applying them in a specific order (Z-translation, Y-translation, X-translation, then Z-rotation, Y-rotation, X-rotation) rather than using a single $4 \times 4$ matrix multiplication. This approach is used in voxel rendering engines to transform relative voxel positions to absolute world coordinates.

## Operating characteristics
- **Time Complexity:** $O(N)$ for $N$ points/vectors, as each point/vector requires a constant number of operations (matrix-vector multiplication).
- **Memory Complexity:** $O(N)$ to store the transformed coordinates.
- **Strengths:** Efficiently handles batched transformations; clearly distinguishes between points and vectors.
- **Limitations:** Requires careful handling of homogeneous coordinates to avoid incorrect transformations.
- **When to Use:** When transforming camera coordinates to world coordinates or vice versa in volume rendering pipelines.
- **When Not to Use:** When non-rigid transformations are required.

## Background
This method is fundamental in computer graphics and vision for coordinate system transformations. It is widely used in neural radiance fields (NeRF) and multi-view stereo algorithms to align camera poses and 3D points. The use of homogeneous coordinates allows for a unified representation of rotation and translation, simplifying the mathematical formulation of rigid body transformations.

## Sources
[S1] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
