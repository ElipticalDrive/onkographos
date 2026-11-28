# Q2 Finite Element Compression

Also called: Hierarchical Q2 Finite Element Compression, Hierarchical finite elements framework, Q2 wavelet-like compression

## Overview
Q2 Finite Element Compression is a multidimensional compression scheme that decomposes a function into coarse and detail coefficients using a hierarchical basis of quadratic polynomials (Q2), enabling sparse storage via thresholding [S1, unofficial]. Unlike standard wavelet methods that rely on separable filter banks, this method constructs a specific hierarchical basis from piecewise quadratic functions, allowing for linear complexity and efficient parallelization through local operations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A function $f$ defined on an interval $[a, b]$ (or tensor product in higher dimensions), discretized at the finest resolution level $n$ [S1, unofficial].
*   **Output:** A compressed representation $f_c$ consisting of a sparse set of coefficients $\{c_i\}$ and $\{d_i\}$ stored in a hierarchical structure [S1, unofficial].

**Data Structures:**
The method uses a hierarchical tree structure [S1, unofficial]. Each node represents an element $I^j_i$ at resolution level $j$ [S1, unofficial].
*   **Element $I^j_i$:** Represents the sub-interval $[x_i, x_{i+1}]$ at level $j$ [S1, unofficial].
*   **Coefficients per Element:** Each element stores three coefficients corresponding to the quadratic basis functions defined on that interval [S1, unofficial]. For a parent element $I^j_i$, the coefficients are denoted $c^j_{2i+l}$ for $l=0, 1, 2$ [S1, unofficial].
*   **Storage:** The compressed function $f_c$ is a set of active elements. If a detail coefficient exceeds the threshold, the parent element and its children (if applicable) are added to $f_c$ [S1, unofficial].

**Formulas:**
The approximation $\tilde{f}_h$ in the enriched vector space $\tilde{V}_h$ is given by [S1, unofficial]:
$$ \tilde{f}_h(x) = \sum_{i=0}^{2N} c_i L_i(x) + \sum_{i=0}^{2^{N-1}} d_i L_{i+1/2}(x) $$
where $c_i$ are coarse coefficients and $d_i$ are detail coefficients [S1, unofficial].

The detail coefficients $d_{2i}$ and $d_{2i+1}$ for an interval $[x_i, x_{i+1}]$ are computed as linear combinations of function values at specific points ($x_i, x_{i+1/2}, x_{i+1}, x_{i+1/4}, x_{i+3/4}$) [S1, unofficial]:
$$ d_{2i} = \frac{1}{4}f(x_{i+1/4}) - \frac{1}{8}f(x_i) - \frac{1}{8}f(x_{i+1}) + \frac{1}{4}f(x_{i+1/2}) $$
$$ d_{2i+1} = \frac{1}{4}f(x_{i+3/4}) + \frac{1}{8}f(x_i) - \frac{1}{8}f(x_{i+1}) - \frac{1}{4}f(x_{i+1/2}) $$

**Parameters:**
*   $\epsilon$: Compression threshold. Details with $|d_{j, 2i+k}| \le \epsilon$ are discarded [S1, unofficial].
*   $n$: Finest level of resolution [S1, unofficial].
*   $l$: Coarsest level of resolution to keep ($0 \le l \le n - 1$) [S1, unofficial].

**Algorithm Steps:**
1.  Initialize the compressed function $f_c$ as an empty set [S1, unofficial].
2.  Iterate through resolution levels $j$ from $n-1$ down to $l$ (inclusive) [S1, unofficial].
3.  For each level $j$, iterate through indices $i$ from $0$ to $2^j - 1$ (inclusive) [S1, unofficial].
4.  Initialize a flag `insert` to 0 [S1, unofficial].
5.  For $k$ from $0$ to $1$:
    a. Compute the detail coefficient $d_{j, 2i+k}$ using the formulas above [S1, unofficial].
    b. If $|d_{j, 2i+k}| > \epsilon$:
        i. Add the sub-element $I^{j+1}_{2i+k}$ to $f_c$ [S1, unofficial]. This involves storing the three coefficients associated with this sub-element [S1, unofficial].
        ii. Set `insert` = 1 [S1, unofficial].
