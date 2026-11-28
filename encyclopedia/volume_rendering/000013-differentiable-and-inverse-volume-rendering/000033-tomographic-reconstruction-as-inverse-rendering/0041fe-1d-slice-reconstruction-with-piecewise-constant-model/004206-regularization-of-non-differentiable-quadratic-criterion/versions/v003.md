# Regularization of Non-Differentiable Quadratic Criterion

Also called: Regularization for Piecewise Constant Model

## Overview
This method addresses the non-differentiability of a quadratic criterion minimized under constraints for a 1D piecewise constant model [S1, unofficial]. It applies regularization techniques to stabilize parameter estimation when the objective function lacks smoothness [S1, unofficial]. It differs from conventional Model-Based Iterative Reconstruction (MBIR) by employing non-differentiable penalties and coordinate descent rather than simple L2-norm regularization and gradient descent [S2, unofficial][S1, unofficial].

## How it works
**Inputs:**
- Measured projection data vector $y$.
- Forward operator matrix $F$ mapping the unknown radial field $x$ to the projection space [S2, unofficial].
- Regularization weight parameter $\mu_2$ [S2, unofficial].

**Outputs:**
- Reconstructed radial field vector $x$.

**Data Structures:**
- $x$: Vector of size $n$ representing the parameters of the 1D piecewise constant model.
- $F$: Matrix of size $m \times n$ (where $m$ is the number of projections).
- $L_2$: Second-order finite-difference operator matrix of size $n \times n$.

**Steps:**
1. Expand the objective function terms into quadratic forms: $\|Fx - y\|_2^2 = x^T F^T Fx - 2x^T F^T y + y^T y$ and $\mu_2^2 \|L_2 x\|_2^2 = \mu_2^2 x^T L_2^T L_2 x$ [S2, unofficial].
2. Combine the expanded terms to form the full objective function $\Phi(x)$ [S2, unofficial].
3. Compute the derivative $\frac{\partial \Phi}{\partial x} = 2F^T Fx - 2F^T y + 2\mu_2^2 L_2^T L_2 x$ and set it to zero to find the minimum [S2, unofficial].
4. Rearrange into the regularized normal equation $(F^T F + \mu_2^2 L_2^T L_2) x = F^T y$ [S2, unofficial].
5. Define $B^T = F^T F + \mu_2^2 L_2^T L_2$ and solve the linear system $B^T x = F^T y$ using Cholesky factorization of $B^T$ [S2, unofficial].
6. Determine the optimal regularization weight $\mu_2$ using the L-curve method: scan $\mu_2$ from $10^{-10}$ to $10^{-2}$ with $N_k = 201$ steps, solve for $x_k$ at each step, plot $\log_{10} \|Fx_k - y\|_2$ versus $\log_{10} \|L_2 x_k\|_2$, and select the $\mu_2$ corresponding to the corner of the curve [S2, unofficial].

**Formulas:**
The objective function $\Phi(x)$ combines data fidelity and a smoothness penalty [S2, unofficial]:
$$
\Phi(x) = \|Fx - y\|_2^2 + \mu_2^2 \|L_2 x\|_2^2
$$
where:
- $\|Fx - y\|_2^2$ is the data fidelity term [S2, unofficial].
- $\mu_2^2 \|L_2 x\|_2^2$ is the Tikhonov regularization term [S2, unofficial].
- $L_2$ is defined as:
$$
L_2 = \frac{1}{\Delta r^2} \begin{bmatrix} 1 & -2 & 1 & 0 & \cdots & 0 \\ 0 & 1 & -2 & 1 & \cdots & 0 \\ \vdots & & & \ddots & & \vdots \\ 0 & \cdots & 0 & 1 & -2 & 1 \end{bmatrix}
$$
- $\Delta r$ is the radial step size [S2, unofficial].

**Parameters:**
- $\mu_2$: Regularization weight controlling the trade-off between data fidelity and smoothness. Typical scanning range is $10^{-10}$ to $10^{-2}$ with 201 steps for L-curve analysis [S2, unofficial].

**Edge cases and pitfalls:**
- The second-order operator $L_2$ acts globally and does not explicitly enforce the physical center-axis condition ($\partial x / \partial r |_{r=0} = 0$) for axisymmetric fields, leading to inaccuracies near the symmetry axis [S2, unofficial].

## Subtopics

### Single-Parameter Tikhonov Regularization
A regularization technique to mitigate noise amplification in direct and iterative inversions by adding a second-order smoothness penalty to the objective function [S2, unofficial]. It stabilizes the reconstruction but fails to explicitly impose the zero-gradient condition at the center axis of axisymmetric fields [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Mitigates noise amplification observed in direct and iterative inversions [S2, unofficial].
- **Limitations:** Does not enforce the physical center-axis condition ($\partial x / \partial r |_{r=0} = 0$), causing inaccuracies near the symmetry axis in axisymmetric tomography [S2, unofficial].

## Background
The method addresses the non-differentiability of a quadratic criterion minimized under constraints for a 1D piecewise constant model [S1, unofficial]. Single-parameter Tikhonov regularization was proposed as a standard approach to handle noise amplification in this context [S2, unofficial].

## Sources
[S1] Reconstruction tomographique à partir d'un petit nombre de vues, https://openalex.org/W650467298, fetched 2026-10-29 (unofficial)
[S2] Tomographic Reconstruction of Infrared Radiance from Axisymmetric Sprays in a Hot Environment Using Multi-Parameter Regularization, https://doi.org/10.21203/rs.3.rs-10907544/v1, fetched 2026-10-26 (unofficial)
