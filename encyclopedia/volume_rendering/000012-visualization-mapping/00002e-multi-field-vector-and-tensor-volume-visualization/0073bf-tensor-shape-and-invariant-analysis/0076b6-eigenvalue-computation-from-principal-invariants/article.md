# Eigenvalue computation from principal invariants

Also called: Cubic eigenvalue solver, Invariant-based eigenvalue extraction

## Overview
This method computes the three eigenvalues of a symmetric $3 \times 3$ diffusion tensor directly from its principal invariants, avoiding explicit matrix diagonalization [S1, unofficial]. It leverages the fact that tensor eigenvalues are invariant under coordinate rotation, allowing the characteristic cubic polynomial to be solved analytically using only arithmetic operations and trigonometric functions [S1, unofficial]. Unlike direct diagonalization, which requires eigenvector computation, this approach is computationally cheaper and inherently sorts the eigenvalues in descending order due to the standard branch choice of the arccosine function [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A symmetric $3 \times 3$ tensor $\mathbf{D}$ represented in the laboratory frame, with components $D_{xx}, D_{yy}, D_{zz}$ (diagonal) and $D_{xy}, D_{xz}, D_{yz}$ (off-diagonal).
- Output: Three scalar eigenvalues $\lambda_1, \lambda_2, \lambda_3$, sorted such that $\lambda_1 \ge \lambda_2 \ge \lambda_3$.

**Data Structures:**
The tensor $\mathbf{D}$ is typically stored as a flat array of 6 or 9 floating-point values. For a symmetric tensor, only the 6 unique components are required:
- `D[0]`: $D_{xx}$
- `D[1]`: $D_{yy}$
- `D[2]`: $D_{zz}$
- `D[3]`: $D_{xy}$
- `D[4]`: $D_{xz}$
- `D[5]`: $D_{yz}$

**Steps:**
1. **Compute Principal Invariants:** Calculate $J_1, J_2, J_3$ from the tensor components.
2. **Compute Intermediate Invariants:** Calculate $Q$ and $R$ using combinations of $J_i$.
3. **Compute Angle:** Calculate $\Theta$ using the arccosine formula.
4. **Compute Eigenvalues:** Calculate $\lambda_1, \lambda_2, \lambda_3$ using cosine formulas.

**Formulas:**
The principal invariants are defined as:
$$ J_1 = \text{tr}(\mathbf{D}) = D_{xx} + D_{yy} + D_{zz} $$
$$ J_2 = D_{xx}D_{yy} + D_{xx}D_{zz} + D_{yy}D_{zz} - D_{xy}^2 - D_{xz}^2 - D_{yz}^2 $$
$$ J_3 = 2D_{xy}D_{xz}D_{yz} + D_{xx}D_{yy}D_{zz} - D_{xz}^2D_{yy} - D_{xx}D_{yz}^2 - D_{xy}^2D_{zz} $$

Alternatively, $J_2$ can be computed using the trace of the square of the tensor:
$$ J_2 = \frac{\text{tr}(\mathbf{D})^2 - \text{tr}(\mathbf{D}^2)}{2} $$
where $\text{tr}(\mathbf{D}^2) = D_{xx}^2 + D_{yy}^2 + D_{zz}^2 + 2D_{xy}^2 + 2D_{xz}^2 + 2D_{yz}^2$.

The squared norm invariant $J_4$ is useful for computing $Q$ and $R$ with potentially better numerical stability or simpler expressions:
$$ J_4 = \|\mathbf{D}\|^2 = D_{xx}^2 + 2D_{xy}^2 + 2D_{xz}^2 + D_{yy}^2 + 2D_{yz}^2 + D_{zz}^2 $$

The intermediate invariants $Q$ and $R$ are:
$$ Q = \frac{J_1^2 - 3J_2}{9} = \frac{3J_4 - J_1^2}{18} $$
$$ R = \frac{-9J_1J_2 + 27J_3 + 2J_1^3}{54} = \frac{-5J_1J_2 + 27J_3 + 2J_1J_4}{54} $$

The angle $\Theta$ is:
$$ \Theta = \frac{1}{3} \cos^{-1} \left( \frac{R}{\sqrt{Q^3}} \right) $$

The eigenvalues are:
$$ \lambda_1 = \frac{J_1}{3} + 2\sqrt{Q} \cos(\Theta) $$
$$ \lambda_2 = \frac{J_1}{3} + 2\sqrt{Q} \cos\left(\Theta - \frac{2\pi}{3}\right) $$
$$ \lambda_3 = \frac{J_1}{3} + 2\sqrt{Q} \cos\left(\Theta + \frac{2\pi}{3}\right) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_eigenvalues_from_invariants(D):
    """
    Computes eigenvalues of a symmetric 3x3 tensor D from its components.
    D is a list/tuple of 6 elements: [Dxx, Dyy, Dzz, Dxy, Dxz, Dyz]
    Returns sorted eigenvalues [lambda1, lambda2, lambda3] with lambda1 >= lambda2 >= lambda3.
    """
    Dxx, Dyy, Dzz, Dxy, Dxz, Dyz = D
    
    # Step 1: Compute Principal Invariants
    J1 = Dxx + Dyy + Dzz
    
    # J2 calculation using component-wise formula
    J2 = (Dxx * Dyy + Dxx * Dzz + Dyy * Dzz 
          - Dxy**2 - Dxz**2 - Dyz**2)
    
    # J3 calculation using component-wise formula
    J3 = (2 * Dxy * Dxz * Dyz 
          + Dxx * Dyy * Dzz 
          - Dxz**2 * Dyy 
          - Dxx * Dyz**2 
          - Dxy**2 * Dzz)
          
    # Compute J4 (squared norm) for Q and R calculation
    J4 = (Dxx**2 + Dyy**2 + Dzz**2 
          + 2 * Dxy**2 + 2 * Dxz**2 + 2 * Dyz**2)
          
    # Step 2: Compute Intermediate Invariants Q and R
    # Using the forms involving J4 for consistency with source [S1, unofficial]
    Q = (3 * J4 - J1**2) / 18.0
    
    R = (-5 * J1 * J2 + 27 * J3 + 2 * J1 * J4) / 54.0
    
    # Step 3: Compute Theta
    # Guard against numerical errors causing argument > 1 or < -1
    arg = R / (Q**1.5)
    if arg > 1.0:
        arg = 1.0
    elif arg < -1.0:
        arg = -1.0
        
    Theta = (1.0 / 3.0) * math.acos(arg)
    
    # Step 4: Compute Eigenvalues
    mean = J1 / 3.0
    radius = 2 * math.sqrt(Q)
    
    cos_theta = math.cos(Theta)
    cos_theta_minus_2pi3 = math.cos(Theta - 2 * math.pi / 3)
    cos_theta_plus_2pi3 = math.cos(Theta + 2 * math.pi / 3)
    
    lambda1 = mean + radius * cos_theta
    lambda2 = mean + radius * cos_theta_minus_2pi3
    lambda3 = mean + radius * cos_theta_plus_2pi3
    
    # The standard branch choice of acos in [0, pi] ensures Theta in [0, pi/3]
    # which inherently sorts lambda1 >= lambda2 >= lambda3.
    # However, floating point inaccuracies might slightly violate this.
    # Sorting ensures correct order for downstream applications.
    eigenvalues = sorted([lambda1, lambda2, lambda3], reverse=True)
    
    return eigenvalues
```

**Parameters:**
- No tunable parameters. The method is deterministic based on the input tensor components.

**Edge Cases and Pitfalls:**
- **Numerical Stability:** When $Q$ is very small or zero, the term $\sqrt{Q^3}$ in the denominator of the arccosine argument may cause division by zero or instability. If $Q=0$, the tensor is spherical ($\lambda_1=\lambda_2=\lambda_3$), and all eigenvalues are equal to $J_1/3$ [S1, unofficial].
- **Argument Clipping:** Due to floating-point precision, the value $R/\sqrt{Q^3}$ may slightly exceed $[-1, 1]$. It must be clipped to this range before passing to $\cos^{-1}$ [G].
- **Sorting:** The standard branch choice $\cos^{-1}(x) \in [0, \pi]$ bounds $\Theta$ to $[0, \pi/3]$, which inherently sorts the eigenvalues in descending order ($\lambda_1 \ge \lambda_2 \ge \lambda_3$) [S1, unofficial]. A different branch choice yields a different fixed ordering.
- **Degenerate Cases:** When $\Theta = \pi/3$, $\lambda_1 = \lambda_2$ (minimal skewness). When $\Theta = 0$, $\lambda_2 = \lambda_3$ (maximal skewness) [S1, unofficial].

## Subtopics

### Principal Invariant Computation
This variant focuses specifically on the calculation of the three principal invariants $J_1, J_2, J_3$ of a symmetric diffusion tensor $\mathbf{D}$ directly from its laboratory-frame matrix components, serving as precursors to solving for eigenvalues [S1, unofficial]. It involves evaluating the characteristic polynomial $\det(\lambda\mathbf{I} - \mathbf{D}) = \lambda^3 - J_1\lambda^2 + J_2\lambda - J_3 = 0$ in the laboratory frame [S1, unofficial]. Computing invariants first avoids unnecessary computational expense compared to directly solving for individual eigenvalues, as the moments are precursors to the eigenvalue computation [S1, unofficial]. It is chosen when only the shape characteristics (derived from invariants) are needed, or as a necessary first step for the full eigenvalue solver.

## Operating characteristics
- **Time Complexity:** $O(1)$ per tensor. The method involves a fixed number of arithmetic operations and one trigonometric function evaluation. It is generally faster than iterative diagonalization methods and avoids the overhead of eigenvector computation.
- **Memory Complexity:** $O(1)$. Requires storage for a few temporary scalar variables ($J_1, J_2, J_3, Q, R, \Theta$).
- **Measured Performance:** Not given in the sources.
- **Strengths:** Avoids eigenvector computation; uses only arithmetic and trigonometric functions; inherently sorts eigenvalues; numerically stable for symmetric tensors if implemented with care (e.g., clipping arccos argument).
- **Limitations:** Requires the tensor to be symmetric; potential numerical instability if $Q$ is near zero; does not provide eigenvectors (which may be needed for some visualization techniques like glyph orientation).
- **When to use:** When only eigenvalues are needed for transfer function mapping or shape analysis, and computational efficiency is critical.
- **When not to use:** When eigenvectors are required, or when the tensor is not symmetric (though diffusion tensors are theoretically symmetric, numerical errors in estimation might break symmetry, requiring symmetrization before application).

## Background
This method originates from the analytical solution of cubic polynomials, adapted for tensor eigenvalue computation in the context of Diffusion Tensor Imaging (DTI) and visualization [S1, unofficial]. The symmetry of diffusion tensors is a nontrivial result of statistical physics, explained by Onsager’s principle of “microscopic reversibility” [S1, unofficial]. The specific formulation using principal invariants $J_1, J_2, J_3$ and the intermediate variables $Q, R$ follows standard formulas for solving cubic equations, as adapted in the literature on tensor visualization [S1, unofficial]. The "eigenvalue wheel" geometric analogy was introduced to provide an intuitive understanding of the degrees of freedom in tensor shape [S1, unofficial].

## Sources
[S1] Visualization and Analysis of Diffusion Tensor Fields Abstract, http://people.cs.uchicago.edu/~glk/pubs/pdf/Kindlmann-VisualizationAndAnalysisOfDiffusionTensorFields-PhD-2004.pdf, fetched 2026-10-30 (unofficial)
