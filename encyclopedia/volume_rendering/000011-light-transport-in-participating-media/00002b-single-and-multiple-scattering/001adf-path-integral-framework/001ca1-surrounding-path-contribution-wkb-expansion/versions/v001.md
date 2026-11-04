# Surrounding Path Contribution (WKB Expansion)
Also called: Gaussian Weighting of Paths, WKB Expansion

## Overview
Surrounding Path Contribution is a method for approximating the path integral in participating media by integrating only over the neighborhood of the Most Probable Path (MPP). It solves the problem of high-dimensional path integration by exploiting the fact that the action function has a global minimum at the MPP, allowing the integrand to be approximated as a Gaussian distribution. This approach is distinct from the Null-Scattering Path Integral Framework [3070-null-scattering-path-integral-framework.md], which uses virtual scattering to handle high optical thickness, whereas this method relies on the geometric concentration of path contributions around the MPP.

## How it works
**Inputs and Outputs:**
*   **Input:** The Most Probable Path (MPP) $\gamma_{MPP}$ for a given boundary condition (typically single-sided: fixed eye position and viewing ray, free termination). The action value $A(MPP)$ and the Hessian of the action at the MPP.
*   **Output:** An approximation of the radiance contribution from the MPP and its surrounding paths.

**Data Structures:**
*   **MPP:** A sequence of points and directions defining the path of least action.
*   **Parameter Vector $\xi$:** A vector representing deviations from the MPP in the re-parameterized path space. $\xi = 0$ corresponds to the MPP.

**Steps:**
1.  **Identify MPP:** Determine the Most Probable Path $\gamma_{MPP}$ for the specific boundary conditions (e.g., from eye to light source).
2.  **Parameterize Neighborhood:** Define a local coordinate system for paths near the MPP using a parameter vector $\xi$.
3.  **Expand Action:** Perform a Taylor expansion of the action function $A(\xi)$ around the MPP ($\xi=0$). Since the MPP is a minimum, the first-order derivative is zero.
    $$A(\xi) = A(0) + \frac{1}{2}\xi^2 \frac{\partial^2 A}{\partial \xi^2} + \dots$$
4.  **Approximate Integrand:** Substitute the expanded action into the path integral weight $\exp(-A(path))$. The linear term is omitted because $A(0)$ is a minimum.
    $$\exp(-A(path)) \sim \exp(-A(MPP)) \exp\left(-\frac{1}{2}\xi^2 \frac{\partial^2 A}{\partial \xi^2}\right)$$
    This results in a Gaussian form that weights nearby paths according to their "distance" (action difference) from the MPP.
5.  **Integrate:** Evaluate the integral over the re-parameterized path space. For a vector $\xi$, the propagator $G$ is approximated as:
    $$G \sim \exp(-A(MPP)) \int_{\xi} \exp\left(-\frac{1}{2}(\xi \nabla_{\xi})^2 A\right) d\xi$$
    In practice, this often reduces to a Gaussian integral over spatial deviations perpendicular to the MPP.

**Formulas:**
*   **Action Expansion:** $A(\xi) \approx A(MPP) + \frac{1}{2} \xi^T H \xi$, where $H$ is the Hessian matrix of the action at the MPP.
*   **Radiance Approximation:** The radiance $L$ is computed by multiplying the MPP contribution by the Gaussian integral of the surrounding paths. A specific implementation for multiple scattering [S1, unofficial] uses:
    $$L(s, \mathbf{x}, \vec{\omega}) = \exp \left( - \int_0^s c(MPP_s(s')) ds' \right) \exp(-A_c(MPP_s)) P_{MS}(\vec{\omega} - \vec{\omega}') \int_{plane\perp MPP} gauss(\mathbf{x}', w(s)) b(\mathbf{x}') L_0(\mathbf{x}', \vec{\omega}') d\mathbf{x}'$$
    where $w(s)$ is the spatial width of the Gaussian derived from the action curvature, and $P_{MS}$ is the multiply-scattered phase function.

**Parameters:**
*   **Spatial Width $w(s)$:** Measures radiation blurring along the path. It is derived from the second derivative of the action (curvature) at the MPP.
*   **Integration Domain:** The integral is typically restricted to the plane perpendicular to the MPP or a reduced subset of path space defined by the MPPs.

**Edge Cases and Pitfalls:**
*   **Degenerate MPP:** If the action does not have a clear minimum or the Hessian is singular, the Gaussian approximation fails.
*   **High Scattering:** The method assumes the single-scattered light distribution is strongly forward-peaked. It may be inaccurate for highly isotropic scattering media.
*   **Boundary Conditions:** The method is primarily designed for single-sided boundary conditions. Other boundary conditions may require different parameterizations.

## Subtopics

### WKB-based Multiple Scattering Approximation
An approximation for multiple scattering in participating media that replaces integration over complete $(x_0, \tilde{\theta}_0)$-space with integration over a reduced subset selected by a set of Multiple Path Points (MPPs). It relies on the assumption that single scattered light distribution is strongly forward peaked [S1, unofficial]. It is chosen when multiple scattering effects are significant but the forward-peaking assumption holds, allowing for efficient integration over a reduced path space.

## Operating characteristics
*   **Time Complexity:** Depends on the cost of finding the MPP and evaluating the Gaussian integral. Generally faster than full path integral methods for highly scattering media where the MPP dominates.
*   **Memory Complexity:** Low, as it only stores the MPP and local curvature information.
*   **Strengths:** Efficiently handles multiple scattering by focusing on the most probable paths. Provides a physical interpretation of radiance blurring.
*   **Limitations:** Accuracy degrades if the action landscape is not well-approximated by a quadratic form (i.e., if the MPP is not a sharp minimum). Relies on the forward-peaking assumption.
*   **When to Use:** Suitable for participating media with strong multiple scattering and forward-peaked phase functions.
*   **When Not to Use:** Unsuitable for media with isotropic scattering or when the MPP is not well-defined.

## Background
The method originates from the application of the WKB (Wentzel-Kramers-Brillouin) approximation to light transport in participating media. It was introduced in the context of practical rendering of multiple scattering effects, leveraging the concept of the Most Probable Path (MPP) to simplify the high-dimensional path integral [S1, unofficial]. The approach is part of a broader family of path integral formulations for light transport.

## Sources
[S1] Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28 (unofficial)
