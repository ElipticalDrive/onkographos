# Surrounding Path Contribution (WKB Expansion)
Also called: Gaussian Weighting of Paths, WKB Expansion

## Overview
Surrounding Path Contribution approximates the complete path integral in participating media by integrating only over paths in the vicinity of the Most Probable Path (MPP) [S1]. It solves the problem of efficiently estimating radiance in sparse or inhomogeneous media where full path integration is expensive, by replacing the infinite-dimensional integral with a Gaussian-weighted sum of "surrounding" paths [S1]. Unlike the diffusion approximation, which requires dense, uniform media, this method remains valid for sparse and inhomogeneous media provided the scattering coefficients vary smoothly and the phase function is strongly forward-peaked [S1, S2]. It sets itself apart from the [Null-Scattering Path Integral Framework](3070-null-scattering-path-integral-framework.md) by avoiding null-scattering rejection and instead analytically approximating the integral via a WKB expansion around the MPP.

## How it works
**Inputs:**
- The Most Probable Path (MPP) for a given boundary condition (typically single-sided: fixed start point and direction, e.g., eye position and viewing ray).
- The action function $A(\text{path})$ associated with the light transport equation.
- A parameterization vector $\vec{\xi}$ representing deviations from the MPP, where $\vec{\xi} = \vec{0}$ at the MPP [S1].

**Outputs:**
- An approximation of the radiance contribution, dominated by the MPP term and a Gaussian integral over the surrounding path space [S1].

**Data Structures:**
- The method relies on the scalar action value $A(0)$ for the MPP and the Hessian matrix (second derivatives) of the action with respect to the parameterization vector $\vec{\xi}$ evaluated at the MPP [S1].
- Not given in the sources.

**Steps:**
1. Identify the Most Probable Path (MPP) for the specified boundary condition. Compute its action value $A(0)$ [S1].
2. Parameterize the family of nearby paths using a vector $\vec{\xi}$ such that $\vec{\xi} = \vec{0}$ corresponds to the MPP [S1].
3. Perform a Taylor series expansion of the action $A(\vec{\xi})$ around $\vec{\xi} = \vec{0}$. Since $A$ has a global minimum at the MPP, the linear term is zero. The expansion is:
   $$ A(\vec{\xi}) = A(0) + \frac{1}{2} \vec{\xi}^T \mathbf{H} \vec{\xi} + \dots $$
   where $\mathbf{H}$ is the Hessian matrix with elements $\frac{\partial^2 A}{\partial \xi_i \partial \xi_j}$ [S1].
4. Substitute the expanded action into the path integral weight $\exp(-A(\text{path}))$. The weight for a surrounding path becomes:
   $$ \exp(-A(\text{path})) \sim \exp(-A(0)) \exp\left(-\frac{1}{2} \vec{\xi}^T \mathbf{H} \vec{\xi}\right) $$
   This results in a Gaussian form that weights nearby paths according to their "distance" (in action space) from the MPP [S1].
5. Evaluate the integral over the re-parameterized path space. The propagator $G$ is approximated as:
   $$ G \sim \exp(-A(0)) \int_{\vec{\xi}} \exp\left(-\frac{1}{2} (\vec{\xi} \nabla_{\vec{\xi}})^2 A \right) d\vec{\xi} $$
   where $(\vec{\xi} \nabla_{\vec{\xi}})^2 A$ represents the quadratic form derived from the Hessian [S1].
6. Use this expression to estimate properties of the radiance distribution, such as radiation blurring, which is measured by the spatial width of the contributing paths [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surrounding_path_contribution(mpp_action, hessian_matrix, num_samples):
    """
    Approximates the path integral using WKB expansion around the MPP.
    
    Args:
        mpp_action (float): The action value A(0) of the Most Probable Path.
        hessian_matrix (ndarray): The Hessian matrix H of the action at the MPP.
        num_samples (int): Number of samples to estimate the Gaussian integral.
        
    Returns:
        float: Approximated radiance contribution.
    """
    # Step 1: Compute the MPP contribution factor
    mpp_factor = math.exp(-mpp_action)
    
    # Step 2: Sample from the Gaussian distribution defined by the Hessian
    # The quadratic form is 0.5 * xi^T * H * xi
    # We approximate the integral of exp(-0.5 * xi^T * H * xi) over xi
    
    # Note: The exact evaluation of the integral depends on the dimensionality of xi.
    # If xi is 1D, the integral is sqrt(2*pi / H[0,0]).
    # If xi is multi-dimensional, the integral is (2*pi)^(n/2) / sqrt(det(H)).
    # The sources do not specify the dimensionality of xi or the exact integration method.
    # [G] Assuming a 1D parameterization for simplicity as per the source's scalar example.
    
    hessian_val = hessian_matrix[0, 0] # Extract second derivative for 1D case
    
    # Integral of exp(-0.5 * H * xi^2) dxi = sqrt(2*pi / H)
    gaussian_integral = math.sqrt(2 * math.pi / hessian_val)
    
    # Step 3: Combine MPP contribution with the Gaussian integral
    result = mpp_factor * gaussian_integral
    
    return result
```

**Parameters:**
- Radiation blurring along the path is measured by the spatial width of contributing paths [S1].
- Not given in the sources.

**Edge Cases and Pitfalls:**
- The expansion assumes $A$ has a global minimum at the MPP, so the linear term is omitted [S1].
- The method is valid only if scattering coefficients vary smoothly and the phase function is strongly forward-peaked [S1, S2].
- The integration space is still infinite-dimensional, but the method estimates properties by keeping some parameters fixed [S1].

## Subtopics

### [WKB-based Multiple Scattering Approximation](1490-wkb-based-multiple-scattering-approximation.md)
The WKB-based Multiple Scattering Approximation is a method for simulating light transport in participating media that approximates the multiple scattering term by integrating contributions only over paths close to the Most Probable Path (MPP). Unlike the diffusion approximation, which is restricted to dense, uniform media, this method is valid for sparse and inhomogeneous media, provided the scattering coefficients vary smoothly and the phase function is strongly forward-peaked. It sets itself apart by replacing the full integration over the $(x_0, \vec{\omega}_0)$-space with a reduced integration over a subset of paths defined by MPPs, effectively modeling the spatial spreading and blurring of incident radiance without spawning secondary rays.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Efficiently models radiation blurring in sparse and inhomogeneous media without spawning secondary rays [S1, S2]. Valid where diffusion approximation fails [S1, S2].
- **Limitations:** Requires smooth variation of scattering coefficients and strongly forward-peaked phase functions [S1, S2]. The integration space is still infinite-dimensional, requiring approximation [S1].
- **When to use:** When simulating light transport in sparse or inhomogeneous participating media with forward-peaked scattering, and full path integration is too expensive [S1, S2].
- **When not to use:** When the medium is dense and uniform (use diffusion approximation) or when the phase function is not forward-peaked [S1, S2].

## Background
The Surrounding Path Contribution method is based on the WKB (Wentzel-Kramers-Brillouin) expansion of the path integral [S1]. It was introduced in the context of practical rendering of multiple scattering effects in participating media [S1]. The method builds on the Path Integral Framework [873-path-integral-framework.md], which formulates light transport as a Monte Carlo integration problem over the space of light transport paths [873-path-integral-framework.md]. The WKB expansion allows for the approximation of the path integral by focusing on the neighborhood of the Most Probable Path (MPP) [S1].

## Sources
[S1] Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
