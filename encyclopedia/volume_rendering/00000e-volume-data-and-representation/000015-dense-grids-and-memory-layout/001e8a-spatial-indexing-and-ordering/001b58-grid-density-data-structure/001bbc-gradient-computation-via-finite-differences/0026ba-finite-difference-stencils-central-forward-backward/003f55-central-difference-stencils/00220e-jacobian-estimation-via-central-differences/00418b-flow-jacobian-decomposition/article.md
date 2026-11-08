# Flow Jacobian Decomposition

Also called: Infinitesimal Rotation Tensor, Infinitesimal Strain Tensor, Strain-Rate Decomposition

## Overview
Flow Jacobian Decomposition splits the 3x3 Jacobian matrix of a displacement field into symmetric and antisymmetric components to isolate local volume/shape deformation from local rigid rotation. It builds on [Jacobian Estimation via Central Differences](2650-jacobian-estimation-via-central-differences.md) by interpreting the estimated gradient $\nabla u$ through continuum mechanics, distinguishing it from sibling methods that normalize the Jacobian for numerical stability [S1, unofficial].

## How it works
**Inputs:** A 3D vector field $u(x,y,z)$ representing displacement, and its 3x3 Jacobian matrix $J = \nabla u$ estimated via central differences.
**Outputs:** Two 3x3 tensors: the Infinitesimal Strain Tensor $\epsilon$ and the Infinitesimal Rotation Tensor $\omega$.

**Data Structures:**
- $J$: $3 \times 3$ float matrix, $J_{ij} = \frac{\partial u_i}{\partial x_j}$.
- $\epsilon$: $3 \times 3$ symmetric float matrix.
- $\omega$: $3 \times 3$ antisymmetric float matrix.

**Steps:**
1. Compute the Jacobian $J$ at the point of interest using central differences [S1, unofficial].
2. Extract the symmetric part to form the Infinitesimal Strain Tensor $\epsilon$:
   $$ \epsilon_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} + \frac{\partial u_j}{\partial x_i} \right) $$
3. Extract the antisymmetric part to form the Infinitesimal Rotation Tensor $\omega$:
   $$ \omega_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} - \frac{\partial u_j}{\partial x_i} \right) $$
4. Verify decomposition: $J = \epsilon + \omega$.

**Parameters:**
- Validity requires small deformations: $\|u\| \ll 1$ and $\|\nabla u\| \ll 1$ [S1, unofficial].

**Pitfalls:**
- If $\|\nabla u\|$ is not small, this linearization fails to represent large rigid rotations or significant non-linear strain [S1, unofficial].
- Numerical noise in central differences can introduce non-zero diagonal elements in $\omega$ or non-zero off-diagonal asymmetry in $\epsilon$; projection to symmetric/antisymmetric subspaces mitigates this.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flow_jacobian_decomposition(jacobian_matrix):
    """
    Decomposes a 3x3 Jacobian matrix into Infinitesimal Strain and Rotation tensors.
    
    Args:
        jacobian_matrix: 3x3 list of lists or numpy array, J_ij = du_i/dx_j
        
    Returns:
        strain_tensor: 3x3 symmetric matrix
        rotation_tensor: 3x3 antisymmetric matrix
    """
    # Initialize tensors
    strain = [[0.0, 0.0, 0.0], [0.0, 0.0, 0.0], [0.0, 0.0, 0.0]]
    rotation = [[0.0, 0.0, 0.0], [0.0, 0.0, 0.0], [0.0, 0.0, 0.0]]
    
    for i in range(3):
        for j in range(3):
            # Symmetric part: Infinitesimal Strain Tensor
            strain[i][j] = 0.5 * (jacobian_matrix[i][j] + jacobian_matrix[j][i])
            
            # Antisymmetric part: Infinitesimal Rotation Tensor
            rotation[i][j] = 0.5 * (jacobian_matrix[i][j] - jacobian_matrix[j][i])
            
    return strain, rotation
```

## Subtopics

### Infinitesimal Rotation Tensor
Describes local rotation around the curl of the displacement field ($\nabla \times u$). It is the antisymmetric part of the displacement gradient. Choose this when isolating rigid-body rotation components from deformation is required. It is directly related to the vorticity of the flow field [S1, unofficial].

## Operating characteristics
**Time Complexity:** $O(1)$ per voxel, involving 9 additions and 9 subtractions.
**Memory Complexity:** $O(1)$ auxiliary storage per voxel.
**Strengths:** Physically interpretable separation of deformation and rotation; computationally cheap.
**Limitations:** Only valid for infinitesimal deformations; fails for large rotations or significant non-linear strain.
**When to use:** Analyzing small-displacement volume data, such as medical imaging of tissue strain or fluid flow visualization where local rotation is distinct from expansion/contraction.

## Background
Originated in continuum mechanics for analyzing non-rigid body behavior. Defined in Definition 3.19 and 3.19 of [S1, unofficial] as linearizations of the Lagrangian Strain Tensor for small deformations. Used in multi-field visualization to interpret flow fields as combinations of geometric displacement and orientation change [S1, unofficial].

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
