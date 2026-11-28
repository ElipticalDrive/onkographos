# Surrounding Path Contribution (WKB Expansion)
Also called: Gaussian Weighting of Paths, WKB Expansion

## Overview
Surrounding Path Contribution approximates the complete path integral in participating media by integrating only over paths in the vicinity of the Most Probable Path (MPP) [S1]. It solves the problem of efficiently estimating radiance by replacing the infinite-dimensional integral with a Gaussian-weighted sum of "surrounding" paths [S1].

## How it works
**Inputs:**
- The Most Probable Path (MPP) for a single-sided boundary condition (fixed start point and direction, e.g., eye position and primary viewing ray) [S1].
- The action function $A(\text{path})$ associated with the light transport equation [S1].
- A parameterization vector $\vec{\xi}$ representing deviations from the MPP, where $\vec{\xi} = \vec{0}$ at the MPP [S1].

**Outputs:**
- An approximation of the radiance contribution, dominated by the MPP term and a Gaussian integral over the re-parameterized path space [S1].

**Data Structures:**
- The scalar action value $A(0)$ for the MPP and the Hessian matrix $\mathbf{H}$ (second derivatives of $A$ with respect to $\vec{\xi}$) evaluated at the MPP [S1].

**Steps:**
1. Identify the MPP for the specified single-sided boundary condition and compute its action value $A(0)$ [S1].
2. Parameterize the family of nearby paths using a vector $\vec{\xi}$ such that $\vec{\xi} = \vec{0}$ corresponds to the MPP [S1].
3. Perform a Taylor series expansion of the action $A(\vec{\xi})$ around $\vec{\xi} = \vec{0}$ [S1]. Since $A$ has a global minimum at the MPP, the linear term is omitted [S1]. The expansion is:
   $$ A(\xi) = A(0) + \frac{1}{2}\xi^2 \frac{\partial^2 A}{\partial \xi^2} + \dots $$
   For a vector parameterization, this generalizes to a quadratic form involving the Hessian matrix $\mathbf{H}$ [S1].
4. Substitute the expanded action into the path integral weight $\exp(-A(\text{path}))$ [S1]. The weight for a surrounding path becomes:
   $$ \exp(-A(\text{path})) \sim \exp(-A(MPP)) \exp\left(-\frac{1}{2}\xi^2 \frac{\partial^2 A}{\partial \xi^2}\right) $$
   This yields a Gaussian form that weights nearby paths according to their "distance" from the MPP [S1].
5. Evaluate the integral over the re-parameterized path space [S1]. The propagator $G$ is approximated as:
   $$ G \sim \exp(-A(MPP)) \int_{\xi} \exp\left(-\frac{1}{2} (\xi \nabla_{\xi})^2 A \right) d\xi $$
   where $(\xi \nabla_{\xi})^2 A$ represents the quadratic form $\vec{\xi}^T \mathbf{H} \vec{\xi}$ derived from the Hessian [S1].
6. Use this expression to estimate properties of the radiance distribution, such as radiation blurring, which is measured by the spatial width of the contributing paths [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def surrounding_path_contribution(mpp_action, hessian_matrix):
    """
    Approximates the path integral using WKB expansion around the MPP.
    
    Args:
        mpp_action (float): The action value A(0) of the Most Probable Path.
        hessian_matrix (list of lists): The Hessian matrix H of the action at the MPP.
        
    Returns:
        float: Approximated radiance contribution (propagator G).
    """
    # Step 1: Compute the MPP contribution factor
    mpp_factor = math.exp(-mpp_action)
    
    # Step 2: Evaluate the Gaussian integral over the re-parameterized path space.
    # The integral is of the form int(exp(-0.5 * xi^T * H * xi)) dxi.
    # For an n-dimensional Gaussian integral, the analytical result is (2*pi)^(n/2) / sqrt(det(H)).
    n = len(hessian_matrix)
    
    # Compute determinant of Hessian analytically for small dimensions
    if n == 1:
        det_H = hessian_matrix[0][0]
    elif n == 2:
        det_H = hessian_matrix[0][0] * hessian_matrix[1][1] - hessian_matrix[0][1] * hessian_matrix[1][0]
    else:
        # [G] For n > 2, compute det(H) using LU decomposition or standard linear algebra routines.
        det_H = 1.0 
        
    # Guard against non-positive definite Hessian or zero determinant
    if det_H <= 0:
        raise ValueError("Hessian must be positive definite for a valid Gaussian integral.")
        
    gaussian_integral = (2 * math.pi) ** (n / 2) / math.sqrt(det_H)
    
    # Step 3: Combine MPP contribution with the Gaussian integral
    result = mpp_factor * gaussian_integral
    return result
```

**Parameters:**
- Radiation blurring along the path is measured by the spatial width of contributing paths [S1].

**Edge Cases and Pitfalls:**
- The expansion assumes $A$ has a global minimum at the MPP, so the linear term is omitted [S1].
- The integration space is still infinite-dimensional, but properties are estimated by keeping some parameters fixed [S1].

## Subtopics

### [WKB-based Multiple Scattering Approximation](1490-wkb-based-multiple-scattering-approximation.md)
The WKB-based Multiple Scattering Approximation is a method for computing multiply-scattered light in participating media by replacing the integration over the complete initial path space $(x_0, \tilde{\omega}_0)$ with an integration over a reduced subset defined by Multiple Path Points (MPPs). It is distinguished from the diffusion approximation by its validity in sparse and inhomogeneous media, though it assumes strongly forward-peaked phase functions and ignores backscattering.

## Operating characteristics
- **Strengths:** Efficiently models radiation blurring by considering the contribution of surrounding paths via WKB expansion and Gaussian weighting [S1].
- **Limitations:** The integration space remains infinite-dimensional, requiring approximation by fixing parameters [S1].

## Background
The Surrounding Path Contribution method is based on the WKB (Wentzel-Kramers-Brillouin) expansion of the path integral [S1]. It was introduced in the context of practical rendering of multiple scattering effects in participating media [S1]. The method builds on the Path Integral Framework [873-path-integral-framework.md]. The WKB expansion allows for the approximation of the path integral by focusing on the neighborhood of the Most Probable Path (MPP) [S1].

## Sources
[S1] Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
