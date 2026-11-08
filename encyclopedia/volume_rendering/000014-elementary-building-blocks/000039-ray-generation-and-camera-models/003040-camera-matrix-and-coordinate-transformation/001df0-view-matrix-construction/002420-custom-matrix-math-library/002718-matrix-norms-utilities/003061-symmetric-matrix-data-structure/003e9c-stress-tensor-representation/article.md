# Stress Tensor Representation

Also called: Stress Matrix, Symmetric Second-Order Tensor

## Overview
A stress tensor representation models the internal forces acting on a material point by defining the traction vector on any arbitrary cutting plane through that point [S1, book]. It is distinguished from general symmetric matrices by its specific physical interpretation of diagonal components as normal stress and off-diagonal components as shear stress, along with a defined sign convention for eigenvalues [S1, book].

## How it works
**Inputs and Outputs**
*   **Input:** A symmetric second-order tensor field (typically stored as a 3x3 symmetric matrix per voxel/point).
*   **Output:** The tensor data structure itself, enabling the calculation of traction vectors, principal stresses, and shape descriptors.

**Data Structure**
The stress tensor $\sigma$ is stored as a 3x3 symmetric matrix. Due to symmetry, only 6 unique values are required, though the full 9-element layout is often used for linear algebra operations.
*   **Layout:**
    *   Diagonal elements $\sigma_{11}, \sigma_{22}, \sigma_{33}$: Normal stress components.
    *   Off-diagonal elements $\tau_{12}, \tau_{13}, \tau_{23}$: Shear stress components.
    *   Symmetry constraint: $\sigma_{ij} = \sigma_{ji}$.

**Mathematical Formulation**
The tensor is defined as [S1, book]:
$$
\sigma = \begin{pmatrix} \sigma_{11} & \tau_{12} & \tau_{13} \\ \tau_{12} & \sigma_{22} & \tau_{23} \\ \tau_{13} & \tau_{23} & \sigma_{33} \end{pmatrix}
$$

The traction vector $t$ acting on a cutting plane with unit normal vector $n$ is calculated via matrix-vector multiplication [S1, book]:
$$
t = \sigma \cdot n
$$
This traction vector decomposes into a normal stress component $\sigma_n$ and a shear stress component $\tau$:
$$
t = \tau + \sigma_n
$$
where $\sigma_n$ is the projection of $t$ onto $n$, and $\tau$ is the orthogonal remainder.

**Parameters and Sign Convention**
*   **Eigenvalue Interpretation:** The tensor is diagonalized to find eigenvalues $\lambda_1, \lambda_2, \lambda_3$ and eigenvectors $e_i$. The eigenvectors form an orthonormal basis (principal stress axes), and eigenvalues are principal stresses [S1, book].
*   **Sign Convention:** Negative eigenvalues are interpreted as compressive forces (volume reduction), and positive eigenvalues as tensile forces (volume expansion) [S1, book].
*   **Ordering:** Eigenvalues are typically ordered such that $\lambda_1 \ge \lambda_2 \ge \lambda_3$ [S1, book].

**Edge Cases and Pitfalls**
*   **Sign Ambiguity:** In some application areas, the sign convention is reversed (positive for compression, negative for tension) [S1, book]. Implementers must verify the source's convention.
*   **Indefiniteness:** Because eigenvalues can be positive or negative, the stress tensor is indefinite [S1, book].
*   **Symmetry Assumption:** The tensor is symmetric by assumption (forces are balanced, no rotation on infinitesimal elements) [S1, book]. If asymmetry exists, this representation is invalid without modification.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_traction(stress_tensor: SymmetricMatrix3x3, normal: Vector3) -> Vector3:
    """
    Calculates the traction vector t on a plane with normal n.
    stress_tensor: 3x3 symmetric matrix [[s11, t12, t13], [t12, s22, t23], [t13, t23, s33]]
    normal: Unit vector n
    Returns: Vector t = sigma * n
    """
    s11, t12, t13 = stress_tensor[0][0], stress_tensor[0][1], stress_tensor[0][2]
    s22, t23 = stress_tensor[1][1], stress_tensor[1][2]
    s33 = stress_tensor[2][2]
    
    nx, ny, nz = normal.x, normal.y, normal.z
    
    # Matrix-vector multiplication
    tx = s11 * nx + t12 * ny + t13 * nz
    ty = t12 * nx + s22 * ny + t23 * nz
    tz = t13 * nx + t23 * ny + s33 * nz
    
    return Vector3(tx, ty, tz)

def decompose_traction(traction: Vector3, normal: Vector3) -> Tuple[float, Vector3]:
    """
    Decomposes traction t into normal stress sigma_n and shear stress tau.
    Returns: (sigma_n_magnitude, tau_vector)
    """
    # Normal stress magnitude is the projection of t onto n
    sigma_n_magnitude = dot(traction, normal)
    
    # Shear stress vector is the component of t orthogonal to n
    # tau = t - (t . n) * n
    projection = scale(normal, sigma_n_magnitude)
    tau = subtract(traction, projection)
    
    return sigma_n_magnitude, tau

def get_principal_stresses(stress_tensor: SymmetricMatrix3x3) -> Tuple[List[float], List[Vector3]]:
    """
    Computes eigenvalues and eigenvectors for the symmetric stress tensor.
    Returns: (sorted_eigenvalues, corresponding_eigenvectors)
    """
    # Standard symmetric eigenvalue decomposition
    # Note: Implementation details depend on linear algebra library
    # Ensure ordering lambda1 >= lambda2 >= lambda3
    eigenvalues, eigenvectors = symmetric_eig(stress_tensor)
    
    # Sort by eigenvalue descending
    indices = argsort_descending(eigenvalues)
    sorted_eigenvalues = [eigenvalues[i] for i in indices]
    sorted_eigenvectors = [eigenvectors[:, i] for i in indices]
    
    return sorted_eigenvalues, sorted_eigenvectors
```

## Subtopics

### Moment Tensor
A symmetric second-order tensor capturing displacement discontinuities (force couples) in inelastic deformations or seismic sources [S2, unofficial]. It is defined by components $m_{ij}$ representing the magnitude of the moment caused by a force couple in the $i$-direction acting on the $j$-direction [S2, unofficial]. It is symmetric by construction ($m_{ij} = m_{ji}$) because the total angular momentum of a seismic source is zero [S2, unofficial].

## Operating characteristics
*   **Complexity:** Eigenvalue decomposition is $O(1)$ for fixed 3x3 size, typically using analytical solutions or iterative methods.
*   **Strengths:** Provides complete mechanical state at a point; enables visualization of stress ellipsoids and Mohr's circles [S1, book].
*   **Limitations:** Assumes continuum mechanics validity (infinitesimal elements); sign convention varies by domain [S1, book].

## Background
The stress tensor is a fundamental concept in continuum mechanics. In visualization, it is used to analyze stress tensor fields, often visualized via Lame’s stress ellipsoid and Mohr’s circles [S1, book]. The moment tensor variant is central to seismic source characterization [S2, unofficial].

## Sources
[S1] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
[S2] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
