# Regularization of Non-Differentiable Quadratic Criterion

Also called: Regularization for Piecewise Constant Model

## Overview
This method addresses the non-differentiability of the quadratic criterion used in 1D slice reconstruction with a piecewise constant model [S1, unofficial]. It applies regularization techniques to handle the optimization problem where the objective function is not smooth, enabling stable parameter estimation [S1, unofficial].

## How it works
**Inputs:**
- Measured projection data vector $y$.
- Forward operator matrix $F$ mapping the unknown radial field $x$ to the projection space.
- Regularization weight parameter $\mu_2$.

**Outputs:**
- Reconstructed radial field vector $x$.

**Data Structures:**
- $x$: Vector of size $n$ (number of parameters in the 1D piecewise constant model).
- $F$: Matrix of size $m \times n$ (where $m$ is the number of projections).
- $L_2$: Second-order finite-difference operator matrix of size $n \times n$.

**Formulas:**
The objective function $\Phi(x)$ combines data fidelity and a smoothness penalty [S2, unofficial]:
$$
\Phi(x) = \|Fx - y\|_2^2 + \mu_2^2 \|L_2 x\|_2^2
$$
where:
- $\|Fx - y\|_2^2$ is the data fidelity term.
- $\mu_2^2 \|L_2 x\|_2^2$ is the Tikhonov regularization term [S2, unofficial].
- $L_2$ is defined as:
$$
L_2 = \frac{1}{\Delta r^2} \begin{bmatrix} 1 & -2 & 1 & 0 & \cdots & 0 \\ 0 & 1 & -2 & 1 & \cdots & 0 \\ \vdots & & & \ddots & & \vdots \\ 0 & \cdots & 0 & 1 & -2 & 1 \end{bmatrix}
$$
- $\Delta r$ is the radial step size [S2, unofficial].

**Steps:**
1. Expand the objective function terms into quadratic forms [S2, unofficial]:
   $$
   \|Fx - y\|_2^2 = x^T F^T Fx - 2x^T F^T y + y^T y
   $$
   $$
   \mu_2^2 \|L_2 x\|_2^2 = \mu_2^2 x^T L_2^T L_2 x
   $$
2. Combine the expanded terms to form the full objective function [S2, unofficial]:
   $$
   \Phi(x) = x^T F^T Fx - 2x^T F^T y + y^T y + \mu_2^2 x^T L_2^T L_2 x
   $$
3. Compute the derivative of $\Phi(x)$ with respect to $x$ and set it to zero to find the minimum [S2, unofficial]:
   $$
   \frac{\partial \Phi}{\partial x} = 2F^T Fx - 2F^T y + 2\mu_2^2 L_2^T L_2 x = 0
   $$
4. Rearrange into the regularized normal equation [S2, unofficial]:
   $$
   (F^T F + \mu_2^2 L_2^T L_2) x = F^T y
   $$
   Define $B^T = F^T F + \mu_2^2 L_2^T L_2$. The equation becomes:
   $$
   B^T x = F^T y
   $$
5. Solve the linear system $B^T x = F^T y$ using Cholesky factorization of $B^T$ [S2, unofficial].
6. Determine the optimal regularization weight $\mu_2$ using the L-curve method [S2, unofficial]:
   - Scan $\mu_2$ over a range (e.g., $10^{-10}$ to $10^{-2}$ with $N_k = 201$ steps) [S2, unofficial].
   - For each $\mu_{2,k}$, solve for $x_k$.
   - Compute the residual norm $R(\mu_{2,k}) = \|Fx_k - y\|_2$ and the regularization norm $P(\mu_{2,k}) = \|L_2 x_k\|_2$ [S2, unofficial].
   - Plot $\log_{10} R(\mu_{2,k})$ vs $\log_{10} P(\mu_{2,k})$ to form the L-curve [S2, unofficial].
   - Select the $\mu_2$ corresponding to the corner of the L-curve as the optimal value [S2, unofficial].

**Parameters:**
- $\mu_2$: Regularization weight. Typical scanning range is $10^{-10}$ to $10^{-2}$ with 201 steps for L-curve analysis [S2, unofficial].

**Edge cases and pitfalls:**
- The second-order operator $L_2$ acts globally and does not explicitly enforce the physical center-axis condition ($\partial x / \partial r |_{r=0} = 0$) for axisymmetric fields, leading to inaccuracies near the symmetry axis [S2, unofficial].

## Subtopics

### Single-Parameter Tikhonov Regularization
A regularization technique to mitigate noise amplification in direct and iterative inversions by adding a second-order smoothness penalty to the objective function [S2, unofficial]. It stabilizes the reconstruction but fails to explicitly impose the zero-gradient condition at the center axis of axisymmetric fields [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Solving the linear system via Cholesky factorization is $O(n^3)$ for dense matrices, though $F^T F$ and $L_2^T L_2$ may have structure. The L-curve search requires solving the system $N_k$ times (e.g., 201 times) [S2, unofficial].
- **Memory Complexity:** Requires storage for $F$, $L_2$, and the factorized matrix $B^T$.
- **Strengths:** Mitigates noise amplification observed in direct and iterative inversions [S2, unofficial].
- **Limitations:** Does not enforce the physical center-axis condition ($\partial x / \partial r |_{r=0} = 0$), causing inaccuracies near the symmetry axis in axisymmetric tomography [S2, unofficial].

## Background
The method originates from tomographic reconstruction of a 2D slice from parallel projections using a 1D piecewise constant model [S1, unofficial]. The quadratic criterion for this model is non-differentiable, necessitating regularization techniques [S1, unofficial]. Single-parameter Tikhonov regularization was proposed as a standard approach to handle noise amplification [S2, unofficial].

## Sources
[S1] Reconstruction tomographique à partir d'un petit nombre de vues, https://openalex.org/W650467298, fetched 2026-10-29 (unofficial)
[S2] Tomographic Reconstruction of Infrared Radiance from Axisymmetric Sprays in a Hot Environment Using Multi-Parameter Regularization, https://doi.org/10.21203/rs.3.rs-10907544/v1, fetched 2026-10-26 (unofficial)
