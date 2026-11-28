# Optimal Delaunay Triangulation (ODT) Loss


## Overview
The Optimal Delaunay Triangulation (ODT) loss is a mesh quality regularization term that penalizes tetrahedral elements for deviating from a regular shape. It is defined as the absolute difference between the moment of inertia of the tetrahedron's circumsphere and the sum of the tetrahedron's principal moments of inertia relative to that circumsphere. Unlike sibling methods that may rely on edge lengths or dihedral angles, ODT directly enforces geometric regularity by minimizing the energy to zero for regular tetrahedra, making it suitable for gradient-based optimization of vertex positions in non-regular volume representations.

## How it works
The method computes a scalar energy value $E_{\text{ODT}}$ for a single tetrahedron $T$ given its four vertices. The computation proceeds through the following steps:

1.  **Compute Edge Vectors and Volume**:
    Define vectors from vertex $v_0$ to the other vertices:
    $$ \mathbf{a} = v_1 - v_0, \quad \mathbf{b} = v_2 - v_0, \quad \mathbf{c} = v_3 - v_0 $$
    Compute the determinant $D$ of the matrix formed by these vectors:
    $$ D = \det([\mathbf{a}, \mathbf{b}, \mathbf{c}]) $$
    The volume $V_T$ is:
    $$ V_T = \frac{|D|}{6} $$

2.  **Compute Circumcenter**:
    The circumcenter $c_T$ is calculated as [S1]:
    $$ c_T = v_0 + \frac{\|\mathbf{a}\|^2(\mathbf{b} \times \mathbf{c}) + \|\mathbf{b}\|^2(\mathbf{c} \times \mathbf{a}) + \|\mathbf{c}\|^2(\mathbf{a} \times \mathbf{b})}{2D} $$
    *Note: If $D=0$, the tetrahedron is degenerate and the circumcenter is undefined.*

3.  **Compute Circumradius and Sphere Moment**:
    The circumradius $R_T$ is:
    $$ R_T = \|c_T - v_0\| $$
    Assuming unit mass density, the moment of inertia of the circumsphere $S_T$ (with mass equal to $V_T$) is:
    $$ M_{S_T} = \frac{2}{5} V_T R_T^2 $$

4.  **Compute Relative Coordinates**:
    For each vertex $v_i$ ($i=0,1,2,3$), compute coordinates relative to $c_T$:
    $$ x_i = v_i^x - c_T^x, \quad y_i = v_i^y - c_T^y, \quad z_i = v_i^z - c_T^z $$

5.  **Compute Coordinate Quadratic Sums**:
    Calculate the sums $S_x, S_y, S_z$ for each axis. For $S_x$:
    $$ S_x = \sum_{i=0}^3 x_i^2 + \sum_{0 \leq i < j \leq 3} x_i x_j $$
    Repeat for $S_y$ and $S_z$ using $y_i$ and $z_i$ respectively.

6.  **Compute Principal Moments and Sum**:
    The principal moments of inertia are:
    $$ I_x = \frac{V_T}{10}(S_y + S_z), \quad I_y = \frac{V_T}{10}(S_x + S_z), \quad I_z = \frac{V_T}{10}(S_x + S_y) $$
    The sum of principal moments $M_T$ is:
    $$ M_T = I_x + I_y + I_z = \frac{1}{5}V_T(S_x + S_y + S_z) $$

7.  **Calculate Loss**:
    $$ E_{\text{ODT}}(T) = |M_{S_T} - M_T| $$

