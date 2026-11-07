# Analytic Gradient Computation via Box-Spline Derivatives

Also called: Derivative Spline Evaluation, Exact Gradient for FCC

## Overview
This method computes the exact gradient vector for volume rendering shading by deriving analytic derivatives of the underlying box-spline basis function, avoiding the approximation errors and additional sampling costs of finite differences. It is distinguished from its parent method, [Analytic Gradient Computation](2167-analytic-gradient-computation.md), by specifically leveraging the symmetry and support properties of Face-Centered Cubic (FCC) box-splines to reuse existing texture fetches. Unlike [Analytic Gradient Computation for Triangle Distributions](5829-analytic-gradient-computation-for-triangle-distributions.md), which handles triangular supports, this method operates on the hexagonal/triangular lattice structure of FCC grids.

## How it works
**Inputs and Outputs:**
- Input: Volume data $V$ defined on an FCC grid, position $x$ within the grid.
- Output: Gradient vector $\nabla s(x)$ where $s(x) = V * M_{\text{fcc}}$ is the reconstructed scalar field.

**Data Structures:**
- The method relies on the precomputed B-spline coefficients (BB-coefficients) for the derivative kernel $M_{110}$ across three polynomial piece types: $\tau_0, \tau_2, \tau_4$. These coefficients define the stencil weights for the derivative evaluation.
- The support of the derivative kernel $M_{110}$ is a subset of the support of the primary FCC box-spline $M_{\text{fcc}}$.

**Steps:**
1. **Fetch Samples:** Retrieve the 19 voxel values required for the primary FCC box-spline evaluation at position $x$. These samples are reused for gradient computation.
2. **Compute Directional Derivatives:** Calculate three directional derivatives $d_{110}, d_{101}, d_{011}$ along the vectors $\xi_{110}, \xi_{101}, \xi_{011}$ respectively.
   - Due to symmetry, only the BB-coefficients for $M_{110}$ are explicitly needed. The kernels for $M_{101}$ and $M_{011}$ are obtained via permutation.
   - Evaluate the derivative kernel $M_{110}$ using the de Casteljau algorithm. Because $M_{110}$ has lower symmetry than $M_{\text{fcc}}$, the evaluation must distinguish between three polynomial piece types: $\tau_0, \tau_2$, and $\tau_4$.
   - The directional derivative is defined as $d_\xi = V * M_\xi$.
3. **Reconstruct Gradient:** Combine the directional derivatives to form the Cartesian gradient vector using the following linear combination:
   $$
   \nabla s(x) = \frac{1}{2} \begin{bmatrix} d_{110} + d_{101} - d_{011} \\ d_{110} + d_{011} - d_{101} \\ d_{101} + d_{011} - d_{110} \end{bmatrix}
   $$

**Formulas:**
The directional derivative of the FCC box-spline $M_{\text{fcc}}$ along a direction $\xi \in \Xi_{\text{fcc}}$ is expressed as another box-spline $M_\xi$:
$$
D_\xi M_{\text{fcc}} = \nabla_\xi 2M_{\Xi_{\text{fcc}} \setminus \{\xi\}}(\cdot + \xi_c)
$$
where $\nabla_\xi$ is the backward difference operator. The kernel $M_\xi$ is defined as:
$$
M_\xi(x) = 2M_{\Xi_{\text{rec}} \setminus \{\xi\}}(x + \xi_c) - 2M_{\Xi_{\text{rec}} \setminus \{\xi\}}(x + \xi_c - \xi)
$$
Here, $\Xi_{\text{rec}}$ represents the set of directions defining the reconstruction spline, and $\xi_c$ is a centering vector.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_analytic_gradient_fcc(volume_data, position):
    """
    Computes the gradient of the FCC box-spline reconstruction at position.
    Assumes volume_data provides access to the 19 samples needed for M_fcc.
    """
    # Step 1: Fetch the 19 samples required for the primary FCC spline evaluation.
    # These are stored in a list/array 'samples' indexed by their relative offset.
    samples = fetch_fcc_samples(volume_data, position)
    
    # Step 2: Compute directional derivative d_110
    # Determine the polynomial piece type (tau_0, tau_2, or tau_4) for position
    # based on the local lattice coordinates.
    piece_type = get_polynomial_piece_type(position)
    
    # Get the precomputed BB-coefficients for M_110 for the specific piece type.
    # These coefficients define the stencil weights.
    coeffs_110 = get_bb_coefficients_M110(piece_type)
    
    # Evaluate M_110 using de Casteljau algorithm or direct stencil convolution
    # using the fetched samples and the coefficients.
    d_110 = evaluate_derivative_kernel(samples, coeffs_110)
    
    # Step 3: Compute directional derivatives d_101 and d_011
    # Due to symmetry, these are permutations of d_110.
    # The implementation must permute the sample indices and coefficients accordingly.
    d_101 = evaluate_derivative_kernel_permuted(samples, coeffs_110, direction='101')
    d_011 = evaluate_derivative_kernel_permuted(samples, coeffs_110, direction='011')
    
    # Step 4: Reconstruct the gradient vector
    grad_x = 0.5 * (d_110 + d_101 - d_011)
    grad_y = 0.5 * (d_110 + d_011 - d_101)
    grad_z = 0.5 * (d_101 + d_011 - d_110)
    
    return Vector3(grad_x, grad_y, grad_z)
