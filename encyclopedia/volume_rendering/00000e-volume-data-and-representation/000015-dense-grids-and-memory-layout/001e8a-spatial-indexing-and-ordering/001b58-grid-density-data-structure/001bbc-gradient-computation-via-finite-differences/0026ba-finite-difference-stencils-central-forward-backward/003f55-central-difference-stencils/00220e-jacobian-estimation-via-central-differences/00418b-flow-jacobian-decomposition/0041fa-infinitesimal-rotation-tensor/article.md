# Infinitesimal Rotation Tensor

Also called: Antisymmetric part of displacement gradient, Local rotation tensor

## Overview
The Infinitesimal Rotation Tensor is a second-order tensor that quantifies the local rigid-body rotation within a continuous medium, derived from the antisymmetric part of the displacement gradient [S1, unofficial]. It decomposes the flow Jacobian into rotational and deformational components, distinguishing orientation changes from geometric distortion [S1, unofficial]. Unlike the Infinitesimal Strain Tensor, which captures symmetric stretching and shearing, this tensor isolates the rotational component directly related to the curl of the displacement field [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A differentiable displacement function $u: \Omega \subseteq \mathbb{R}^n \rightarrow \mathbb{R}^n$, where $\Omega$ is the domain of the continuum body [S1, unofficial].
- **Output:** The Infinitesimal Rotation Tensor $r$, an antisymmetric tensor with components $r_{ij}$ [S1, unofficial].

**Data Structures:**
- The tensor is stored as a dense grid of $n \times n$ matrices aligned with spatial coordinates, where each element $r_{ij}$ holds the partial derivative combination for the $i$-th and $j$-th directions [S1, unofficial].

**Formulas:**
The components are defined as [S1, unofficial]:
$$r_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} - \frac{\partial u_j}{\partial x_i} \right)$$
Where:
- $u_i$ is the $i$-th component of the displacement vector field $u$ [S1, unofficial].
- $x_j$ is the $j$-th spatial coordinate [S1, unofficial].
- $\frac{\partial u_i}{\partial x_j}$ is the partial derivative of the $i$-th displacement component with respect to the $j$-th spatial coordinate [S1, unofficial].

This tensor describes a rotation around the axis defined by the curl of the displacement field, $\nabla \times u$ [S1, unofficial].

**Parameters:**
- The method assumes small deformations, specifically requiring $\|u_k\| \ll 1$ and $\|\nabla u\| \ll 1$ [S1, unofficial].

**Steps:**
1. **Compute Displacement Gradient:** Calculate the Jacobian matrix $J$ of the displacement function $u$, where $J_{ij} = \frac{\partial u_i}{\partial x_j}$ [S1, unofficial].
2. **Extract Antisymmetric Part:** Compute the tensor components using the formula above, equivalent to $r = \frac{1}{2}(J - J^T)$ [S1, unofficial].
3. **Interpretation:** The resulting tensor $r$ represents the local rotation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_infinitesimal_rotation_tensor(u, dx):
    """
    Computes the Infinitesimal Rotation Tensor from a displacement vector field.
    u: 3D list of shape (N, N, N, 3) representing u(x,y,z).
    dx: float grid spacing.
    Returns: 3D list of shape (N, N, N, 3, 3) for r_ij.
    """
    N = len(u)
    dim = 3
    
    # Initialize output tensor grid
    r = [[[[[0.0]*dim for _ in range(dim)] for _ in range(N)] for _ in range(N)] for _ in range(N)]
    
    # Precompute gradients for each displacement component
    # grad[i][d][x][y][z] = du_i/dx_d
    grad = []
    for i in range(dim):
        # Extract scalar field for component u_i
        s = [[[u[x][y][z][i] for z in range(N)] for y in range(N)] for x in range(N)]
        comp_grad = []
        for d in range(dim):
            g = [[[0.0]*N for _ in range(N)] for _ in range(N)]
            for x in range(N):
                for y in range(N):
                    for z in range(N):
                        # Central difference with boundary clamping [G]
                        if d == 0:
                            left_idx, right_idx = max(0, x-1), min(N-1, x+1)
                            left_val, right_val = s[left_idx][y][z], s[right_idx][y][z]
                        elif d == 1:
                            left_idx, right_idx = max(0, y-1), min(N-1, y+1)
                            left_val, right_val = s[x][left_idx][z], s[x][right_idx][z]
                        else:
                            left_idx, right_idx = max(0, z-1), min(N-1, z+1)
                            left_val, right_val = s[x][y][left_idx], s[x][y][right_idx]
                        g[x][y][z] = (right_val - left_val) / (2.0 * dx)
            comp_grad.append(g)
        grad.append(comp_grad)
        
    # Compute r_ij = 0.5 * (du_i/dx_j - du_j/dx_i)
    for x in range(N):
        for y in range(N):
            for z in range(N):
                for i in range(dim):
                    for j in range(dim):
                        r[x][y][z][i][j] = 0.5 * (grad[i][j][x][y][z] - grad[j][i][x][y][z])
    return r
