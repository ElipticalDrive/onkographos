# Transmittance Mapping

Also called: Moment-based transmittance mapping, TM-n

## Overview
Transmittance Mapping is a differentiable method for approximating volume transmittance (visibility) using bounds derived from the theory of moments [S1]. It solves the problem of computing visibility in inverse rendering tasks, such as pose recovery and 3D reconstruction, where gradients with respect to scene geometry are required [S1]. Unlike shadow mapping variants, which fail in empty space and suffer from geometric inaccuracies, Transmittance Mapping provides a principled, volume-based approximation that correctly reports constant 1 transmittance in free space while maintaining efficiency comparable to shadow mapping [S1]. It builds on [Heterogeneous Medium Transmittance](4909-heterogeneous-medium-transmittance.md) by providing a differentiable, moment-based approximation of the transmittance integral.

## How it works
**Inputs:**
*   Scene geometry (e.g., Signed Distance Fields or triangle meshes) [S1].
*   Density scaling parameter $s$ [S1].
*   Order of approximation $n$ (integer $\ge 1$) [S1].
*   Light source position (assumed point light; area lights are not supported) [S1].
*   Camera ray direction and origin [S1].

**Outputs:**
*   Approximated visibility $V$ along the ray [S1].
*   Gradient of $V$ with respect to scene parameters (e.g., pose, density) [S1].

**Data Structures:**
*   **Transmittance Map:** A 2D texture or grid storing the computed transmittance bounds for a given light view. The resolution determines spatial accuracy; coarse resolutions miss small structures [S1].
*   **Moment Matrices:** For order $n$, the method constructs matrices based on $2n + 1$ moments [S1]. The exact layout of these matrices is not specified in the sources, but they are derived from the truncated Hamburger moment problem [S1].

**Parameters:**
*   **Order $n$:** Determines the number of moments used ($2n + 1$). Higher orders (e.g., TM-5) provide better accuracy but increase computational cost and numerical instability risks [S1]. Typical values range from 1 to 5 [S1].
*   **Density Scaling $s$:** Controls volume density spikes. Optimized from low to high values to balance gradient density and surface localization [S1].

**Steps:**
1.  **Discretization:** The space is discretized to form the transmittance map. The resolution of this map affects accuracy; adaptive techniques are suggested but not detailed [S1].
2.  **Moment Calculation:** Compute the moments of the density distribution along the ray or within the voxel/region. The sources do not provide the explicit formula for moment calculation from density $\sigma$ [G: Typically involves integrating $\sigma^k$ or weighted integrals of density].
3.  **Bound Construction:** Construct the lower and upper bounding functions for the transmittance integral using the theory of moments. The truncated Hamburger moment problem provides these bounds [S1].
4.  **Differentiability:** The bounding functions are continuously differentiable with respect to their parameters (geometry, density) [S1].
5.  **Evaluation:** Evaluate the transmittance $T(t)$ and visibility $V$ using the bounds. The visibility is computed as:
    $$ V = \int_0^\infty \sigma_{\mathbf{r}(t)} T(t) \, dt $$
    where $V$ is the visibility, $\sigma$ is the density, and $T(t)$ is the transmittance from the light source to point $\mathbf{r}(t)$ [S1]. A direction-dependent normalization factor is omitted for brevity [S1].
