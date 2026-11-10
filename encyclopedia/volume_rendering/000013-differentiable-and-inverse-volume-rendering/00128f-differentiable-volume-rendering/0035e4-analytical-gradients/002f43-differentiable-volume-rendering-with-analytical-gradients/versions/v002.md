# Differentiable Volume Rendering with Analytical Gradients

Also called: Analytical Differentiable Rendering, DiffRenderer

## Overview
A rendering technique that computes radiance and its analytical derivatives with respect to volume density directly, avoiding the overhead of propagating gradients through long computational graphs inherent in automatic differentiation [S1]. It solves the problem of efficient optimization of dynamic volumetric densities, such as in fluid reconstruction, by deriving an explicit form of the radiance transfer equation [S1]. Unlike [Differentiable Volume Rendering via Box Splines](2449-differentiable-volume-rendering-via-box-splines.md) which uses basis function interpolation, this method provides explicit gradients for density parameters [S1].

## How it works
**Inputs:** Ray origin $\mathbf{x}_a$, ray direction $\boldsymbol{\omega}$, ray segment endpoints $\mathbf{x}_a, \mathbf{x}_b$, and volume density field $\sigma(\mathbf{x})$.
**Outputs:** Radiance $L(\mathbf{x}, \boldsymbol{\omega})$ and its analytical derivative with respect to density parameters.

**Data Structures:**
The method operates on a ray marching structure. The ray is advanced from start $\mathbf{x}_a$ to end $\mathbf{x}_b$.
*   `x_u`: Current ray position.
*   `tau`: Accumulated transmittance.
*   `L_ss`: Accumulated radiance.

**Steps:**
1.  Initialize accumulated radiance $L_{ss} \leftarrow 0$ and transmittance $\tau \leftarrow 1$ [S1].
2.  Initialize current ray position $x_u \leftarrow \mathbf{x}_a$ [S1].
3.  Repeat the following loop until the ray reaches $\mathbf{x}_b$:
    a.  **Advance Ray Position:** Update the current position using the step size $\triangle d$ and direction $\boldsymbol{\omega}$:
        $$ x_u \rightarrow x_u + \boldsymbol{\omega} \triangle d $$ [S1]
    b.  **Compute Radiance Contribution:** Calculate the radiance contribution $J_{ss}(v_{x_u}, \boldsymbol{\omega})$ based on the local density and emission at $x_u$ [S1]. This involves the term $\kappa_t$ (extinction coefficient) and the transmittance $\tau(v_{x_u}, v_{x_b})$ [S1].
    c.  **Accumulate:** Update the total radiance:
        $$ L_{ss}(\mathbf{x}, \boldsymbol{\omega}) \leftarrow L_{ss}(\mathbf{x}, \boldsymbol{\omega}) + \tau(v_{x_u}, v_{x_b}) \kappa_t(\dots) $$ [S1]
        *Note: The exact functional form of $\kappa_t$ and the transmittance calculation between $v_{x_u}$ and $v_{x_b}$ is referenced as Equation (5) in the source [S1].*
4.  Return $L_{ss}$ and the analytical derivatives computed via the derived radiance transfer equation [S1].

**Parameters:**
*   $\triangle d$: Step size for ray marching. Value not given in the sources.
*   $\kappa_t$: Extinction coefficient. Defined as $\sigma + \sigma_s$ (density + scattering density) in standard volume rendering, but specific composition is not detailed in the provided text [S1].

**Edge Cases and Pitfalls:**
*   **Singularities:** The method must handle singularities in the moment bounds if used in conjunction with [Differentiable Moment Bounds](8855-differentiable-moment-bounds.md) [S1].
*   **Discrete Steps:** As with other methods in this family, discrete steps like voxel traversal order are non-differentiable and require masking or fixed selection [S1].

## Subtopics

### [Differentiable Moment Bounds](8855-differentiable-moment-bounds.md)
Computes continuously differentiable upper and lower bounds on the cumulative mass of a measure consistent with a given sequence of power moments. It solves the truncated Hamburger moment problem by constructing a unique "canonical" discrete measure. Unlike standard visibility functions based on Chebyshev's inequality, it generalizes to arbitrary moment sequences ($2n+1$ moments) to provide tighter bounds. It is distinguished by providing a robust, numerically stable implementation that handles singularities, enabling its use in differentiable rendering pipelines where gradients with respect to scene parameters are required.

## Operating characteristics
*   **Time Complexity:** Avoids the overhead of automatic differentiation (AD) which propagates gradients through long computational graphs [S1]. This leads to improved performance for optimizing dynamic volumetric densities [S1]. Exact complexity not given.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Direct and efficient computation of radiance derivatives with respect to density. Avoids gradient tracking and redundant operations [S1].
*   **Limitations:** Requires deriving analytical forms for the radiance transfer equation, which may be complex for arbitrary volume representations.
*   **Use Cases:** Fluid reconstruction tasks, dynamic volumetric density optimization [S1].

## Background
Originated in the work "Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video" [S1]. The work builds upon insights that calculating derivatives via AD fundamentally relies on propagating gradients through long computational graphs, which carries inherent overhead compared to directly computing derivatives analytically [S1].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
