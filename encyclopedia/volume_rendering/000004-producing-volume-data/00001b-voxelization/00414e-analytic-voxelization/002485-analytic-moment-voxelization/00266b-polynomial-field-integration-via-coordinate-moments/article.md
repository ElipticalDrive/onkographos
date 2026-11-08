# Polynomial Field Integration via Coordinate Moments

Also called: Coordinate Moment Integration, Polynomial Coefficient Integration

## Overview
This method computes the integral of arbitrary scalar fields (or component-wise vector fields) over a voxel by evaluating coordinate moments and combining them via polynomial coefficients [S1, unofficial]. It allows the simultaneous integration of multiple fields without re-clipping voxels, distinguishing it from methods that require per-field geometric intersection tests. It builds on [Analytic Moment Voxelization](3307-analytic-moment-voxelization.md).

## How it works
**Inputs:**
- A voxel volume $V$ (defined by its intersection with a polyhedron).
- A scalar field $f(x,y,z)$ represented as a polynomial of order $N$.
- The polynomial coefficients of $f$.

**Outputs:**
- The integral $\int_V f(x,y,z) \, dV$.

**Data Structures:**
- **Coordinate Moments:** A list or array storing values of $\int_V x^i y^j z^k \, dV$ for all monomials up to order $N$. For a second-order polynomial, this includes $\int_V 1 \, dV, \int_V x \, dV, \int_V y \, dV, \int_V z \, dV, \int_V x^2 \, dV, \int_V y^2 \, dV, \int_V z^2 \, dV, \int_V xy \, dV, \int_V xz \, dV, \int_V yz \, dV$.
- **Polynomial Coefficients:** A vector or array containing the weights $A, B, C, \dots, J$ for the polynomial terms.

**Steps:**
1. **Compute Coordinate Moments:** For the given voxel volume $V$, calculate the integral of each basis monomial $x^i y^j z^k$ up to the desired polynomial order. This is typically done using the geometric clipping and simplicial decomposition methods provided by the parent technique [Analytic Moment Voxelization](3307-analytic-moment-voxelization.md).
2. **Linear Combination:** Multiply each computed coordinate moment by its corresponding polynomial coefficient from the field representation.
3. **Summation:** Sum the products to obtain the total integral.

**Formula:**
For a second-order polynomial field $f(x,y,z) = Ax^2 + By^2 + Cz^2 + Dxy + Exz + Fyz + Gx + Hy + Iz + J$, the integral is:

$$
\int_V f(x,y,z) \, dV = A \int_V x^2 \, dV + B \int_V y^2 \, dV + C \int_V z^2 \, dV + D \int_V xy \, dV + E \int_V xz \, dV + F \int_V yz \, dV + G \int_V x \, dV + H \int_V y \, dV + I \int_V z \, dV + J \int_V 1 \, dV
$$

**Parameters:**
- Polynomial order: Determines the number of moments required. Second-order is common for smooth fields.

**Edge Cases and Pitfalls:**
- **Precision:** High-order moments can suffer from numerical instability if the voxel is far from the origin. Use local coordinates if available (see [Cancellation Error Reduction via Local Coordinates](5926-cancellation-error-reduction-via-local-coordinates.md)).
- **Degenerate Volumes:** If $V$ is zero (voxel outside the polyhedron), all moments are zero.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def integrate_polynomial_field(voxel_volume, polynomial_coeffs, moment_order):
    """
    Integrate a polynomial field over a voxel volume using precomputed coordinate moments.
    
    Args:
        voxel_volume: Object representing the volume V, with methods to compute moments.
        polynomial_coeffs: List of coefficients [A, B, C, D, E, F, G, H, I, J] for 2nd order.
        moment_order: Integer, max degree of monomials to compute.
        
    Returns:
        Float: The integral of the field over the volume.
    """
    # Step 1: Compute all coordinate moments for the desired order
    # This relies on the parent method's geometric capabilities.
    # We assume a helper function that returns a dictionary of moments.
    moments = compute_coordinate_moments(voxel_volume, moment_order)
    
    # Step 2: Map coefficients to monomials
    # For 2nd order:
    # x^2 -> A, y^2 -> B, z^2 -> C, xy -> D, xz -> E, yz -> F
    # x -> G, y -> H, z -> I, 1 -> J
    
    integral = 0.0
    
    # Add contributions from each term
    integral += polynomial_coeffs[0] * moments['x^2']
    integral += polynomial_coeffs[1] * moments['y^2']
    integral += polynomial_coeffs[2] * moments['z^2']
    integral += polynomial_coeffs[3] * moments['xy']
    integral += polynomial_coeffs[4] * moments['xz']
    integral += polynomial_coeffs[5] * moments['yz']
    integral += polynomial_coeffs[6] * moments['x']
    integral += polynomial_coeffs[7] * moments['y']
    integral += polynomial_coeffs[8] * moments['z']
    integral += polynomial_coeffs[9] * moments['1'] # Volume
    
    return integral

def compute_coordinate_moments(voxel_volume, order):
    """
    Computes integrals of monomials x^i y^j z^k over the voxel volume.
    Implementation details depend on the specific analytic voxelization algorithm.
    """
    # Placeholder for the actual geometric integration logic from Analytic Moment Voxelization
    # This function would iterate over all combinations of i, j, k such that i+j+k <= order
    # and compute the integral using the parent method's simplicial decomposition.
    # For brevity, we assume this is implemented in the parent article.
    raise NotImplementedError("See Analytic Moment Voxelization for implementation details")
```

## Subtopics

### [Cancellation Error Reduction via Local Coordinates](5926-cancellation-error-reduction-via-local-coordinates.md)
Mitigates cancellation errors in volume and moment calculations for simplicial decomposition by shifting the integration domain into a local coordinate system near the origin before calculation.

## Operating characteristics
- **Time Complexity:** Dominated by the computation of coordinate moments, which depends on the complexity of the voxel-polyhedron intersection and the polynomial order. Once moments are computed, integration is $O(1)$ with respect to the number of fields.
- **Memory Complexity:** Proportional to the number of moments, which grows as $O(N^3)$ for order $N$ in 3D.
- **Strengths:** Efficient for integrating many different fields over the same geometry, as moments are computed once. Avoids redundant geometric clipping.
- **Limitations:** Requires pre-computation of moments. High-order polynomials may require significant memory and precision.
- **When to use:** When integrating multiple scalar fields or performing operations like gradient estimation on the same voxelized geometry.
- **When not to use:** For single-field integration where geometric clipping is cheap, or for non-polynomial fields that cannot be easily approximated by polynomials.

## Background
Originated in the context of exact general remeshing schemes for conservative voxelization [S1, unofficial]. The method leverages the linearity of integration to separate geometric computation (moments) from field-specific computation (coefficients).

## Sources
[S1] An exact general remeshing scheme applied to conservative voxelization, https://arxiv.org/pdf/1412.4941, fetched 2026-10-26 (unofficial)
