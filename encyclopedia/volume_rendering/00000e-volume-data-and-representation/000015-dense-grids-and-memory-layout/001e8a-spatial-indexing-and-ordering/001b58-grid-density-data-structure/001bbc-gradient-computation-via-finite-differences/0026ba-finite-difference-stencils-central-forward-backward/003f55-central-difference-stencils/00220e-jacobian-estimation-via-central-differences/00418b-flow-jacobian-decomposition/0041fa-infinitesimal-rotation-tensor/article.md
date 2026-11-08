# Infinitesimal Rotation Tensor

Also called: Antisymmetric part of displacement gradient, Local rotation tensor

## Overview
The Infinitesimal Rotation Tensor is a second-order tensor that quantifies the local rigid-body rotation within a continuous medium, derived from the antisymmetric part of the displacement gradient [S1, unofficial]. It serves to decompose the local flow behavior into rotation and deformation (strain), allowing volume rendering and visualization to distinguish between pure orientation changes and geometric distortion [S1, unofficial]. Unlike the Infinitesimal Strain Tensor, which captures symmetric stretching and shearing, this tensor isolates the rotational component related to the curl of the vector field [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A differentiable displacement function $u: \Omega \subseteq \mathbb{R}^n \rightarrow \mathbb{R}^n$, where $\Omega$ is the domain of the continuum body [S1, unofficial].
- **Output:** The Infinitesimal Rotation Tensor $r$, a symmetric matrix (in 2D/3D representation) or tensor with components $r_{ij}$ [S1, unofficial].

**Data Structures:**
- The tensor is typically stored as a dense grid of $n \times n$ matrices (where $n$ is the spatial dimension, usually 2 or 3) aligned with the spatial coordinates $x_j$ [S1, unofficial].
- Each element $r_{ij}$ is a floating-point value representing the contribution of the $j$-th coordinate direction to the rotation in the $i$-th direction.

**Formulas:**
The components of the Infinitesimal Rotation Tensor are defined as [S1, unofficial]:

$$
r_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} - \frac{\partial u_j}{\partial x_i} \right)
$$

Where:
- $u_i$ is the $i$-th component of the displacement vector field $u$.
- $x_j$ is the $j$-th spatial coordinate.
- $\frac{\partial u_i}{\partial x_j}$ is the partial derivative of the $i$-th displacement component with respect to the $j$-th spatial coordinate.

This tensor describes a rotation around the axis defined by the curl of the displacement field, $\nabla \times u$ [S1, unofficial].

**Parameters:**
- The method assumes small deformations, specifically requiring $\|u_k\| \ll 1$ and $\|\nabla u\| \ll 1$ [S1, unofficial]. This ensures the linearization is valid.

**Steps:**
1.  **Compute Displacement Gradient:** Calculate the Jacobian matrix $J$ of the displacement function $u$, where $J_{ij} = \frac{\partial u_i}{\partial x_j}$. This requires estimating partial derivatives using a finite difference stencil (e.g., Central Difference) on the input vector field [S1, unofficial].
2.  **Extract Antisymmetric Part:** Compute the tensor components using the formula above. This is equivalent to taking the antisymmetric part of the Jacobian: $r = \frac{1}{2}(J - J^T)$.
3.  **Interpretation:** The resulting tensor $r$ represents the local rotation. The magnitude of rotation can be derived from the norm of the tensor or directly from the curl vector $\nabla \times u$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_infinitesimal_rotation_tensor(displacement_field, grid_spacing):
    """
    Computes the Infinitesimal Rotation Tensor from a displacement vector field.
    
    Args:
        displacement_field: A 3D array (or 2D for 2D space) of shape (N, N, N, 3) 
                            representing u(x,y,z).
        grid_spacing: Scalar float representing the uniform grid spacing dx.
        
    Returns:
        rotation_tensor: A 3D array of shape (N, N, N, 3, 3) representing r_ij.
    """
    # Dimensions
    N = displacement_field.shape[0]
    dim = displacement_field.shape[3] # Usually 2 or 3
    
    # Initialize output tensor
    rotation_tensor = np.zeros((N, N, N, dim, dim))
    
    # Compute partial derivatives using central differences
    # du_i / dx_j
    # We need a function to compute gradient of a scalar field component
    # Since displacement_field is a vector field, we compute gradient for each component
    
    # Precompute gradients for each component u_0, u_1, u_2
    # grad_u[i] is the gradient of the i-th component of u
    grad_u = []
    for i in range(dim):
        # Extract scalar field for u_i
        scalar_field = displacement_field[..., i]
        # Compute gradient of scalar_field
        # Using central differences: df/dx = (f(x+dx) - f(x-dx)) / (2*dx)
        gradient = compute_gradient_central(scalar_field, grid_spacing)
        grad_u.append(gradient)
        
    # Compute r_ij = 0.5 * (du_i/dx_j - du_j/dx_i)
    for i in range(dim):
        for j in range(dim):
            # du_i / dx_j is the j-th component of the gradient of u_i
            du_i_dx_j = grad_u[i][..., j]
            
            # du_j / dx_i is the i-th component of the gradient of u_j
            du_j_dx_i = grad_u[j][..., i]
            
            rotation_tensor[..., i, j] = 0.5 * (du_i_dx_j - du_j_dx_i)
            
    return rotation_tensor

