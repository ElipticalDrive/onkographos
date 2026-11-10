# Differentiable Volume Rendering with Analytical Gradients

Also called: Analytical Differentiable Rendering, DiffRenderer

## Overview
This method computes radiance and its analytical derivatives with respect to volume density directly, avoiding the overhead of propagating gradients through long computational graphs inherent in automatic differentiation [S1]. It builds on [Differentiable volume rendering](43-differentiable-volume-rendering.md) by deriving an analytical form of the radiance transfer equation, enabling efficient optimization of dynamic volumetric densities [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray $\mathbf{r}$ defined by origin and direction; volume density field $\sigma(\mathbf{x})$; step size $\triangle d$; ray segment bounds $x_a, x_b$.
*   **Output:** Radiance $L$ and its derivative with respect to density parameters.

**Data Structures:**
*   Accumulator $L_{ss}$ for radiance.
*   Current ray position $x_u$.
*   Transmittance factor $\tau$.
*   Absorption coefficient $\kappa_t$.

**Steps:**
1.  Initialize radiance accumulator $L_{ss} \leftarrow 0$ [S1].
2.  Initialize current ray position $x_u$ to the start of the ray segment $x_a$ [S1].
3.  Repeat the following loop until the ray exits the volume or reaches $x_b$ [S1]:
    a.  **Advance Ray Position:** Update the current position using the step size $\triangle d$ and direction vector $\boldsymbol{u}$:
        $$ x_u \rightarrow x_u + \boldsymbol{u} \triangle d $$
        [S1]
    b.  **Compute Radiance Contribution:** Calculate the radiance contribution $J_{ss}(v_{x_u}, \omega)$ using the analytical radiance transfer equation (Equation 5 in [S1]) [S1].
    c.  **Accumulate Radiance:** Update the total radiance:
        $$ L_{ss}(x, \omega) \leftarrow L_{ss}(x, \omega) + \tau(v_{x_u}, v_{x_b}) \kappa_t(\dots) $$
        [S1]
        *Note: The exact functional form of $\tau$ and $\kappa_t$ within the accumulation step is not fully specified in the provided text snippets, but the structure implies a weighted sum based on transmittance and absorption.* [S1]

**Parameters:**
*   $\triangle d$: Step size for ray marching. Value not given in the sources.
*   $\boldsymbol{u}$: Unit direction vector of the ray.

**Edge Cases and Pitfalls:**
*   The method relies on the analytical derivation of the radiance transfer equation. If the density field is not differentiable or contains discontinuities not handled by the analytical model, gradients may be inaccurate [S1].
*   The loop termination condition (when the ray reaches $x_b$ or exits the volume) is implied but not explicitly detailed in the provided pseudocode snippet [S1].

## Subtopics

### [Differentiable Moment Bounds](8855-differentiable-moment-bounds.md)
Computes continuously differentiable upper and lower bounds on cumulative mass using power moments. It solves the truncated Hamburger moment problem and provides robust, numerically stable bounds for differentiable rendering pipelines.

## Operating characteristics
*   **Performance:** Avoids the overhead of automatic differentiation (AD) by directly computing derivatives, leading to improved performance for optimizing dynamic volumetric densities [S1].
*   **Complexity:** The computational complexity is not explicitly stated, but the avoidance of long computational graphs suggests a reduction in memory and time overhead compared to AD-based approaches [S1].
*   **Strengths:** Efficient for fluid reconstruction and dynamic volumetric density optimization [S1].
*   **Limitations:** Requires an analytical form of the radiance transfer equation, which may limit applicability to specific volume rendering models [S1].

## Background
Originated in the context of physics-based differentiable rendering for fluid modeling from monocular video [S1]. The work addresses the limitations of automatic differentiation in volumetric data, where gradient propagation through long computational graphs carries inherent overhead [S1].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
