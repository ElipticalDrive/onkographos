# Infinitesimal Rotation Tensor

Also called: Antisymmetric part of displacement gradient, Local rotation tensor

## Overview
The Infinitesimal Rotation Tensor extracts the local rotational component of a flow field by isolating the antisymmetric part of the displacement gradient. It complements the Infinitesimal Strain Tensor to decompose the flow Jacobian into geometric displacement (rotation) and deformation (strain) [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: A differentiable displacement vector field $\mathbf{u}(\mathbf{x})$ defined on spatial coordinates $\mathbf{x} = (x_1, x_2, x_3)$.
- Output: A $3 \times 3$ antisymmetric tensor $\mathbf{r}$ with components $r_{ij}$.

**Data structures:**
- The tensor $\mathbf{r}$ is stored as a $3 \times 3$ matrix. Due to antisymmetry ($r_{ij} = -r_{ji}$), only the upper (or lower) triangular part and the zero diagonal need explicit storage, though the full matrix is typically computed for Jacobian decomposition.

**Formulas:**
The components of the Infinitesimal Rotation Tensor are defined as [S1, unofficial]:
$$
r_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} - \frac{\partial u_j}{\partial x_i} \right)
$$
where $u_i$ is the $i$-th component of the displacement vector $\mathbf{u}$, and $x_j$ is the $j$-th spatial coordinate.

**Relation to Curl:**
The tensor describes rotation around the axis defined by the curl of the displacement field, $\nabla \times \mathbf{u}$. Specifically, the dual vector of the antisymmetric tensor $\mathbf{r}$ is $\frac{1}{2} \nabla \times \mathbf{u}$ [S1, unofficial].

**Parameters:**
- The method assumes small deformations, requiring $\|\mathbf{u}\| \ll 1$ and $\|\nabla \mathbf{u}\| \ll 1$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_infinitesimal_rotation_tensor(displacement_field, grid_spacing):
    """
    Computes the Infinitesimal Rotation Tensor r_ij at each voxel.
    displacement_field: 3D array of shape (H, W, D, 3) containing u_x, u_y, u_z.
    grid_spacing: float, uniform spacing dx.
    Returns: 3D array of shape (H, W, D, 3, 3) containing r_ij.
    """
    H, W, D, _ = displacement_field.shape
    r = np.zeros((H, W, D, 3, 3))
    
    # Compute gradients using central differences
    # grad_u[i, j] is the derivative of u_i with respect to spatial dimension j
    # u components: 0->x, 1->y, 2->z
    # spatial dims: 0->x, 1->y, 2->z
    
    # Gradient of u_x (component 0)
    du0_dx = np.gradient(displacement_field[:,:, :, 0], axis=0) / grid_spacing
    du0_dy = np.gradient(displacement_field[:,:, :, 0], axis=1) / grid_spacing
    du0_dz = np.gradient(displacement_field[:,:, :, 0], axis=2) / grid_spacing
    
    # Gradient of u_y (component 1)
    du1_dx = np.gradient(displacement_field[:,:, :, 1], axis=0) / grid_spacing
    du1_dy = np.gradient(displacement_field[:,:, :, 1], axis=1) / grid_spacing
    du1_dz = np.gradient(displacement_field[:,:, :, 1], axis=2) / grid_spacing
    
    # Gradient of u_z (component 2)
    du2_dx = np.gradient(displacement_field[:,:, :, 2], axis=0) / grid_spacing
    du2_dy = np.gradient(displacement_field[:,:, :, 2], axis=1) / grid_spacing
    du2_dz = np.gradient(displacement_field[:,:, :, 2], axis=2) / grid_spacing
    
    # Assemble gradients into a 3x3 Jacobian-like structure for each voxel
    # J[i,j] = d(u_i)/d(x_j)
    # J[0,0]=du0_dx, J[0,1]=du0_dy, J[0,2]=du0_dz
    # J[1,0]=du1_dx, J[1,1]=du1_dy, J[1,2]=du1_dz
    # J[2,0]=du2_dx, J[2,1]=du2_dy, J[2,2]=du2_dz
    
    J = np.zeros((H, W, D, 3, 3))
    J[:,:, :, 0, 0] = du0_dx
    J[:,:, :, 0, 1] = du0_dy
    J[:,:, :, 0, 2] = du0_dz
    J[:,:, :, 1, 0] = du1_dx
    J[:,:, :, 1, 1] = du1_dy
    J[:,:, :, 1, 2] = du1_dz
    J[:,:, :, 2, 0] = du2_dx
    J[:,:, :, 2, 1] = du2_dy
    J[:,:, :, 2, 2] = du2_dz
    
    # Compute r_ij = 0.5 * (J[i,j] - J[j,i])
    # r is antisymmetric: r_ij = -r_ji, r_ii = 0
    for i in range(3):
        for j in range(3):
            r[:,:, :, i, j] = 0.5 * (J[:,:, :, i, j] - J[:,:, :, j, i])
            
    return r
```

**Edge cases and pitfalls:**
- The assumption $\|\nabla \mathbf{u}\| \ll 1$ must hold. If deformations are large, this linearization fails, and the Lagrangian Strain Tensor should be used instead [S1, unofficial].
- Boundary conditions for finite differences (e.g., central differences) require handling edges where neighbors are missing. [G] Standard practice is to use forward/backward differences at boundaries or pad the field.

## Subtopics

### Infinitesimal Strain Tensor
The symmetric part of the displacement gradient, representing linearized strain for small deformations. It is defined as $\varepsilon_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} + \frac{\partial u_j}{\partial x_i} \right)$ [S1, unofficial]. It is chosen when analyzing deformation (stretching/compression) rather than rotation. It requires the same small deformation assumptions $\|\mathbf{u}\| \ll 1$ and $\|\nabla \mathbf{u}\| \ll 1$ [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels, dominated by gradient computation.
- **Memory Complexity:** $O(N)$ to store the gradient fields and the resulting tensor.
- **Strengths:** Provides a direct geometric interpretation of local rotation via the curl of the displacement field.
- **Limitations:** Only valid for infinitesimal deformations. Not suitable for large rigid body rotations or significant stretching [S1, unofficial].

## Background
The Infinitesimal Rotation Tensor originates from continuum mechanics and linearized elasticity theory. It is part of the decomposition of the flow Jacobian into strain and rotation components, allowing flow fields to be interpreted as combinations of absolute/relative geometric displacement, orientation change, and deformation [S1, unofficial].

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