def compute_gradient_central(scalar_field, dx):
    """
    Computes the gradient of a scalar field using central differences.
    Returns an array of shape (..., dim) containing the gradient components.
    """
    dim = scalar_field.ndim - 1
    gradient = np.zeros_like(scalar_field)
    
    for d in range(dim):
        # Central difference along dimension d
        # Handle boundaries by clamping or using one-sided differences if necessary
        # For simplicity, assuming valid interior points or padded field
        lower = scalar_field[..., :-1] if d == dim-1 else scalar_field[..., :-1] # Simplified indexing logic for generic dim
        # Correct generic indexing for central diff
        # Shift left and right along dimension d
        left = np.roll(scalar_field, 1, axis=d)
        right = np.roll(scalar_field, -1, axis=d)
        
        # Handle boundary conditions (wrap-around or zero-gradient)
        # For robust implementation, explicit boundary handling is needed.
        # Here we assume periodic or that boundaries are not critical for the core formula.
        # A safer approach for non-periodic:
        # left = scalar_field.copy(); left[..., 0, ...] = scalar_field[..., 1, ...]
        # right = scalar_field.copy(); right[..., -1, ...] = scalar_field[..., -2, ...]
        
        # Using roll for brevity in pseudocode, but in practice, boundary guards are required [G]
        gradient[..., d] = (right - left) / (2.0 * dx)
        
    return gradient
```

**Edge Cases and Pitfalls:**
- **Large Deformations:** If $\|u_k\|$ or $\|\nabla u\|$ are not small, the linearization fails. The quadratic terms in the Lagrangian Strain Tensor do not vanish, and the Infinitesimal Rotation Tensor no longer accurately represents the physical rotation [S1, unofficial].
- **Boundary Conditions:** Central differences require neighbors on both sides. At domain boundaries, special handling (e.g., one-sided differences or padding) is required, which may reduce accuracy [G].
- **Noise:** Derivatives amplify high-frequency noise in the displacement field. Smoothing the input field before differentiation is often necessary [G].

## Subtopics

### Infinitesimal Strain Tensor
The Infinitesimal Strain Tensor is the symmetric counterpart to the Infinitesimal Rotation Tensor. It measures the relative geometric displacement (deformation) of positions in a continuum body, derived from the symmetric part of the displacement gradient [S1, unofficial].

**Definition:**
For a differentiable displacement function $u$ with $\|u_k\| \ll 1$ and $\|\nabla u\| \ll 1$, the components are defined as [S1, unofficial]:

$$
\varepsilon_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} + \frac{\partial u_j}{\partial x_i} \right)
$$

**Comparison:**
- **Infinitesimal Strain Tensor:** Captures stretching and shearing (symmetric part).
- **Infinitesimal Rotation Tensor:** Captures rigid-body rotation (antisymmetric part).
- **Lagrangian Strain Tensor:** The general form including quadratic terms $\sum_k \frac{\partial u_k}{\partial x_i} \frac{\partial u_k}{\partial x_j}$, valid for large deformations [S1, unofficial].

**When to Choose:**
Use the Infinitesimal Strain Tensor when analyzing volume changes, shearing, or stretching in a medium under small deformation assumptions. It is complementary to the Rotation Tensor; together they decompose the local flow Jacobian [S1, unofficial].

**Pitfall:**
For large deformations, the linearized Infinitesimal Strain Tensor becomes inaccurate because the quadratic term $\sum_k \frac{\partial u_k}{\partial x_i} \frac{\partial u_k}{\partial x_j}$ does not vanish [S1, unofficial]. In such cases, the Lagrangian Strain Tensor should be used instead.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels/pixels, as it involves a constant number of arithmetic operations per voxel after computing the gradient. The gradient computation itself is $O(N)$.
- **Memory Complexity:** $O(N \cdot n^2)$ where $n$ is the spatial dimension (e.g., $9N$ for 3D). This is required to store the full tensor field.
- **Strengths:** Provides a clear physical interpretation of local rotation; computationally efficient for small deformations; decomposes complex flow into intuitive components (rotation vs. strain).
- **Limitations:** Only valid for small deformations ($\|\nabla u\| \ll 1$); sensitive to noise in the displacement field due to differentiation; requires careful boundary handling.
- **When to Use:** Visualization of rotational components in fluid flow or deformation fields where small deformation assumptions hold.
- **When Not to Use:** Large deformation scenarios (use Lagrangian tensors); noisy data without prior smoothing.

## Background
The Infinitesimal Rotation Tensor originates from continuum mechanics, where it is used to describe the local rotation of a material element [S1, unofficial]. It is part of the decomposition of the displacement gradient (Jacobian) into symmetric (strain) and antisymmetric (rotation) parts. This decomposition allows for the interpretation of flow fields as a combination of absolute and relative geometric displacement, leading to changes in position, orientation, and deformation of participating media [S1, unofficial]. The specific definitions and context for volume rendering and multi-field visualization are detailed in [S1, unofficial].

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
