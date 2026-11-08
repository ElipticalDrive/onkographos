# Flow Jacobian Decomposition
Also called: Infinitesimal Strain Tensor, Strain-Rate Decomposition

## Overview
Flow Jacobian Decomposition splits the 3x3 Jacobian matrix of a displacement field into symmetric and antisymmetric components, isolating local deformation (strain) from local rotation. It builds on [Jacobian Estimation via Central Differences](2650-jacobian-estimation-via-central-differences.md) by applying a specific algebraic partition to the estimated gradient matrix. Unlike [Jacobian Normalization for ODE Step](2946-jacobian-normalization-for-ode-step.md), which scales the matrix for numerical stability, this method extracts physical kinematic properties.

## How it works
**Inputs and Outputs:**
- Input: The 3x3 Jacobian matrix $J$ of the displacement field $\mathbf{u}$, where $J_{ij} = \frac{\partial u_i}{\partial x_j}$.
- Output: The Infinitesimal Strain Tensor $\epsilon$ (symmetric) and the Infinitesimal Rotation Tensor $\omega$ (antisymmetric).

**Data Structures:**
- $J$: A 3x3 matrix of floating-point values.
- $\epsilon$: A 3x3 symmetric matrix.
- $\omega$: A 3x3 antisymmetric matrix.

**Steps:**
1. Compute the symmetric part of $J$ to obtain the strain tensor.
2. Compute the antisymmetric part of $J$ to obtain the rotation tensor.

**Formulas:**
The decomposition is defined as:
$$ J_{ij} = \epsilon_{ij} + \omega_{ij} $$
where:
$$ \epsilon_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} + \frac{\partial u_j}{\partial x_i} \right) $$
$$ \omega_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} - \frac{\partial u_j}{\partial x_i} \right) $$
Here, $\epsilon_{ij}$ represents the Infinitesimal Strain Tensor, and $\omega_{ij}$ represents the Infinitesimal Rotation Tensor [S1, unofficial].

**Parameters:**
- None. The decomposition is purely algebraic.

**Edge Cases and Pitfalls:**
- **Large Deformations:** The decomposition assumes small deformations ($\|\mathbf{u}\| \ll 1, \|\nabla \mathbf{u}\| \ll 1$). For large deformations, the Lagrangian Strain Tensor should be used instead, which includes higher-order terms: $e_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} + \frac{\partial u_j}{\partial x_i} + \sum_k \frac{\partial u_k}{\partial x_i} \frac{\partial u_k}{\partial x_j} \right)$ [S1, unofficial].
- **Precision:** Numerical noise in the Jacobian estimation can break the exact symmetry/antisymmetry. It is good practice to enforce symmetry by averaging $J_{ij}$ and $J_{ji}$ for $\epsilon$ and subtracting for $\omega$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flow_jacobian_decomposition(J):
    """
    Decomposes Jacobian J into strain (epsilon) and rotation (omega) tensors.
    J: 3x3 matrix, J[i][j] = du_i/dx_j
    """
    epsilon = [[0.0, 0.0, 0.0], [0.0, 0.0, 0.0], [0.0, 0.0, 0.0]]
    omega = [[0.0, 0.0, 0.0], [0.0, 0.0, 0.0], [0.0, 0.0, 0.0]]

    for i in range(3):
        for j in range(3):
            # Symmetric part: Strain
            epsilon[i][j] = 0.5 * (J[i][j] + J[j][i])
            # Antisymmetric part: Rotation
            omega[i][j] = 0.5 * (J[i][j] - J[j][i])

    return epsilon, omega
```

## Subtopics

### [Infinitesimal Rotation Tensor](10633-infinitesimal-rotation-tensor.md)
Describes local rotation around $\nabla \times \mathbf{u}$, representing the antisymmetric part of the displacement gradient. Choose this when isolating vorticity or rigid-body rotation components from the flow field.

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel, as it involves a fixed number of arithmetic operations on a 3x3 matrix.
- **Memory Complexity:** $O(1)$ auxiliary memory per voxel for storing the two resulting tensors.
- **Strengths:** Provides direct physical interpretation of local flow behavior (stretching vs. rotation). Computationally cheap.
- **Limitations:** Only valid for infinitesimal deformations. Does not account for volume change explicitly (though trace of strain tensor relates to it).
- **When to use:** Analyzing small-displacement fields, visualizing strain/rotation in medical imaging or fluid dynamics where deformation is minimal.
- **When not to use:** Large deformation scenarios (use Lagrangian strain) or when only the magnitude of deformation is needed without rotational context.

## Background
The decomposition originates from continuum mechanics, specifically the kinematic analysis of deformable bodies. The Infinitesimal Strain Tensor and Infinitesimal Rotation Tensor are standard definitions for differentiable displacement functions with small gradients [S1, unofficial]. It is used in multi-field visualization to separate rigid body motion from elastic deformation.

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
