# Basis Transform for Trivariate Polynomials

Also called: B-spline to Bernstein/Power Basis Transform

## Overview
This method transforms a trivariate polynomial representation from the B-spline basis (used in tri-linear or tri-cubic filtering) to the scaled Bernstein or power basis [S1]. It solves the problem of high memory consumption when storing explicit polynomial descriptions for large datasets, allowing on-the-fly transformation instead [S1]. It is distinct from naive substitution by leveraging the tensor product structure of the basis to reduce computational complexity, and differs from [Composition via Scaled Bernstein Form](1592-composition-via-scaled-bernstein-form.md) by its specific focus on the initial basis conversion step for efficient Horner scheme integration [S1].

## How it works
**Inputs:**
- Polynomial degree $d$ (corresponding to the filtering order, e.g., $d=1$ for tri-linear, $d=3$ for tri-cubic) [S1].
- Data values in the B-spline basis for a voxel [S1].

**Outputs:**
- Coefficients in the scaled Bernstein or power basis [S1].

**Data Structures:**
- Basis vector $\mathbf{b}$: A 1D array of size $d+1$ representing the B-spline coefficients [S1].
- Transformation Matrix $M$: A $(d+1) \times (d+1)$ matrix containing constants (zeros, ones, and a single scaling factor) [S1].

**Steps:**
1. **One-Dimensional Basis Transform:** For each dimension, compute the transformation from B-spline to the target basis by multiplying the basis vector by the transformation matrix $M$ [S1]. The matrix $M$ is precomputed to contain mostly zeros and ones, scaled by a constant factor to minimize arithmetic operations [S1]. Transition to the scaled Bernstein basis requires 6 multiplications per dimension, while transition to the power basis requires 8 multiplications per dimension [S1].
2. **N-Dimensional Basis Transform Construction:** The N-dimensional basis is a tensor product of the 1D bases [S1]. The full N-dimensional transform is constructed from $N \cdot (d + 1)^{N-1}$ one-dimensional basis transforms [S1]. For trivariate polynomials ($N=3$), this involves applying the 1D transforms sequentially along each axis [S1].
3. **Integration with Horner Scheme:** The basis transform is integrated into the Horner scheme for polynomial evaluation/composition [S1]. This separation of dimensions allows efficient evaluation without explicitly storing the full trivariate polynomial coefficients [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_1d_transform(coeffs, degree, M):
    """Applies the precomputed sparse transformation matrix M to a 1D coefficient vector."""
    out = [0.0] * (degree + 1)
    for i in range(degree + 1):
        for j in range(degree + 1):
            out[i] += M[i][j] * coeffs[j]
    return out

def basis_transform_3d(b_spline_coeffs, degree, M):
    """
    Transforms trivariate B-spline coefficients to target basis using tensor product.
    Applies 1D transforms sequentially along each axis.
    """
    # Allocate output array
    out = [[[0.0] * (degree + 1) for _ in range(degree + 1)] for _ in range(degree + 1)]

    # Transform along X (axis 0)
    temp_x = [[[0.0] * (degree + 1) for _ in range(degree + 1)] for _ in range(degree + 1)]
    for k in range(degree + 1):
        for l in range(degree + 1):
            col = [b_spline_coeffs[i][k][l] for i in range(degree + 1)]
            transformed_col = apply_1d_transform(col, degree, M)
            for i in range(degree + 1):
                temp_x[i][k][l] = transformed_col[i]

    # Transform along Y (axis 1)
    temp_y = [[[0.0] * (degree + 1) for _ in range(degree + 1)] for _ in range(degree + 1)]
    for k in range(degree + 1):
        for l in range(degree + 1):
            row = [temp_x[k][j][l] for j in range(degree + 1)]
            transformed_row = apply_1d_transform(row, degree, M)
            for j in range(degree + 1):
                temp_y[k][j][l] = transformed_row[j]

    # Transform along Z (axis 2)
    for k in range(degree + 1):
        for l in range(degree + 1):
            plane = [temp_y[k][l][m] for m in range(degree + 1)]
            transformed_plane = apply_1d_transform(plane, degree, M)
            for m in range(degree + 1):
                out[k][l][m] = transformed_plane[m]

    return out
```

**Parameters:**
- Degree $d$ (filtering order) [S1].
- Multiplications per 1D transform: 6 for scaled Bernstein, 8 for power basis [S1].

**Edge Cases:**
- Degree 0 is a trivial case requiring no transformation [G].
- Using matrices with zeros and ones reduces the number of required arithmetic operations rather than affecting floating-point precision [S1].

## Subtopics

### [Polynomial Basis Vector Generation](3511-polynomial-basis-vector-generation.md)
Generates the basis vector $\mathbf{b}$ for a given degree $d$ [G]. It constructs the monomial or Bernstein basis functions required for the transformation [G].

## Operating characteristics
- **Time Complexity:** The transformation requires $N \cdot (d + 1)^{N-1}$ 1D transforms, resulting in a fixed number of multiplications (6 or 8 per dimension) due to the sparse matrix structure [S1].
- **Memory Complexity:** Avoids storing the full explicit polynomial description, reducing memory consumption by a factor of $(d+1)^3$ compared to storing all coefficients explicitly [S1].
- **Strengths:** Efficient on-the-fly transformation; low arithmetic complexity; compatible with Horner scheme for fast evaluation [S1].
- **Limitations:** Requires precomputed transformation matrices; specific to B-spline to Bernstein/Power basis conversion [S1].
- **When to Use:** When memory is constrained and the polynomial needs to be evaluated in a different basis (e.g., for root finding or composition) [S1].
- **When Not to Use:** If the B-spline basis is sufficient for the application (e.g., simple interpolation), as working directly with the original representation avoids unnecessary transformation overhead [S1].

## Background
The method is used in volume rendering pipelines that employ B-spline filtering (tri-linear/tri-cubic) and require efficient root finding or composition [S1]. Tri-linear and tri-cubic filtering are based on B-spline polynomials, and transforming to Bernstein or power basis enables efficient root finding and composition [S1]. The approach was developed to address the memory overhead of storing explicit polynomial descriptions for large volumes [S1].

## Sources
[S1] Wavelet-based Multiresolution Isosurface Rendering, https://doi.org/10.2312/vg/vg10/013-020, fetched 2026-10-26
