# Composition via Convolution (Power Form)

Also called: Power Form

## Overview
Composition via Convolution (Power Form) is a method for evaluating the composition of a trivariate polynomial field with a linear ray parameterization, transforming the 3D function into a univariate polynomial along the ray [S1]. It solves the problem of high computational cost and register pressure associated with naive polynomial substitution, enabling efficient implementation on GPU architectures [S1]. Unlike [Composition via Scaled Bernstein Form](1592-composition-via-scaled-bernstein-form.md), which maintains a uniform data representation for better numerical stability and root-finding compatibility, the Power Form uses power bases and convolution operators to minimize operation counts and register usage, offering superior performance for direct evaluation [S1].

## How it works
The method computes the coefficients of the resulting univariate polynomial $f(t)$ by leveraging the associative property of convolution to reduce the number of multiplications and additions required.

**Inputs and Outputs:**
- Input: Trivariate polynomial coefficients $f_{ijk}$ for $i,j,k \in \{0, \dots, d\}$, where $d$ is the degree of the basis functions (typically $d=3$ for cubic).
- Input: Ray parameterization coefficients for the x, y, and z components: $\mathbf{r}_x = [p_x, v_x]$, $\mathbf{r}_y = [p_y, v_y]$, $\mathbf{r}_z = [p_z, v_z]$. These represent the linear polynomials $p_x + t v_x$, etc.
- Output: Coefficient sequence $\mathbf{f}$ of the resulting univariate polynomial of degree $3d$.

**Data Structures:**
- Coefficients are stored as floating-point arrays.
- Intermediate power expressions $\mathbf{r}_x^i$, $\mathbf{r}_y^j$, $\mathbf{r}_z^k$ are computed incrementally.

**Numbered Steps:**
1. **Initialize Power Sequences:** Compute the power expressions for the ray components incrementally using convolution. For the x-component:
   $$ \mathbf{r}_x^0 = [1] $$
   $$ \mathbf{r}_x^i = \mathbf{r}_x^{i-1} * \mathbf{r}_x \quad \text{for } i = 1, \dots, d $$
   Repeat for $\mathbf{r}_y$ and $\mathbf{r}_z$. The convolution $*$ of two coefficient sequences $A$ and $B$ results in a sequence $C$ where $C_k = \sum_{m=0}^k A_m B_{k-m}$. This step is equivalent to the Horner scheme for polynomial evaluation [S1].

2. **Compute Inner Sum ($\mathbf{f}_{ij}$):** For each pair $(i, j)$, compute the intermediate polynomial coefficients $\mathbf{f}_{ij}$ by convolving the trivariate coefficients with the z-component powers:
   $$ \mathbf{f}_{ij} = \sum_{k=0}^d f_{ijk} \cdot \mathbf{r}_z^k $$
   Here, $f_{ijk} \cdot \mathbf{r}_z^k$ denotes scalar multiplication of the coefficient sequence $\mathbf{r}_z^k$ by $f_{ijk}$.

3. **Compute Middle Sum:** For each $i$, compute the sum over $j$:
   $$ \mathbf{g}_i = \sum_{j=0}^d \mathbf{r}_y^j * \mathbf{f}_{ij} $$
   Note that $\mathbf{f}_{ij}$ is computed and consumed immediately, allowing memory reuse [S1].

4. **Compute Final Sum:** Compute the final coefficient sequence $\mathbf{f}$ by summing over $i$:
   $$ \mathbf{f} = \sum_{i=0}^d \mathbf{r}_x^i * \mathbf{g}_i $$

**Formulas:**
The core composition formula is:
$$ \mathbf{f} = \sum_{i=0}^d \mathbf{r}_x^i * \sum_{j=0}^d \mathbf{r}_y^j * \underbrace{\sum_{k=0}^d f_{ijk} \cdot \mathbf{r}_z^k}_{\mathbf{f}_{ij}} $$
Where $\mathbf{r}_x = [p_x, v_x]$ is the coefficient sequence of the x-component of the ray [S1].

**Parameters:**
- $d$: Degree of the polynomial basis. Typical value is 3 (cubic) [S1].

