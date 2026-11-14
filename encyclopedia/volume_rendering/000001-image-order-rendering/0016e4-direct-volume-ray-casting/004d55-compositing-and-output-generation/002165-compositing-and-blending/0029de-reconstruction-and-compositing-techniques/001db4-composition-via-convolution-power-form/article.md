# Composition via Convolution (Power Form)

Also called: Power Form

## Overview
Composition via Convolution (Power Form) is a method for evaluating the composition of a trivariate polynomial field with a linear ray parameterization, transforming the 3D function into a univariate polynomial along the ray [S1]. It solves the problem of high computational cost and register pressure associated with naive polynomial substitution, which prevents efficient implementation on GPU architectures [S1]. Unlike [Composition via Scaled Bernstein Form](1592-composition-via-scaled-bernstein-form.md), which maintains a uniform data representation for better numerical stability and root finding, the Power Form uses power bases and convolution operators to minimize operation counts and register usage, offering superior performance for direct evaluation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A trivariate polynomial $f(x,y,z)$ of maximum degree $3d$, represented by coefficients $f_{ijk}$ for $0 \le i,j,k \le d$ [S1]. The ray is defined by its linear components: $\mathbf{r}_x = [p_x, v_x]$, $\mathbf{r}_y = [p_y, v_y]$, $\mathbf{r}_z = [p_z, v_z]$, representing the constant and velocity terms of the ray's projection onto the axes [S1].
*   **Output:** The coefficient sequence $\mathbf{f}$ of the resulting univariate polynomial $f(t)$ of degree $3d$ [S1].

**Data Structures:**
*   Coefficients $f_{ijk}$: Stored as floating-point values [S1].
*   Ray components $\mathbf{r}_x, \mathbf{r}_y, \mathbf{r}_z$: Represented as pairs of floats $(p, v)$ [S1].
*   Intermediate convolution results $\mathbf{r}_x^i, \mathbf{r}_y^j, \mathbf{r}_z^k$: Stored as coefficient sequences (arrays/lists of floats) [S1].

**Numbered Steps:**
1.  **Initialize Ray Powers:** Compute the power expressions for the ray components incrementally using convolution. The base case is $\mathbf{r}_x^0 = [1]$ (representing $t^0$) [S1]. For $i$ from 1 to $d$:
    $$ \mathbf{r}_x^i = \mathbf{r}_x^{i-1} * \mathbf{r}_x $$
    where $*$ denotes polynomial convolution [S1]. This is equivalent to the Horner scheme for polynomial evaluation [S1]. Repeat for $\mathbf{r}_y^j$ and $\mathbf{r}_z^k$ for $j, k$ from 1 to $d$ [S1].
2.  **Compute Inner Sums (Z-direction):** For each pair $(i, j)$, compute the intermediate polynomial $\mathbf{f}_{ij}$ by convolving the polynomial coefficients $f_{ijk}$ with the z-ray powers $\mathbf{r}_z^k$:
    $$ \mathbf{f}_{ij} = \sum_{k=0}^d f_{ijk} \cdot \mathbf{r}_z^k $$
    where $\cdot$ denotes scalar multiplication [S1].
3.  **Compute Middle Sums (Y-direction):** For each $i$, compute the intermediate polynomial $\mathbf{g}_i$ by convolving $\mathbf{f}_{ij}$ with the y-ray powers $\mathbf{r}_y^j$:
    $$ \mathbf{g}_i = \sum_{j=0}^d \mathbf{r}_y^j * \mathbf{f}_{ij} $$
4.  **Compute Outer Sum (X-direction):** Compute the final univariate polynomial coefficients $\mathbf{f}$ by convolving $\mathbf{g}_i$ with the x-ray powers $\mathbf{r}_x^i$:
    $$ \mathbf{f} = \sum_{i=0}^d \mathbf{r}_x^i * \mathbf{g}_i $$

**Formulas:**
The core reordered composition formula is:
$$ \mathbf{f} = \sum_{i=0}^d \mathbf{r}_x^i * \sum_{j=0}^d \mathbf{r}_y^j * \underbrace{\sum_{k=0}^d f_{ijk} \cdot \mathbf{r}_z^k}_{\mathbf{f}_{ij}} $$
where $\mathbf{r}_x = [p_x, v_x]$ is the coefficient sequence of the x-component of the ray [S1].