```

**Edge Cases and Pitfalls:**
- **Large Deformations:** If $\|u_k\|$ or $\|\nabla u\|$ are not small, the linearization fails [S1, unofficial].
- **Boundary Conditions:** Central differences require neighbors on both sides; clamping or one-sided differences are used at boundaries [G].
- **Noise:** Derivatives amplify high-frequency noise; smoothing the input field before differentiation is recommended [G].

## Subtopics

### Infinitesimal Strain Tensor
The Infinitesimal Strain Tensor is the symmetric counterpart to the Infinitesimal Rotation Tensor. It measures the relative geometric displacement (deformation) of positions in a continuum body, derived from the symmetric part of the displacement gradient [S1, unofficial].

**Definition:**
For a differentiable displacement function $u$ with $\|u_k\| \ll 1$ and $\|\nabla u\| \ll 1$, the components are defined as [S1, unofficial]:
$$\varepsilon_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} + \frac{\partial u_j}{\partial x_i} \right)$$

**Comparison:**
- **Infinitesimal Strain Tensor:** Captures stretching and shearing (symmetric part) [S1, unofficial].
- **Infinitesimal Rotation Tensor:** Captures rigid-body rotation (antisymmetric part) [S1, unofficial].
- **Lagrangian Strain Tensor:** The general form including quadratic terms $\sum_k \frac{\partial u_k}{\partial x_i} \frac{\partial u_k}{\partial x_j}$, valid for large deformations [S1, unofficial].

**When to Choose:**
Use the Infinitesimal Strain Tensor when analyzing volume changes, shearing, or stretching in a medium under small deformation assumptions. It is complementary to the Rotation Tensor; together they decompose the local flow Jacobian [S1, unofficial].

**Pitfall:**
For large deformations, the linearized Infinitesimal Strain Tensor becomes inaccurate because the quadratic term $\sum_k \frac{\partial u_k}{\partial x_i} \frac{\partial u_k}{\partial x_j}$ does not vanish [S1, unofficial]. In such cases, the Lagrangian Strain Tensor should be used instead.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Provides a clear physical interpretation of local rotation; computationally efficient for small deformations; decomposes complex flow into intuitive components (rotation vs. strain) [G].
- **Limitations:** Only valid for small deformations ($\|\nabla u\| \ll 1$); sensitive to noise in the displacement field due to differentiation; requires careful boundary handling [G].
- **When to Use:** Visualization of rotational components in fluid flow or deformation fields where small deformation assumptions hold [G].
- **When Not to Use:** Large deformation scenarios (use Lagrangian tensors); noisy data without prior smoothing [G].

## Background
The Infinitesimal Rotation Tensor originates from continuum mechanics, where it is used to describe the local rotation of a material element [S1, unofficial]. It is part of the decomposition of the displacement gradient (Jacobian) into symmetric (strain) and antisymmetric (rotation) parts. This decomposition allows for the interpretation of flow fields as a combination of absolute and relative geometric displacement, leading to changes in position, orientation, and deformation of participating media [S1, unofficial].

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
