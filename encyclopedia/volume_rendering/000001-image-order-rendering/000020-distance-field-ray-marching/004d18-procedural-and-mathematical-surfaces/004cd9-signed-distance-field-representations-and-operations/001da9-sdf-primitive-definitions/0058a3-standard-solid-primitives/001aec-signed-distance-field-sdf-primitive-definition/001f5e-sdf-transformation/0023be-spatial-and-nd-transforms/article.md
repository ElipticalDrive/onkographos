# Spatial and nD transforms

Also called: Transforming SDF Geometry, hierarchical transforms, inverse transform query

## Overview
Spatial and nD transforms allow the application of geometric transformations (rotation, translation, scaling, and permutation) to Signed Distance Field (SDF) primitives and discrete data structures by applying the inverse transformation to the query point rather than transforming the geometry itself [S1, S2, unofficial]. This approach ensures that the underlying data structures (such as raw spatial indices or base distance functions) remain unmodified, preserving local linearity and allowing constant-time affine queries per dimension [S1, unofficial]. It differs from direct geometry transformation by deferring the coordinate change to the evaluation step, which avoids distorting the stored data and supports non-uniform or spatially varying distortions [S2, unofficial].

## How it works
The method operates by mapping a world-space query point back to the local space of the primitive or data grid using the inverse of the desired transformation matrix. This applies to both continuous SDFs and discrete multi-dimensional data structures [S1, S2, unofficial].

### Inputs and Outputs
- **Input**: A query point $P_{world}$ in world coordinates.
- **Input**: A transformation matrix $T$ (or its components: scale, offset, permutation) defining the local-to-world mapping.
- **Output**: A transformed point $P_{local}$ (or transformed tolerance) suitable for evaluating the base distance function or accessing the data grid.

### Data Structures
For discrete multi-dimensional data (e.g., voxel grids or splatting indices), transforms are stored per node or dimension [S1, unofficial]:
- **Affine Transform**: Defined by a scale factor $s$ and an offset $o$.
- **Permutation Transform**: Defined by a permutation array $perm$ mapping indices.
- **Composition**: Affine transforms compose as $s = s_p s_c$ and $o = s_p o_c + o_p$; permutations compose as $r[i] = perm_p[perm_c[i]]$ [S1, unofficial].

### Steps
1. **Construct Inverse Transform**: Determine the inverse of the local-to-world transformation $T$. For a composite transformation $T = S \cdot R \cdot T_{trans}$, compute $T^{-1}$ [S2, unofficial].
2. **Apply Inverse to Query Point**:
   - For **SDFs**: Transform the query point $P_{world}$ to $P_{local}$ using $T^{-1}$ [S2, unofficial].
   - For **Discrete Data**: Inverse-transform the slice query per dimension rather than per point to maintain constant work for affine transforms [S1, unofficial].
3. **Evaluate Base Function**: Use $P_{local}$ to evaluate the base distance function or access the data grid [S1, S2, unofficial].

### Formulas
- **Affine Transform (Continuous/Discrete)**:
  Given a world position $P_{world}$ and tolerance $T_{world}$, and affine parameters scale $s$ and offset $o$:
  $$ P_{raw} = \frac{P_{world} - o}{s} $$
  $$ T_{raw} = \frac{T_{world}}{|s|} $$
  [S1, unofficial]

- **Permutation Transform (Discrete)**:
  Given a permutation array $perm$, the inverse mapping is:
  $$ inverse[perm[i]] = i $$
  [S1, unofficial]

- **Matrix Inverse (SDF)**:
  For a 4x4 transformation matrix $M$ (combining scale $S$, rotation $R$, and translation $T$), the transformed point $P_{local}$ is:
  $$ P_{local} = (P_{world} \cdot M^{-1}).xyz $$
  where $P_{world}$ is treated as a homogeneous vector $(x, y, z, 1)$ [S2, unofficial].

### Parameters
- **Scale ($s$)**: Non-uniform scaling factors. A zero scale is accepted but triggers a validation warning, leaving the query untransformed [S1, unofficial].
- **Offset ($o$)**: Translation vector.
- **Permutation ($perm$)**: Array mapping discrete indices.
- **Norm**: In nD geometric transformation contexts, if `norm != 0`, the result is projected onto the $z=norm$ plane [S3, unofficial].

