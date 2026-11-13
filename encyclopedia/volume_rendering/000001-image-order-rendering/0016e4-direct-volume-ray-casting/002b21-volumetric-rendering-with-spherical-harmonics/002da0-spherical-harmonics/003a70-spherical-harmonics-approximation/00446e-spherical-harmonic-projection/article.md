# Spherical Harmonic Projection
Also called: SH Compression, SH Integration, Spherical Harmonic (SH) Projection for Volume Rendering
## Overview
Spherical Harmonic (SH) Projection is the process of approximating an arbitrary spherical function $f(\omega)$ by computing the coefficients of its expansion in the spherical harmonics basis. It solves the problem of efficiently representing complex angular data (such as lighting, material reflectance, or in-scattering) for use in volume rendering, enabling fast integration via dot products. It differs from its parent, [Spherical Harmonics Approximation](8741-spherical-harmonics-approximation.md), by focusing specifically on the computational mechanics of deriving these coefficients via least-squares minimization and leveraging the basis's orthonormality for efficient GPU-based integration.
## How it works
**Inputs and Outputs:**
- Input: A spherical function $f(s)$ defined over the domain $S$ (the unit sphere).
- Output: A vector of coefficients $c_j$ representing the projection of $f$ onto the SH basis.

**Data Structures:**
The coefficients are stored as a flat vector or array. For a maximum band order $L$, the number of coefficients is $(L+1)^2$.
- Field `coeffs`: Array of size $(L+1)^2$.
- Field `L`: Integer, maximum band order.

**Parameters:**
- $L$: Maximum SH band order. Typical value: $L=3$ [S1]. This results in $(3+1)^2 = 16$ coefficients per channel [S1].

**Steps:**
1. **Define Basis:** Select the spherical harmonics basis functions $Y_j(s)$ up to order $L$. The basis is orthonormal [S1].
2. **Compute Coefficients:** For each basis function index $j$ (where $0 \le j < (L+1)^2$), compute the coefficient $c_j$ by integrating the product of the basis function and the target function over the sphere:
   $$c_j = \int_S Y_j(s) f(s) ds \quad (1)$$
   This is a least-squares projection that minimizes the squared error between the approximation and the original function $f$ [S1].
3. **Store Coefficients:** Store the computed $c_j$ values in the coefficient vector.

**Integration of Projected Functions:**
Once functions $f$ and $g$ are projected to coefficients $c$ and $d$ respectively, their integration is computed as the dot product of their coefficient vectors:
$$I(\tilde{f}, \tilde{g}) = \sum_i c_i d_i \quad (2)$$
This property allows integration to be performed efficiently on the GPU using a single dot product operation, avoiding the reconstruction and integration of the original functions [S1].

**Pitfalls and Edge Cases:**
- **Three-Function Integration:** Integrating three functions (e.g., Material $M$, In-scattering $L_{mi}^0$, and Light $L_i$) is complex. Standard techniques exist [Sny06], but for performance reasons in volume rendering, a simpler, non-physical approximation is often used instead [S1].
- **Precision:** The accuracy depends on the band order $L$. Higher $L$ captures more detail but increases computational cost and memory [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_harmonic_projection(f_sample, L=3):
    """
    Computes SH coefficients for a spherical function f.
    
    Args:
        f_sample: A callable or array representing the function f(s) on the sphere.
                  Must support evaluation at arbitrary directions s.
        L: Maximum band order (default 3).
        
    Returns:
        coeffs: List of (L+1)^2 coefficients.
    """
    num_coeffs = (L + 1) ** 2
    coeffs = [0.0] * num_coeffs
    
    # Note: The exact numerical integration method (e.g., Monte Carlo, quadrature)
    # is not specified in the source [S1]. The implementer must choose a method
    # to approximate the integral in Equation (1).
    # For Monte Carlo, this would involve sampling directions s_i and weights w_i.
    
    for j in range(num_coeffs):
        # Compute integral: c_j = integral_S Y_j(s) * f(s) ds
        # Placeholder for specific integration routine
        integral_value = integrate_product(Y_j(j, L), f_sample)
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

## Operating characteristics
- **Time Complexity:** The projection step requires evaluating the integral for each coefficient. With $N$ samples and $K=(L+1)^2$ coefficients, the cost is $O(N \cdot K)$. Integration of two functions is $O(K)$ due to the dot product property [S1].
- **Memory Complexity:** $O(K)$ per function/channel to store coefficients [S1].
- **Performance:** The dot-product integration is highly efficient on GPUs [S1]. Precomputing material projections (e.g., using 1250 samples for BRDFs) is done once and does not impact runtime performance [S1].
- **Strengths:** Rotational invariance allows easy rotation of lighting by applying rotations to coefficients [S1]. Efficient integration via dot products [S1].
- **Limitations:** Approximating three-function integrals requires non-physical simplifications for performance [S1]. Quality depends on the number of samples and band order [S1].
- **When to use:** When efficient, differentiable representation of spherical functions is needed for volume rendering, particularly for lighting and material effects.
- **When not to use:** When high-fidelity three-function integration is required and performance constraints allow for more complex techniques [S1].
## Background
The method is described in the context of advanced light-material interaction for direct volume rendering. It leverages the orthonormality and rotational invariance of spherical harmonics to simplify complex integrations in rendering equations [S1]. The approach uses data from the MERL BRDF library [MPBM03] for material representation [S1].
## Sources
[S1] Advanced Light Material Interaction for Direct Volume Rendering, https://doi.org/10.2312/vg/vg10/101-108, fetched 2026-10-26
