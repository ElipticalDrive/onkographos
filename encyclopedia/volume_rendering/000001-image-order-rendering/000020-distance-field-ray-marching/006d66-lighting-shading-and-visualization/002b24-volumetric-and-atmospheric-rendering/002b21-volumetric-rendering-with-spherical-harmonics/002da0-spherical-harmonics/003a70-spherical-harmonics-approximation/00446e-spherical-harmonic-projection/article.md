# Spherical Harmonic Projection
Also called: SH Compression, SH Integration, Spherical Harmonic (SH) Projection for Volume Rendering
## Overview
Spherical Harmonic (SH) Projection computes the coefficients of a spherical function $f(\omega)$'s expansion in the spherical harmonics basis to approximate it [S1]. It solves the problem of efficiently representing complex angular data (lighting, material reflectance, in-scattering) for volume rendering by leveraging the basis's orthonormality for fast integration [S1]. It builds on [Spherical Harmonics Approximation](8741-spherical-harmonics-approximation.md) by focusing on the least-squares derivation of coefficients and GPU-accelerated dot-product integration [S1].
## How it works
**Inputs and Outputs:**
- Input: A spherical function $f(s)$ defined over the domain $S$ (the unit sphere) [S1].
- Output: A vector of coefficients $c_j$ representing the projection of $f$ onto the SH basis [S1].

**Data Structures:**
The coefficients are stored as a flat array [S1].
- Field `coeffs`: Array of size $(L+1)^2$ [S1].
- Field `L`: Integer, maximum band order [S1].

**Parameters:**
- $L$: Maximum SH band order. Typical value: $L=3$ [S1]. This results in $(3+1)^2 = 16$ coefficients per channel [S1].

**Steps:**
1. **Define Basis:** Select the orthonormal spherical harmonics basis functions $Y_j(s)$ up to order $L$ [S1].
2. **Compute Coefficients:** For each basis function index $j$ (where $0 \le j < (L+1)^2$), compute the coefficient $c_j$ by integrating the product of the basis function and the target function over the sphere:
   $$c_j = \int_S Y_j(s) f(s) ds \quad (1)$$
   This is a least-squares projection that minimizes the squared error between the approximation and the original function $f$ [S1].
3. **Store Coefficients:** Store the computed $c_j$ values in the coefficient vector [S1].

**Integration of Projected Functions:**
Once functions $f$ and $g$ are projected to coefficients $c$ and $d$ respectively, their integration is computed as the dot product of their coefficient vectors:
$$I(\tilde{f}, \tilde{g}) = \sum_i c_i d_i \quad (2)$$
This property allows integration to be performed efficiently on the GPU using a single dot product operation, avoiding the reconstruction and integration of the original functions [S1].

**Pitfalls and Edge Cases:**
- **Three-Function Integration:** Integrating three functions (Material $M$, In-scattering $L_{mi}^0$, and Light $L_i$) is complex. While techniques exist [Sny06], a simpler, non-physical approximation is used for performance reasons [S1].
- **Precision:** Approximation quality improves with more samples, but the number of SH bands must remain consistent across projections to simplify the rendering pipeline [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_harmonic_projection(f_sample, L=3):
    """
    Computes SH coefficients for a spherical function f.
    
    Args:
        f_sample: A callable representing the function f(s) on the sphere.
        L: Maximum band order (default 3).
        
    Returns:
        coeffs: List of (L+1)^2 coefficients.
    """
    num_coeffs = (L + 1) ** 2
    coeffs = [0.0] * num_coeffs
    
    # The exact numerical integration method (e.g., Monte Carlo, quadrature)
    # is not specified in the sources [S1]. The implementer must approximate
    # the integral c_j = \int_S Y_j(s) f(s) ds using a chosen sampling strategy.
    for j in range(num_coeffs):
        # Evaluate integral for coefficient j
        integral_value = 0.0
        # Integration routine would sample directions s_i with weights w_i
        # and accumulate: integral_value += w_i * Y_j(s_i) * f_sample(s_i)
        coeffs[j] = integral_value
        
    return coeffs

def integrate_sh_functions(coeffs_f, coeffs_g):
    """
    Computes the integral of two SH-projected functions via dot product.
    
    Args:
        coeffs_f: Coefficients of function f.
        coeffs_g: Coefficients of function g.
        
    Returns:
        The scalar integral value.
    """
    # I(f, g) = sum_i c_i * d_i
    return sum(c * d for c, d in zip(coeffs_f, coeffs_g))
```
## Subtopics

### [Adaptive Monte-Carlo SH Projection](11255-adaptive-monte-carlo-sh-projection.md)
Adaptive Monte-Carlo SH Projection computes SH coefficients by integrating the product of the environment and SH basis functions over a domain that adapts to local material properties. It distinguishes itself from standard SH projection by dynamically adjusting the integration domain from a full sphere to a hemisphere (plus a transitional ring) based on voxel transparency and gradient magnitude, thereby optimizing the approximation for both translucent volumes and opaque surface-like boundaries.

### Local Material Effects via SH
Computes SH coefficients for material reflectance by integrating over a hemisphere using 1250 samples [S1]. It uses data from the MERL BRDF library [MPBM03] to describe material function $M$ for surface-like structures with high reflectance [S1]. To simplify the rendering pipeline, it uses the same number of SH bands as other projections, relying on the increased sample count to improve approximation quality [S1].

## Operating characteristics
- **Time Complexity:** Projection requires evaluating the integral for each coefficient. Integration of two functions is $O(K)$ where $K=(L+1)^2$, computed as a dot product [S1].
- **Memory Complexity:** $O(K)$ per function/channel to store coefficients [S1].
- **Performance:** Dot-product integration is highly efficient on GPUs [S1]. Precomputing material projections (e.g., 1250 samples for BRDFs) is done once and does not impact runtime [S1].
- **Strengths:** Rotational invariance allows direct application of rotations to SH coefficients, useful for rotating light sources [S1]. Efficient integration via dot products [S1].
- **Limitations:** Three-function integration requires non-physical simplifications for performance [S1].
- **When to use:** When efficient representation of spherical functions is needed for volume rendering, particularly for lighting and material effects [S1].
- **When not to use:** When high-fidelity three-function integration is required and performance constraints allow for more complex techniques [S1].
## Background
The method is described in the context of advanced light-material interaction for direct volume rendering [S1]. It is implemented in a GPU-based volume ray-caster using C++, OpenGL, and GLSL [S1]. It leverages the orthonormality and rotational invariance of spherical harmonics to simplify complex integrations in rendering equations [S1]. The approach uses data from the MERL BRDF library [MPBM03] for material representation [S1].
## Sources
[S1] Advanced Light Material Interaction for Direct Volume Rendering, https://doi.org/10.2312/vg/vg10/101-108, fetched 2026-10-26