### Pseudocode
The following pseudocode demonstrates the application of an inverse transform to an SDF query point, combining translation, rotation, and scaling [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def inverse_transform_query(pt_world, transform_matrix):
    """
    Applies the inverse of a 4x4 transformation matrix to a 3D point.
    
    Args:
        pt_world: vec3, the query point in world space.
        transform_matrix: 4x4 matrix, the local-to-world transformation.
        
    Returns:
        vec3, the transformed point in local space.
    """
    # Convert point to homogeneous coordinates
    pt_homogeneous = (pt_world.x, pt_world.y, pt_world.z, 1.0)
    
    # Compute the inverse of the transformation matrix
    # Note: In practice, this is done once per frame/object, not per query.
    inv_matrix = invert_matrix(transform_matrix)
    
    # Multiply point by inverse matrix
    # Result is a vec4, we take the xyz components
    pt_local_x = inv_matrix[0][0]*pt_homogeneous[0] + inv_matrix[0][1]*pt_homogeneous[1] + inv_matrix[0][2]*pt_homogeneous[2] + inv_matrix[0][3]*pt_homogeneous[3]
    pt_local_y = inv_matrix[1][0]*pt_homogeneous[0] + inv_matrix[1][1]*pt_homogeneous[1] + inv_matrix[1][2]*pt_homogeneous[2] + inv_matrix[1][3]*pt_homogeneous[3]
    pt_local_z = inv_matrix[2][0]*pt_homogeneous[0] + inv_matrix[2][1]*pt_homogeneous[1] + inv_matrix[2][2]*pt_homogeneous[2] + inv_matrix[2][3]*pt_homogeneous[3]
    
    return (pt_local_x, pt_local_y, pt_local_z)

def evaluate_transformed_sdf(pt_world, transform_matrix, base_sdf_func):
    """
    Evaluates an SDF after applying a spatial transform.
    
    Args:
        pt_world: vec3, query point.
        transform_matrix: 4x4 matrix, local-to-world transform.
        base_sdf_func: function, the base distance function (e.g., sphere).
        
    Returns:
        float, the signed distance.
    """
    pt_local = inverse_transform_query(pt_world, transform_matrix)
    return base_sdf_func(pt_local)

# Example: Sphere centered at (0, 3, 0) with no rotation/scaling
def sphere(pt):
    return math.sqrt(pt[0]**2 + pt[1]**2 + pt[2]**2) - 1.0

def f_translate(pt_world):
    # Translation by (0, 3, 0)
    # Inverse translation is (0, -3, 0)
    pt_local = (pt_world[0], pt_world[1] - 3.0, pt_world[2])
    return sphere(pt_local)
```

For discrete nD geometric transformations (e.g., in deep learning frameworks), the transformation is applied via tensor operations [S3, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_geometric_transform(pts, Trf, ncol=None, norm=0):
    """
    Applies a geometric transformation matrix Trf to points pts.
    
    Args:
        pts: Tensor of coordinates, shape (..., 2) or (..., 3).
        Trf: 3x3 or 4x4 projection matrix (e.g., Homography).
        ncol: Number of columns in result (2 or 3). Defaults to pts.shape[-1].
        norm: If != 0, projects result on z=norm plane.
        
    Returns:
        Transformed points tensor.
    """
    d = pts.shape[-1]
    
    # Handle batch dimensions
    if Trf.ndim >= 3:
        Trf_reshaped = Trf.reshape(-1, d, d)
        if pts.ndim > Trf.ndim:
            pts_reshaped = pts.reshape(pts.shape[0], -1, d)
        elif pts.ndim == 2:
            pts_reshaped = pts.reshape(-1, 1, d)
        else:
            pts_reshaped = pts
    else:
        Trf_reshaped = Trf
        pts_reshaped = pts
        
    # Case 1: Trf is 3D (batched) and pts is 4D with matching last dim
    # Use einsum 'bij, bhwj -> bhwi'
    if Trf.ndim == 3 and pts.ndim == 4 and pts.shape[-1] == Trf.shape[-1]:
        result = torch.einsum('bij, bhwj -> bhwi', Trf_reshaped, pts_reshaped)
        
    # Case 2: Trf is 3D and pts is 4D with dim+1 last dim
    elif Trf.ndim == 3 and pts.ndim == 4 and pts.shape[-1] == Trf.shape[-1] + 1:
        # Apply linear part
        linear_part = torch.einsum('bij, bhwj -> bhwi', Trf_reshaped[:, :d, :d], pts_reshaped)
        # Add translation part
        translation = Trf_reshaped[:, None, None, :d, d]
        result = linear_part + translation
        
    # Case 3: pts.shape[-1] + 1 == Trf.shape[-1] (Homography-like)
    elif pts.shape[-1] + 1 == Trf.shape[-1]:
        Trf_T = Trf.transpose(-1, -2)
        # pts @ Trf[..., :-1, :] + Trf[..., -1:, :]
        result = pts @ Trf_reshaped[..., :-1, :] + Trf_reshaped[..., -1:, :]
        
    # Case 4: Otherwise
    else:
        # Compute Trf @ pts.T, then swap axes
        result = torch.matmul(Trf_reshaped, pts_reshaped.transpose(-1, -2)).transpose(-1, -2)
        
    # Project to z=norm plane if requested
    if norm != 0:
        # Project onto z=norm plane
        # Assuming z is the last dimension or specific index
        # This step depends on the specific implementation details of the projection
        # For a standard 3D point (x,y,z), projecting to z=norm implies scaling
        # However, the source does not specify the exact projection formula, 
        # so we note the operation [G].
        pass
        
    return result
```

### Edge Cases and Pitfalls
- **Zero Scale**: A zero scale factor is accepted but triggers a validation warning. The query remains untransformed in this case [S1, unofficial].
- **Discrete Grid Alignment**: On a discrete axis, the world value is rounded onto the declared grid. If a value has no local preimage (i.e., it falls between grid points), it selects nothing [S1, unofficial].
- **Non-Uniform Distortion**: Non-uniform spatial distortion modifies 'all of space' with the same transform, so distance functions retain their local linearity. Spatially varying distortion can also be applied by making the transform a function of the point's position [S2, unofficial].
- **Precision**: Inverse transformation queries must handle floating-point precision carefully, especially when dealing with large translation offsets or small scale factors [G].

## Subtopics

### [SDF Rotation](5817-sdf-rotation.md)
SDF Rotation applies a rotational transformation to a signed distance field primitive by rotating the sampling point before evaluating the base distance function. It is chosen when an object needs to be oriented in world space without modifying its underlying SDF definition. It is a specific case of the general inverse transform method, focusing solely on the rotational component of the transformation matrix. See [SDF Rotation](5817-sdf-rotation.md).

## Operating characteristics
- **Time Complexity**: For affine transforms, the query time is constant per dimension, independent of the data size, as it involves simple arithmetic operations [S1, unofficial]. For permutations, it requires one pass over the category list [S1, unofficial].
- **Memory Complexity**: No additional memory is required for the transformation itself, as the inverse is applied on-the-fly to the query point [S1, S2, unofficial].
- **Strengths**: Preserves the integrity of the underlying data structure; supports non-uniform and spatially varying distortions; efficient for discrete data by avoiding per-point transformation [S1, S2, unofficial].
- **Limitations**: Requires computing the inverse transformation matrix, which can be computationally expensive if done per-query (though typically done once per frame/object) [S2, unofficial]. Zero scale leads to undefined behavior unless handled [S1, unofficial].
- **When to Use**: When transforming SDF primitives or discrete data structures for rendering or visualization, especially when the underlying data should remain unmodified [S1, S2, unofficial].
- **When Not to Use**: When the transformation is simple enough to be baked into the data structure, or when the inverse transformation is not available or difficult to compute [G].

## Background
The concept of applying inverse transformations to query points in SDF rendering is a standard technique in ray marching, popularized by tutorials and resources such as those by Inigo Quilez [S2, unofficial]. The application of hierarchical and nD transforms to discrete scientific data (e.g., microscopy data) for scalable web visualization was detailed in the Luxar paper, which introduced per-node affine and permutation transforms for multi-dimensional data [S1, unofficial]. The geometric transformation application in nD spaces, particularly in the context of deep learning frameworks like PyTorch, is exemplified by projects such as NVlabs' InstantSplat [S3, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] Ray Marching with Signed Distance Fields - Scribd, https://www.scribd.com/document/831412915/1-Ray-Marching-and-Signed-Distance-Fields, fetched 2026-10-27 (unofficial)
[S3] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