6.  **Gradient Propagation:** Compute gradients of $V$ with respect to scene parameters using automatic differentiation, leveraging the differentiability of the moment bounds [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transmittance_mapping(ray_origin, ray_direction, geometry, density_scaling, order_n, light_position):
    """
    Approximates volume transmittance using moment-based bounds.
    
    Args:
        ray_origin: 3D vector, camera ray origin.
        ray_direction: 3D vector, camera ray direction.
        geometry: Scene representation (e.g., SDF).
        density_scaling: Float, parameter s controlling density spikes.
        order_n: Int, order of moment approximation (n).
        light_position: 3D vector, position of the point light.
        
    Returns:
        visibility: Float, approximated visibility V.
        gradient: Float, gradient of V w.r.t. geometry/density.
    """
    # 1. Compute moments along the ray or in the relevant volume region.
    # The sources do not specify the exact moment calculation formula.
    # [G] Assuming moments M_k are computed from density sigma along the ray.
    moments = compute_moments(ray_origin, ray_direction, geometry, density_scaling, order_n)
    
    # 2. Construct transmittance bounds using the truncated Hamburger moment problem.
    # The sources state that lower and upper bounding functions are continuously differentiable.
    # [G] Constructing the Hankel matrix and solving for bounds is implied but not detailed.
    transmittance_lower, transmittance_upper = construct_moment_bounds(moments, order_n)
    
    # 3. Evaluate visibility integral.
    # V = integral of sigma * T(t) dt.
    # The sources provide the integral form but not the numerical integration method.
    # [G] Using the bounds to approximate the integral.
    visibility = evaluate_visibility_integral(ray_origin, ray_direction, geometry, density_scaling, transmittance_lower, transmittance_upper)
    
    # 4. Compute gradients.
    # The method is differentiable.
    gradient = compute_gradient(visibility, geometry, density_scaling)
    
    return visibility, gradient

def compute_moments(ray_origin, ray_direction, geometry, density_scaling, order_n):
    """
    Computes the 2*order_n + 1 moments of the density distribution.
    [G] Specific formula for moment calculation from density is not in sources.
    """
    # [G] Placeholder for moment calculation logic.
    # Typically involves integrating density^k or weighted integrals.
    pass

def construct_moment_bounds(moments, order_n):
    """
    Constructs lower and upper bounds for transmittance using moment theory.
    [G] Specific algorithm for solving the truncated Hamburger moment problem is not in sources.
    """
    # [G] Placeholder for bound construction logic.
    pass

def evaluate_visibility_integral(ray_origin, ray_direction, geometry, density_scaling, transmittance_lower, transmittance_upper):
    """
    Evaluates the visibility integral V = integral(sigma * T(t) dt).
    """
    # [G] Placeholder for numerical integration.
    pass

def compute_gradient(visibility, geometry, density_scaling):
    """
    Computes the gradient of visibility with respect to geometry and density.
    """
    # [G] Placeholder for gradient computation.
    pass
```

**Edge Cases and Pitfalls:**
*   **Empty Space:** Shadow mapping fails in empty space because it is not volume-based; Transmittance Mapping correctly reports constant 1 transmittance [S1].
*   **Numerical Instability:** Higher-order moments ($n > 5$) lead to ill-conditioned matrices and difficult polynomial root finding. Implementations must be designed around these constraints [S1].
*   **Light Source Assumptions:** The approach assumes point lights; area lights or image-based lighting are not supported [S1].
*   **Resolution Dependence:** If the transmittance map is too coarse, small structures are missed. Memory requirements increase with resolution [S1].

## Operating characteristics
*   **Time Complexity:** Equally efficient as shadow mapping, approximately 4× faster than shadow rays [S1].
*   **Memory Complexity:** Reasonable memory consumption; e.g., 10.55 GiB peak for 190x256 pixel views using 128 samples in multi-view 3D reconstruction [S1]. Memory increases with map resolution [S1].
*   **Performance:** In pose recovery, TM-5 achieved a translation error ($\Delta t$) of 0.07 and Hausdorff distance ($D_H$) of 0.09, compared to Tiwary et al.'s 0.50 and 0.33 respectively [S1]. In multi-view 3D reconstruction, TM-3 maintained one iteration per second [S1].
*   **Strengths:** More accurate than shadow mapping variants (Tiwary et al., VSM); handles free space correctly; differentiable; equally efficient as shadow mapping [S1].
*   **Limitations:** Numerical instability for higher orders; not supported for area lights or image-based lighting; quality depends on spatial discretization [S1].
*   **When to Use:** Differentiable shadow mapping and differentiable volumetric visibility for inverse rendering tasks like pose recovery, shadow art, and 3D reconstruction from low-density volumes [S1].
*   **When Not to Use:** When area lights or image-based lighting are required; when very high-order approximations are needed without careful numerical handling [S1].

## Background
Transmittance Mapping was introduced by Worchel and Alexa in 2025 [S1]. It is based on the theory of moments, specifically the truncated Hamburger moment problem, to provide differentiable bounds for volume transmittance [S1]. The method was developed to address the inaccuracies of shadow mapping in inverse rendering, particularly in pose recovery and 3D reconstruction tasks [S1]. It builds on previous work in differentiable volume rendering and shadow mapping [S1].

## Sources
[S1] Moment Bounds are Differentiable: Efficiently Approximating Measures in Inverse Rendering, https://doi.org/10.1145/3730899, fetched 2026-10-27