```

**Parameters:**
- Polynomial piece types: $\tau_0, \tau_2, \tau_4$. The specific type is determined by the position of the query point within the FCC lattice unit cell.
- Stencil size: 19 samples (same as primary FCC spline evaluation).

**Edge Cases and Pitfalls:**
- **Precision:** The method avoids finite difference step-size selection errors. However, the de Casteljau algorithm and linear combinations must be implemented with sufficient floating-point precision to avoid accumulation errors, especially for high-resolution volumes.
- **Boundaries:** The support of $M_{110}$ is a subset of $M_{\text{fcc}}$. Boundary conditions must be handled consistently with the primary spline evaluation (e.g., clamping, wrapping, or zero-padding) to ensure the reused samples are valid.
- **Degenerate Input:** If the volume data is not defined on a regular FCC grid, the lattice coordinate calculation for piece type determination will fail.

## Subtopics

### [Analytic Normal Computation](3205-analytic-normal-computation.md)
Analytic Normal Computation computes surface normal vectors for volume rendering by deriving the gradient directly from the underlying spline basis function, rather than approximating it via finite differences. It solves the problem of normal inaccuracy inherent in numerical differencing, where the choice of offset step size creates a trade-off between rounding errors (small offsets) and approximation errors (large offsets). It is distinguished from its parent method, Analytic Gradient Computation via Box-Spline Derivatives, by specifically focusing on the normal vector derivation for isosurface rendering, ensuring consistent accuracy regardless of zoom level or viewing angle.

## Operating characteristics
- **Time Complexity:** The method requires evaluating three derivative kernels ($M_{110}, M_{101}, M_{011}$). However, due to symmetry, only one set of coefficients is needed, and the same 19 samples are reused. The derivative kernels are quadratic, allowing for fast evaluation via the de Casteljau algorithm.
- **Memory Complexity:** No additional memory is required beyond the 19 samples fetched for the primary spline evaluation.
- **Performance:** This method is significantly faster than finite difference approaches, which require six additional spline evaluations (114 additional samples) to compute the gradient. The reuse of samples reduces memory bandwidth pressure.
- **Strengths:** Exact gradient computation; no approximation error from step size; reduced memory bandwidth compared to finite differences.
- **Limitations:** Requires precomputed BB-coefficients for the derivative kernels; implementation complexity is higher than finite differences due to the need to handle polynomial piece types and symmetry permutations.

## Background
This method was introduced by Minho Kim and Jürgen Peters in their 2012 paper "GPU Isosurface Raycasting of FCC Datasets" [S1]. It addresses the need for efficient and accurate gradient computation for volume rendering on Face-Centered Cubic (FCC) grids, which are known for their optimal packing density and interpolation properties. The method leverages the mathematical property that the directional derivative of a box-spline is another box-spline, allowing for analytic evaluation without numerical approximation.

## Sources
[S1] GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