6.  If `insert` is 1, add the parent element $I^j_i$ to $f_c$ [S1, unofficial]. This involves storing its three coefficients $c^j_{2i+l}$ for $l=0, 1, 2$ [S1, unofficial]. This ensures the hierarchical tree structure is maintained for reconstruction [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def q2_compression(f_values, n, l, epsilon):
    """
    Compress a function using Q2 finite element compression.
    
    Args:
        f_values: Array of function values at the finest resolution points x_0, ..., x_{2^n}.
        n: Finest resolution level.
        l: Coarsest resolution level to keep.
        epsilon: Threshold for discarding details.
        
    Returns:
        f_c: Dictionary mapping (level, index) tuples to lists of 3 coefficients.
    """
    f_c = {}
    
    for j in range(n - 1, l - 1, -1):
        num_elements = 2**j
        for i in range(num_elements):
            insert = False
            step_j = 2**(n - j)
            
            # Global indices for interval [x_i, x_{i+1}] at level j
            idx_i = i * step_j
            idx_i_plus_1 = (i + 1) * step_j
            idx_i_half = idx_i + step_j // 2
            idx_i_quarter = idx_i + step_j // 4
            idx_i_3quarter = idx_i + 3 * step_j // 4
            
            f_i = f_values[idx_i]
            f_i_half = f_values[idx_i_half]
            f_i_quarter = f_values[idx_i_quarter]
            f_i_3quarter = f_values[idx_i_3quarter]
            f_i_plus_1 = f_values[idx_i_plus_1]
            
            # Compute details d_{j, 2i} and d_{j, 2i+1}
            d_2i = 0.25 * f_i_quarter - 0.125 * f_i - 0.125 * f_i_plus_1 + 0.25 * f_i_half
            d_2i_plus_1 = 0.25 * f_i_3quarter + 0.125 * f_i - 0.125 * f_i_plus_1 - 0.25 * f_i_half
            
            for k, d_val in enumerate([d_2i, d_2i_plus_1]):
                if abs(d_val) > epsilon:
                    # Add child element I^{j+1}_{2i+k} to f_c
                    step_j1 = 2**(n - (j + 1))
                    child_idx = (2 * i + k) * step_j1
                    child_coeffs = [
                        f_values[child_idx],
                        f_values[child_idx + step_j1 // 2],
                        f_values[child_idx + step_j1]
                    ]
                    f_c[(j + 1, 2 * i + k)] = child_coeffs
                    insert = True
                    
            if insert:
                # Add parent element I^j_i to f_c
                parent_coeffs = [f_i, f_i_half, f_i_plus_1]
                f_c[(j, i)] = parent_coeffs
                
    return f_c
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The method assumes a uniform subdivision [S1, unofficial]. Boundary points must be handled correctly when computing details near $a$ and $b$ [G].
*   **Precision:** The detail coefficients involve small fractions (1/4, 1/8). Floating-point precision errors can accumulate, especially for large $n$ [G].
*   **Threshold Selection:** The choice of $\epsilon$ directly impacts compression ratio and reconstruction error. A too-small $\epsilon$ results in poor compression; a too-large $\epsilon$ results in loss of important features [G].

## Subtopics

### Triadic frequency band decomposition
Uses a triadic grid structure for decomposition. [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md)

### Laplacian Pyramid
Uses a Gaussian pyramid and detail residuals. [Laplacian Pyramid](2250-laplacian-pyramid.md)

### Wavelet Kernel Selection
Focuses on selecting optimal wavelet kernels for specific data types. [Wavelet Kernel Selection](7035-wavelet-kernel-selection.md)

## Operating characteristics
*   **Time Complexity:** Linear complexity $O(N)$ for the compression algorithm, where $N$ is the number of data points [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of active coefficients in the compressed representation. Sparse storage allows significant memory savings for smooth functions [S1, unofficial].
*   **Strengths:**
    *   Linear complexity [S1, unofficial].
    *   Efficient parallelization due to locality of operations [S1, unofficial].
    *   No separate quantization and encoding steps needed; the sparse data structure serves as the compressed format [S1, unofficial].
*   **Limitations:**
    *   Requires a hierarchical basis construction [S1, unofficial].
    *   Performance depends on the sparsity of the detail coefficients [S1, unofficial].
*   **When to use:** Suitable for large multidimensional functions where linear complexity and parallelization are critical, and the data can be well-approximated by quadratic polynomials [S1, unofficial].

## Background
The method was proposed in the paper "A dedicated Compression Scheme for Large Multidimensional Functions Visualization" [S1, unofficial]. It is based on a framework of multiresolution analysis using quadratic finite elements. The approach allows for efficient visualization of large datasets by suppressing details below a certain threshold [S1, unofficial].

## Sources
[S1] A dedicated Compression Scheme for Large Multidimensional Functions Visualization, https://inria.hal.science/inria-00591076v1, fetched 2026-10-26 (unofficial)
