# Analytical Tensor Glyph for Rank Two Tensors in a Plane

Also called: Analytic tensor glyph visualization, Analytical Symmetric Tensor Glyph Generation, Analytical Tensor Glyph Visualization in a Plane, Rank two tensor component derivation in a plane

## Overview
This method provides a direct analytical expression for visualizing rank-two tensors in a 2D plane, generating omnidirectional graphical representations (glyphs) that are accurate and unambiguous [S1]. Unlike numerical interpolation methods which may fail to preserve tensor invariants, this approach satisfies strict requirements including invariance under isometric domain transformation, scaling invariance, direct encoding of real eigenvalues/eigenvectors, glyph uniqueness, and continual glyph changes [S1]. It overcomes the numerical constraints of existing imaging methods by analytically solving for tensor components at arbitrary rotation angles using eigenvalues and invariants [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A rank-two tensor matrix $B$ in 2D space, defined by four components $b_{ij}$ [S1].
*   **Output:** A set of symmetric tensor components $(a_{11}, a_{12}, a_{21}, a_{22})$ for any rotation angle $\phi$, allowing the construction of a glyph representing the tensor's shape in all directions [S1].

**Data Structures**
The tensor is represented as a $2 \times 2$ matrix. For a general tensor $B$:
$$ B = \begin{bmatrix} b_{11} & b_{12} \\ b_{21} & b_{22} \end{bmatrix} $$
The method decomposes $B$ into a symmetric part $T_S$ and an antisymmetric part $T_A$ [S1]:
$$ T_S = \begin{bmatrix} a_{11} & a_{12} \\ a_{21} & a_{22} \end{bmatrix}, \quad T_A = \begin{bmatrix} 0 & c_{12} \\ c_{21} & 0 \end{bmatrix} $$
where $a_{ij}$ are symmetric components and $c_{ij}$ are antisymmetric components [S1].

**Steps**
1.  **Decomposition:** Decompose the general tensor $B$ into symmetric $T_S$ and antisymmetric $T_A$ parts [S1]:
    $$ a_{ij} = \frac{1}{2}(b_{ij} + b_{ji}) $$
    $$ c_{ij} = \frac{1}{2}(b_{ij} - b_{ji}) $$
    The method focuses the analytical calculation solely on the symmetric tensor $T_S$, as the antisymmetric part $T_A$ does not depend on the symmetric part's size and simplifies the visualization [S1].

2.  **Calculate Invariants:** Compute the first invariant (trace $m$), second invariant (determinant $n$), and the third invariant (squared Frobenius norm $l$) from the symmetric components [S1]:
    $$ m = a_{11} + a_{22} $$
    $$ n = a_{11}a_{22} - a_{12}a_{21} $$
    $$ l = a_{11}^2 + a_{22}^2 + 2a_{12}a_{21} = m^2 - 2n $$
    The third invariant $l$ corresponds to the square of the initial vector's length and is necessary for the analytical calculation [S1].

3.  **Solve for Eigenvalues:** Solve the characteristic quadratic equation to find eigenvalues $\lambda_1$ and $\lambda_2$ [S1]:
    $$ \lambda^2 - m\lambda + n = 0 $$
    $$ \lambda_{1,2} = \frac{m \pm \sqrt{m^2 - 4n}}{2} $$
    To ensure real solutions, enforce $\lambda_1 \ge \lambda_2$. If the discriminant is negative, the roots are complex; the method requires swapping eigenvalues to satisfy $\lambda_1 > \lambda_2$ to ensure real solutions [S1].

4.  **Determine Initial Angle:** Calculate the initial angle $\alpha$ for the vector $r_0(\lambda_1, \lambda_2)$ [S1]:
    $$ \alpha = \arctan\left(\frac{\lambda_2}{\lambda_1}\right) \quad \text{for } \alpha \in \left[-\frac{\pi}{4}, \frac{\pi}{4}\right] $$
    Typical values for the initial state are $\alpha = 0^\circ$ or $45^\circ$ [S1]. At $\alpha = 0^\circ$, the maximum values of tensor component parts in the x-axis direction display the golden section ratio $\phi \approx 0.618$ [S1].

5.  **Calculate Rotation Angle:** Determine the rotation angle $\phi$ of the coordinate system using the eigenvector $u(u_1, u_2)$ of the tensor [S1]:
    $$ \tan \phi = \frac{u_1}{u_2} = \frac{-(a_{11} - \lambda_1)}{a_{12}} $$

6.  **Compute Rotated Components:** Solve the quadratic equation for the component $a_{22}$ at the rotation angle $\phi$ [S1]:
    $$ a_{22}^{1,2} = \frac{(2\lambda_2 \cot^2 \phi + m) \mp \sqrt{(2\lambda_2 \cot^2 \phi + m)^2 - 4(1 + \cot^2 \phi)(n + \lambda_2^2 \cot^2 \phi)}}{2(1 + \cot^2 \phi)} $$
    The quadratic yields two roots for $a_{22}$; one corresponds to the initial tensor state and the other to the state rotated by angle $\phi$. The root corresponding to the rotated state is selected [S1].

7.  **Derive Remaining Components:** Calculate the remaining symmetric components using the solved $a_{22}$ [S1]:
    $$ a_{11} = \lambda_1 + \lambda_2 - a_{22} $$
    $$ a_{21} = \cot \phi (\lambda_1 - a_{11}) $$
    $$ a_{12} = a_{21} $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def analytical_tensor_glyph(tensor_matrix, rotation_angle_phi):
    """
    Computes symmetric rank-two tensor components at a given rotation angle phi.
    
    Args:
        tensor_matrix: 2x2 list of lists [[b11, b12], [b21, b22]]
        rotation_angle_phi: float, rotation angle in radians
        
    Returns:
        list: [a11, a12, a21, a22] components of the symmetric tensor at the rotated angle
    """
    b11, b12 = tensor_matrix[0]
    b21, b22 = tensor_matrix[1]
    
    # Step 1: Decompose into symmetric part
    a11_init = b11
    a22_init = b22
    a12_init = 0.5 * (b12 + b21)
    a21_init = a12_init
    
    # Step 2: Calculate Invariants
    m = a11_init + a22_init
    n = a11_init * a22_init - a12_init * a21_init
    
    # Step 3: Solve for Eigenvalues
    discriminant = m**2 - 4 * n
    sqrt_disc = math.sqrt(max(0.0, discriminant))
    lambda1 = (m + sqrt_disc) / 2.0
    lambda2 = (m - sqrt_disc) / 2.0
    
    # Ensure lambda1 >= lambda2
    if lambda1 < lambda2:
        lambda1, lambda2 = lambda2, lambda1

    # Step 5 & 6: Compute a22 at rotation angle phi
    sin_phi = math.sin(rotation_angle_phi)
    if abs(sin_phi) < 1e-9:
        # At phi = 0 or pi, cot(phi) is undefined; a22 remains the initial state value
        a22_rotated = a22_init
        cot_phi = 0.0
    else:
        cot_phi = math.cos(rotation_angle_phi) / sin_phi
        cot2 = cot_phi ** 2
        A = 1.0 + cot2
        B = -(2.0 * lambda2 * cot2 + m)
        C = n + lambda2**2 * cot2
        
        disc_a22 = B**2 - 4.0 * A * C
        sqrt_disc_a22 = math.sqrt(max(0.0, disc_a22))
        
        # Two roots: one for initial state, one for rotated state
        root_initial = (-B - sqrt_disc_a22) / (2.0 * A)
        root_rotated = (-B + sqrt_disc_a22) / (2.0 * A)
        
        # Select the root corresponding to the rotated state
        if abs(root_initial - a22_init) < 1e-6:
            a22_rotated = root_rotated
        else:
            a22_rotated = root_initial

    # Step 7: Derive remaining components
    a11_rotated = lambda1 + lambda2 - a22_rotated
    a21_rotated = cot_phi * (lambda1 - a11_rotated)
    a12_rotated = a21_rotated
    
    return [a11_rotated, a12_rotated, a21_rotated, a22_rotated]
```

## Subtopics

### General Tensor Decomposition and Glyph Generation
This variant extends the analytical method to general (non-symmetric) 2D tensors by combining the symmetric and antisymmetric parts [S1]. It calculates the invariants for both parts separately and combines them: $n_G = n_S + n_A$, $m_G = m_S + m_A$, and $l_G = l_S + l_A$ [S1]. The antisymmetric components are calculated as $c_{12} = \frac{1}{2}(b_{12} - b_{21})$ and $c_{21} = -c_{12}$ [S1]. For the rotational matrix component of a general tensor, the initial angle is typically set to $\alpha = 45^\circ$ [S1]. This allows the visualization of general tensors while maintaining the analytical accuracy of the symmetric part [S1].

## Operating characteristics
**Time and Memory Complexity**
The method involves constant-time algebraic operations (quadratic formula, trigonometric functions) per tensor evaluation [S1]. Memory complexity is $O(1)$ as it only stores a few scalar values (invariants, eigenvalues, components) [S1].

**Measured Performance and Scaling**
The method is applied to in-depth analysis of dynamic processes where numerical methods fail due to computation constraints [S1]. It provides higher accuracy per point than interpolation-based methods by strictly preserving tensor invariants [S1].

**Strengths and Limitations**
*   **Strengths:** Accurate and unambiguous; satisfies all tensor properties including invariance under isometric domain transformation; avoids numerical interpolation errors; provides omnidirectional representation [S1].
*   **Limitations:** Restricted to 2D plane (rank-two tensors); requires handling of complex roots via eigenvalue swapping; 2D space lacks a third invariant (requiring the derived Frobenius norm invariant); 3D extension is not directly provided due to the missing fourth invariant [S1].

**When to Use**
Use this method when high-fidelity, invariant-preserving visualization of 2D symmetric or general rank-two tensors is required, particularly in dynamic processes where numerical stability is critical [S1]. Avoid when 3D visualization is needed or when computational resources are extremely limited and approximate glyph shapes are acceptable [S1].

## Background
The method was proposed by T. Stejskal, J. Svetlík, and Dobrán in 2022, published in *Machines* [S1]. It addresses the limitations of existing numerical methods for tensor rendering, which often fail to preserve tensor invariants or unambiguously represent all properties [S1]. The work introduces a new analytical approach to deriving rank-two tensor components in a plane and defines a new tensor invariant (the squared Frobenius norm) necessary for the calculation [S1]. It also highlights the presence of the golden ratio in the representation of planar tensors with a zero initial angle [S1].

## Sources
[S1] An Analytical Method for Tensor Visualization in a Plane, https://doi.org/10.3390/machines10020089, fetched 2026-10-27