**Parameters:**
*   $d$: The polynomial degree parameter. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Register Pressure:** A naive implementation requires too many operations and registers to fit into a GPU framework (e.g., 135 registers for degree 3) [S1]. The convolution structure ensures that no more than one factor $f_{ij}$ and $f_i$ needs to be available at the same time, allowing memory reuse (e.g., reusing the memory slot of $f_{00}$ for $f_{01}$) [S1].
*   **Precision:** The Power Form has a higher Mean Squared Error (MSE) compared to Scaled Bernstein forms (22.6e-12 vs 4.78e-12 for degree 3) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compose_power_form(f_coeffs, ray_x, ray_y, ray_z, d):
    """
    f_coeffs: 3D list [i][j][k] of polynomial coefficients f_ijk for 0 <= i,j,k <= d
    ray_x, ray_y, ray_z: tuples (p, v) representing the linear ray components
    d: degree of the polynomial
    Returns: 1D list of coefficients for the resulting univariate polynomial f(t)
    """
    
    def convolve(a, b):
        res = [0.0] * (len(a) + len(b) - 1)
        for i, ca in enumerate(a):
            for j, cb in enumerate(b):
                res[i + j] += ca * cb
        return res

    def add_polys(a, b):
        res = [0.0] * max(len(a), len(b))
        for i, ca in enumerate(a): res[i] += ca
        for i, cb in enumerate(b): res[i] += cb
        return res

    # 1. Compute ray powers incrementally: r^0 = [1], r^i = r^(i-1) * [p, v]
    def compute_ray_powers(ray, d):
        powers = [[1.0]]
        base = [ray[0], ray[1]]
        curr = [1.0]
        for _ in range(d):
            curr = convolve(curr, base)
            powers.append(curr)
        return powers

    r_x_powers = compute_ray_powers(ray_x, d)
    r_y_powers = compute_ray_powers(ray_y, d)
    r_z_powers = compute_ray_powers(ray_z, d)

    # 2-4. Compute nested sums with memory reuse (only one f_ij and f_i needed at a time)
    g_i = [None] * (d + 1)
    for i in range(d + 1):
        sum_gi = [0.0]
        for j in range(d + 1):
            sum_fij = [0.0]
            for k in range(d + 1):
                term = [c * f_coeffs[i][j][k] for c in r_z_powers[k]]
                sum_fij = add_polys(sum_fij, term)
            term_gi = convolve(r_y_powers[j], sum_fij)
            sum_gi = add_polys(sum_gi, term_gi)
        g_i[i] = sum_gi

    final_f = [0.0]
    for i in range(d + 1):
        term = convolve(r_x_powers[i], g_i[i])
        final_f = add_polys(final_f, term)
    return final_f
```

## Subtopics

### [Composition via Scaled Bernstein Form](1592-composition-via-scaled-bernstein-form.md)
Evaluates composition using the scaled Bernstein basis. Chosen when numerical stability and root finding are required, as it avoids basis conversion and offers better stability than the Power Form.

### [Basis Transform for Trivariate Polynomials](1609-basis-transform-for-trivariate-polynomials.md)
Transforms polynomials from B-spline to Bernstein or Power basis. Chosen as a preprocessing step to enable efficient root finding or composition without storing explicit voxel descriptions.

## Operating characteristics
*   **Time Complexity:** Significantly faster than naive substitution. For degree $d=3$, the Power Form requires 234 multiplications and 360 additions, compared to 1839 multiplications and 990 additions for the naive method [S1].
*   **Memory Complexity:** Requires only 27 registers for degree $d=3$, compared to 135 registers for the naive method [S1]. This low register usage allows the method to fit into GPU execution units, enabling high parallelism [S1].
*   **Measured Performance:** In relative timing tests (where naive is 1), the Power Form achieves a relative time of $3.6 \times 10^{-3}$ for degree 3 polynomial data [S1].
*   **Accuracy:** The Mean Squared Error (MSE) is $22.6 \times 10^{-12}$ for degree 3, which is higher (worse) than the Scaled Bernstein Form ($4.78 \times 10^{-12}$) but lower than the naive method ($162.3 \times 10^{-12}$) [S1].
*   **Strengths:** Extremely efficient operation count and low register usage, making it ideal for GPU implementation where register pressure is a bottleneck [S1].
*   **Limitations:** Lower numerical stability compared to Bernstein forms, leading to higher MSE [S1]. Not suitable if root finding is required without basis conversion [S1].

## Background
The method is presented in the context of wavelet-based multiresolution rendering [S1]. It builds on the observation that power expressions can be computed incrementally, similar to the Horner scheme [S1]. The approach was developed to address the limitations of naive polynomial evaluation on GPU architectures [S1].

## Sources
[S1] Wavelet-based Multiresolution Isosurface Rendering, https://doi.org/10.2312/vg/vg10/013-020, fetched 2026-10-26