**Edge Cases and Pitfalls:**
- **Register Pressure:** A naive implementation requires too many operations and registers to fit into a GPU framework (e.g., 135 registers for $d=3$) [S1]. The convolution structure ensures that no more than one factor $f_{ij}$ and $f_i$ needs to be available at the same time, allowing memory reuse and reducing register usage to 27 for $d=3$ [S1].
- **Precision:** The Power Form has higher Mean Squared Error (MSE) compared to Scaled Bernstein forms, but is significantly faster [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convolution(seq_a, seq_b):
    """
    Computes the convolution of two coefficient sequences.
    seq_a, seq_b: lists of floats representing polynomial coefficients.
    Returns: list of floats representing the convolution result.
    """
    len_a = len(seq_a)
    len_b = len(seq_b)
    result_len = len_a + len_b - 1
    result = [0.0] * result_len
    
    for i in range(len_a):
        for j in range(len_b):
            result[i + j] += seq_a[i] * seq_b[j]
    return result

def scalar_multiply(coeff, seq):
    """
    Multiplies a coefficient sequence by a scalar.
    """
    return [c * coeff for c in seq]

def composition_via_convolution_power_form(f_coeffs, ray_coeffs):
    """
    Computes the composition of a trivariate polynomial with a linear ray.
    
    f_coeffs: 4D array f_ijk of shape (d+1, d+1, d+1, 1) or flattened.
              Represents coefficients f_{ijk}.
    ray_coeffs: Tuple (rx, ry, rz) where each is a list [p, v].
    
    Returns: List of coefficients for the resulting univariate polynomial.
    """
    d = 3 # Assuming cubic basis as per source example
    
    # Initialize power sequences
    # r_x^0 = [1]
    r_x = [[1.0]]
    r_y = [[1.0]]
    r_z = [[1.0]]
    
    rx_seq = ray_coeffs[0] # [px, vx]
    ry_seq = ray_coeffs[1] # [py, vy]
    rz_seq = ray_coeffs[2] # [pz, vz]
    
    # Compute powers incrementally
    for i in range(1, d + 1):
        r_x.append(convolution(r_x[i-1], rx_seq))
        r_y.append(convolution(r_y[i-1], ry_seq))
        r_z.append(convolution(r_z[i-1], rz_seq))
        
    # Compute f_ij and then g_i and finally f
    # We reuse memory by computing and consuming f_ij immediately
    
    # Initialize final result accumulator
    final_f = [0.0] * (3 * d + 1)
    
    for i in range(d + 1):
        # Compute g_i = sum_{j=0}^d r_y^j * f_ij
        # First, compute f_ij for all j, but we can process j iteratively
        # To save memory, we compute f_ij, convolve with r_y^j, and add to a temporary sum for g_i
        
        # Initialize g_i accumulator
        g_i = [0.0] * (3 * d + 1) # Max degree of f_ij is 3d? No, f_ij is sum of f_ijk * r_z^k. 
                                  # r_z^k has degree k. Max k=d. So f_ij has degree d.
                                  # Wait, f_ijk is scalar. r_z^k is poly of degree k.
                                  # So f_ij is poly of degree d.
        
        # Actually, let's look at the formula:
        # f_ij = sum_{k=0}^d f_ijk * r_z^k
        # r_z^k has degree k. So f_ij has degree d.
        
        # Then g_i = sum_{j=0}^d r_y^j * f_ij
        # r_y^j has degree j. f_ij has degree d.
        # So r_y^j * f_ij has degree j + d.
        # Max degree for g_i is d + d = 2d.
        
        # Then final_f = sum_{i=0}^d r_x^i * g_i
        # r_x^i has degree i. g_i has degree 2d.
        # Max degree for final_f is d + 2d = 3d.
        
        # Let's refine sizes
        # r_z^k size: k+1
        # f_ij size: d+1 (since max k=d)
        # r_y^j size: j+1
        # g_i size: d + j + 1? No. Convolution of size (d+1) and (j+1) is (d+j+1).
        # Max j=d, so g_i size is 2d+1.
        # r_x^i size: i+1
        # final_f size: (2d+1) + (i+1) - 1 = 2d + i + 1.
        # Max i=d, so final_f size is 3d+1.
        
        # Initialize g_i with zeros
        g_i = [0.0] * (2 * d + 1)
        
        for j in range(d + 1):
            # Compute f_ij = sum_{k=0}^d f_ijk * r_z^k
            f_ij = [0.0] * (d + 1)
            for k in range(d + 1):
                # f_ijk is a scalar
                # r_z^k is a list
                term = scalar_multiply(f_coeffs[i][j][k], r_z[k])
                # Add term to f_ij
                for m in range(len(term)):
                    f_ij[m] += term[m]
            
            # Compute r_y^j * f_ij and add to g_i
            conv_res = convolution(r_y[j], f_ij)
            for m in range(len(conv_res)):
                g_i[m] += conv_res[m]
                
        # Compute r_x^i * g_i and add to final_f
        conv_res_final = convolution(r_x[i], g_i)
        for m in range(len(conv_res_final)):
            final_f[m] += conv_res_final[m]
            
    return final_f
```

## Subtopics

### [Composition via Scaled Bernstein Form](1592-composition-via-scaled-bernstein-form.md)
Evaluates composition using scaled Bernstein basis for better numerical stability and uniform data representation, avoiding basis conversion for root finding.

### [Basis Transform for Trivariate Polynomials](1609-basis-transform-for-trivariate-polynomials.md)
Transforms trivariate polynomials from B-spline to scaled Bernstein or power basis to enable efficient root finding and compositing.

## Operating characteristics
- **Time Complexity:** The Power Form method requires significantly fewer operations than the naive approach. For $d=3$, it uses 234 multiplications and 360 additions, compared to 1839 multiplications and 990 additions for the naive method [S1].
- **Memory Complexity:** The convolution structure reduces register usage from 135 registers (naive) to 27 registers (PF), allowing it to fit into GPU register space [S1].
- **Performance:** In relative terms, the PF method is approximately 278 times faster than the naive method (relative time $3.6 \times 10^{-3}$ vs 1) [S1].
- **Accuracy:** The Mean Squared Error (MSE) for PF is $22.6 \times 10^{-12}$, which is higher than the Scaled Bernstein forms (e.g., SB2 has MSE $4.78 \times 10^{-12}$) but lower than the naive method ($162.3 \times 10^{-12}$) [S1].
- **Strengths:** High performance due to low operation count and register usage; suitable for GPU implementation.
- **Limitations:** Lower numerical stability compared to Bernstein forms; not suitable if root finding is required without basis conversion.

## Background
The method was proposed in the context of wavelet-based multiresolution isosurface rendering, where efficient evaluation of polynomial fields along rays is critical [S1]. It builds on the use of convolution operators for polynomial composition, as described in [SR03], to optimize computation for GPU frameworks [S1].

## Sources
[S1] Wavelet-based Multiresolution Isosurface Rendering, https://doi.org/10.2312/vg/vg10/013-020, fetched 2026-10-26