The loss is minimized ($E_{\text{ODT}}=0$) when the tetrahedron is regular [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_odt_loss(v0, v1, v2, v3):
    """
    Computes the ODT loss for a single tetrahedron.
    Inputs: v0, v1, v2, v3 are tensors of shape (3,) representing vertex coordinates.
    Output: scalar loss E_ODT.
    """
    # 1. Edge vectors
    a = v1 - v0
    b = v2 - v0
    c = v3 - v0
    
    # 2. Determinant and Volume
    # det([a, b, c]) = a . (b x c)
    b_cross_c = torch.cross(b, c)
    D = torch.dot(a, b_cross_c)
    V_T = torch.abs(D) / 6.0
    
    # Guard against degenerate tetrahedra (D=0)
    # [G] If D is zero, the circumcenter is undefined. 
    # In optimization, this usually results in NaN gradients. 
    # A common guard is to return a large loss or skip gradient.
    if D == 0:
        return torch.tensor(float('inf'))

    # 3. Circumcenter
    # Numerator terms
    term1 = torch.norm(a)**2 * torch.cross(b, c)
    term2 = torch.norm(b)**2 * torch.cross(c, a)
    term3 = torch.norm(c)**2 * torch.cross(a, b)
    
    circumcenter_offset = (term1 + term2 + term3) / (2.0 * D)
    c_T = v0 + circumcenter_offset
    
    # 4. Circumradius and Sphere Moment
    R_T_sq = torch.sum((c_T - v0)**2)
    M_S_T = (2.0 / 5.0) * V_T * R_T_sq
    
    # 5. Relative Coordinates
    # v0, v1, v2, v3 relative to c_T
    rel_v = torch.stack([v0, v1, v2, v3]) - c_T
    x = rel_v[:, 0]
    y = rel_v[:, 1]
    z = rel_v[:, 2]
    
    # 6. Coordinate Quadratic Sums
    # Sum of squares
    sum_sq_x = torch.sum(x**2)
    sum_sq_y = torch.sum(y**2)
    sum_sq_z = torch.sum(z**2)
    
    # Sum of pairwise products for i < j
    # Indices: (0,1), (0,2), (0,3), (1,2), (1,3), (2,3)
    # Using broadcasting or explicit indexing
    # x_i * x_j
    sum_pair_x = (x[0]*x[1] + x[0]*x[2] + x[0]*x[3] + 
                  x[1]*x[2] + x[1]*x[3] + x[2]*x[3])
    sum_pair_y = (y[0]*y[1] + y[0]*y[2] + y[0]*y[3] + 
                  y[1]*y[2] + y[1]*y[3] + y[2]*y[3])
    sum_pair_z = (z[0]*z[1] + z[0]*z[2] + z[0]*z[3] + 
                  z[1]*z[2] + z[1]*z[3] + z[2]*z[3])
                  
    S_x = sum_sq_x + sum_pair_x
    S_y = sum_sq_y + sum_pair_y
    S_z = sum_sq_z + sum_pair_z
    
    # 7. Sum of Principal Moments M_T
    M_T = (1.0 / 5.0) * V_T * (S_x + S_y + S_z)
    
    # 8. Final Loss
    loss = torch.abs(M_S_T - M_T)
    
    return loss
```

### Parameters
- **Optimization Step Size**: The source specifies a constant step size of 0.002 for Signed Distance Function (SDF) values and 0.0003 for point positions during optimization [S1].
- **Optimizer**: AdamW is used in the reference implementation [S1].

### Edge Cases and Pitfalls
- **Degenerate Tetrahedra**: If $D=0$, the tetrahedron has zero volume and the circumcenter calculation involves division by zero. This results in undefined behavior or NaN gradients. Implementers must handle $D \approx 0$ explicitly, either by clamping, returning a high loss, or skipping the element.
- **Precision**: The calculation involves cross products and determinants which can suffer from precision issues in highly skewed tetrahedra.

## Operating characteristics
- **Time Complexity**: The computation is $O(1)$ per tetrahedron, involving a fixed number of arithmetic operations.
- **Memory Complexity**: $O(1)$ auxiliary memory per tetrahedron.
- **Strengths**: The loss is differentiable and can be integrated into gradient-based optimizers (e.g., via PyTorch autograd) to simultaneously optimize mesh quality and reconstruction [S1]. It theoretically reaches zero for regular tetrahedra, providing a strong geometric prior [S1].
- **Limitations**: It does not explicitly prevent inverted tetrahedra (negative volume) unless combined with other constraints, though the volume term $V_T$ is used in the calculation. It is computationally more expensive than simple edge-length based regularizers due to the circumcenter and cross-product calculations.

## Background
The ODT loss originates from the field of mesh generation and optimization, specifically referencing the work of Alliez et al. [2005] and Chen and Xu [2004] [S1]. While Chen and Xu derived update steps to minimize this loss, the method described here utilizes the loss function itself for gradient-based optimization in the context of gradient-based mesh optimization for volume rendering [S1].

## Sources
[S1] TetWeave: Isosurface Extraction using On-The-Fly Delaunay Tetrahedral Grids for Gradient-Based Mesh Optimization, https://arxiv.org/abs/2505.04590v2, fetched 2026-10-26
