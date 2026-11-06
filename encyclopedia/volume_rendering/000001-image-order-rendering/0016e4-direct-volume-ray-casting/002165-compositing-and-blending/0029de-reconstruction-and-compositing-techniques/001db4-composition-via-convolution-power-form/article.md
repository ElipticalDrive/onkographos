# Composition via Convolution (Power Form)
Also called: Power Form

## Overview
Composition via Convolution (Power Form) evaluates the composition of a trivariate polynomial field with a linear ray parameterization, transforming a 3D function into a univariate function along a ray [S1]. It solves the problem of efficient numerical evaluation in direct volume ray casting by using convolution operators to reduce computation and register usage compared to naive substitution [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Trivariate polynomial coefficients $f_{ijk}$ of degree $d$ in each dimension (total degree $3d$), and ray parameters $p_x, v_x, p_y, v_y, p_z, v_z$ defining the ray $r(t) = p + t \cdot v$ [S1].
*   **Output:** Coefficients of the resulting univariate polynomial $f(t)$ of degree $3d$ [S1].

**Data Structures**
*   **Ray Coefficient Sequences:** For each axis, a sequence representing the linear polynomial components [S1]. For the x-axis, $\mathbf{r}_x = [p_x, v_x]$ [S1]. Similarly, $\mathbf{r}_y = [p_y, v_y]$ and $\mathbf{r}_z = [p_z, v_z]$ [S1].
*   **Power Expressions:** Incrementally computed convolution results where $\mathbf{r}_x^i$ represents the $i$-th power of the x-component ray polynomial [S1].

**Numbered Steps**
1.  **Initialize Ray Powers:** Compute the power expressions for each axis incrementally using convolution [S1]. For $i$ from 1 to $d$: $\mathbf{r}_x^i = \mathbf{r}_x^{i-1} * \mathbf{r}_x$ [S1]. Repeat for $\mathbf{r}_y^j$ and $\mathbf{r}_z^k$ [S1]. This is equivalent to the Horner scheme for polynomial evaluation [S1].
2.  **Compute Inner Sum:** For each pair $(i, j)$, compute the intermediate polynomial $\mathbf{f}_{ij}$ by convolving the z-axis ray powers with the corresponding polynomial coefficients: $\mathbf{f}_{ij} = \sum_{k=0}^d f_{ijk} \cdot \mathbf{r}_z^k$ [S1].
3.  **Compute Middle Sum:** For each $i$, compute the intermediate polynomial $\mathbf{g}_i$ by convolving the y-axis ray powers with the previously computed $\mathbf{f}_{ij}$: $\mathbf{g}_i = \sum_{j=0}^d \mathbf{r}_y^j * \mathbf{f}_{ij}$ [S1].
4.  **Compute Final Sum:** Compute the final univariate polynomial coefficients $\mathbf{f}$ by convolving the x-axis ray powers with the intermediate polynomials $\mathbf{g}_i$: $\mathbf{f} = \sum_{i=0}^d \mathbf{r}_x^i * \mathbf{g}_i$ [S1].

**Formulas**
The core composition formula using convolution operators is:
$$ \mathbf{f} = \sum_{i=0}^d \sum_{j=0}^d \sum_{k=0}^d f_{ijk} \cdot \mathbf{r}_x^i * \mathbf{r}_y^j * \mathbf{r}_z^k $$
Where $\mathbf{r}_x = [p_x, v_x]$ is the coefficient sequence of the x-component of the ray [S1].

The reordered composition, which enables the memory-efficient iterative approach, is:
$$ \mathbf{f} = \sum_{i=0}^d \mathbf{r}_x^i * \sum_{j=0}^d \mathbf{r}_y^j * \underbrace{\sum_{k=0}^d f_{ijk} \cdot \mathbf{r}_z^k}_{\mathbf{f}_{ij}} $$

**Parameters**
*   $d$: The degree of the polynomial in each dimension. Typical values in the source are $d=3$ (total degree 9) [S1].

**Edge Cases and Pitfalls**
*   **Register Pressure:** A naive implementation requires too many operations and registers to fit into a GPU framework [S1]. The convolution structure ensures that no more than one factor $\mathbf{f}_{ij}$ and $\mathbf{g}_i$ needs to be available at the same time, allowing memory reuse [S1].
*   **Precision:** The method exhibits lower Mean Squared Error (MSE) compared to naive substitution, indicating better numerical stability for the given test cases [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def composition_via_convolution_power_form(f_coeffs, ray_p, ray_v):
    """
    Computes the composition of a trivariate polynomial with a linear ray.
    
    Args:
        f_coeffs: 3D array of coefficients f_ijk of shape (d+1, d+1, d+1).
        ray_p: Array [px, py, pz] of ray origins.
        ray_v: Array [vx, vy, vz] of ray directions.
        
    Returns:
        Array of coefficients for the resulting univariate polynomial.
    """
    d = f_coeffs.shape[0] - 1
    
    # Step 1: Compute ray power sequences via convolution
    # r_x[i] is the coefficient sequence for (px + t*vx)^i
    r_x = [[1.0]]
    for i in range(1, d + 1):
        r_x.append(convolve(r_x[-1], [ray_p[0], ray_v[0]]))
        
    r_y = [[1.0]]
    for j in range(1, d + 1):
        r_y.append(convolve(r_y[-1], [ray_p[1], ray_v[1]]))
        
    r_z = [[1.0]]
    for k in range(1, d + 1):
        r_z.append(convolve(r_z[-1], [ray_p[2], ray_v[2]]))
        
    # Steps 2-4: Iterative composition with memory reuse
    # The nested loop structure ensures only one f_ij and one g_i 
    # are held in registers at any time, matching the source's claim [S1].
    final_f = [0.0] * (3 * d + 1)
    
    for i in range(d + 1):
        for j in range(d + 1):
            # Compute f_ij = sum_k f_ijk * r_z[k]
            f_ij = [0.0] * (d + 1)
            for k in range(d + 1):
                if f_coeffs[i, j, k] != 0.0:
                    scaled_rz = scale(r_z[k], f_coeffs[i, j, k])
                    f_ij = add_polynomials(f_ij, scaled_rz)
            
            # Compute g_i += r_y[j] * f_ij
            g_i = convolve(r_y[j], f_ij)
            
            # Compute final_f += r_x[i] * g_i
            final_f = add_polynomials(final_f, convolve(r_x[i], g_i))
            
    return final_f

def convolve(poly1, poly2):
    """Convolution of two polynomial coefficient arrays."""
    deg1 = len(poly1) - 1
    deg2 = len(poly2) - 1
    result = [0.0] * (deg1 + deg2 + 1)
    for i, c1 in enumerate(poly1):
        for j, c2 in enumerate(poly2):
            result[i + j] += c1 * c2
    return result

def add_polynomials(p1, p2):
    """Add two polynomial coefficient arrays."""
    max_len = max(len(p1), len(p2))
    result = [0.0] * max_len
    for i in range(len(p1)):
        result[i] += p1[i]
    for i in range(len(p2)):
        result[i] += p2[i]
    return result

def scale(poly, scalar):
    """Scale a polynomial by a scalar."""
    return [c * scalar for c in poly]
```

## Subtopics

### [Composition via Scaled Bernstein Form](1592-composition-via-scaled-bernstein-form.md)
Composition via Scaled Bernstein Form (SB) evaluates the composition of a trivariate polynomial field with a ray parameterization, transforming a 3D function into a univariate function along a ray. It solves the problem of efficient numerical evaluation in direct volume ray casting by leveraging the scaled Bernstein basis. Unlike [Composition via Convolution (Power Form)](1591-composition-via-convolution-power-form.md), which uses power bases and convolution operators, SB maintains a uniform data representation throughout the pipeline, offering better numerical stability and avoiding the need for basis conversion when root finding is required.

### [Basis Transform for Trivariate Polynomials](1609-basis-transform-for-trivariate-polynomials.md)
This method transforms a trivariate polynomial representation from the B-spline basis (used in tri-linear or tri-cubic filtering) to the scaled Bernstein or power basis. It solves the problem of high memory consumption when storing explicit polynomial descriptions for large datasets, allowing on-the-fly transformation instead. It is distinct from naive substitution by leveraging the tensor product structure of the basis to reduce computational complexity, and differs from [Composition via Scaled Bernstein Form](1592-composition-via-scaled-bernstein-form.md) by its specific focus on the initial basis conversion step for efficient Horner scheme integration.

## Operating characteristics
*   **Time Complexity:** The method significantly reduces the number of multiplications and additions compared to naive substitution [S1]. For a degree $d=3$ polynomial, it requires 234 multiplications and 360 additions, compared to 1839 multiplications and 990 additions for the naive method [S1].
*   **Memory Complexity:** The method uses only 27 registers, compared to 135 registers for the naive method [S1]. This is achieved by reusing memory for intermediate factors $\mathbf{f}_{ij}$ and $\mathbf{g}_i$ [S1].
*   **Measured Performance:** In tests on a GPU framework, the Power Form method achieved a relative execution time of $3.6 \times 10^{-3}$ compared to the naive method (relative time 1) [S1].
*   **Accuracy:** The Mean Squared Error (MSE) for the Power Form method was $22.6 \times 10^{-12}$, compared to $162.3 \times 10^{-12}$ for the naive method [S1].
*   **Strengths:** High performance on GPUs due to low register usage and reduced operation count [S1]. Good numerical stability [S1].
*   **Limitations:** Requires basis conversion if Bernstein form is needed for root finding, unlike the Scaled Bernstein Form method [S1].
*   **When to use:** When evaluating trivariate polynomial fields on GPUs where register pressure is a bottleneck and root finding is not required immediately [S1].
*   **When not to use:** When the downstream pipeline requires root finding in Bernstein basis without incurring basis conversion overhead [S1].

## Background
The method is described in the paper *Wavelet-based Multiresolution Isosurface Rendering* [S1]. It builds on the idea of using convolution operators to efficiently evaluate polynomial compositions, as described in [SR03] (cited in [S1]). The approach addresses the limitations of naive substitution, which is too computationally expensive and register-intensive for GPU-based frameworks [S1].

## Sources
[S1] Wavelet-based Multiresolution Isosurface Rendering, https://doi.org/10.2312/vg/vg10/013-020, fetched 2026-10-26
