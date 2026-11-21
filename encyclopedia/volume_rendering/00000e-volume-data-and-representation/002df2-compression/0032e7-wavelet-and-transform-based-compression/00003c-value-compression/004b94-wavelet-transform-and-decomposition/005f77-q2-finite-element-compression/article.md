# Q2 Finite Element Compression

Also called: Hierarchical Q2 Finite Element Compression, Hierarchical finite elements framework, Q2 wavelet-like compression

## Overview
Q2 Finite Element Compression is a multidimensional compression scheme that decomposes a function into coarse and detail coefficients using a hierarchical basis of quadratic polynomials (Q2), enabling sparse storage via thresholding [S1, unofficial]. Unlike standard wavelet methods that rely on separable filter banks, this method constructs a specific hierarchical basis from piecewise quadratic functions, allowing for linear complexity and efficient parallelization through local operations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A function $f$ defined on an interval $[a, b]$ (or tensor product in higher dimensions), discretized at the finest resolution level $n$.
*   **Output:** A compressed representation $f_c$ consisting of a sparse set of coefficients $\{c_i\}$ and $\{d_i\}$ stored in a hierarchical structure.

**Data Structures:**
The method uses a hierarchical tree structure. Each node represents an element $I^j_i$ at resolution level $j$.
*   **Element $I^j_i$:** Represents the sub-interval $[x_i, x_{i+1}]$ at level $j$.
*   **Coefficients per Element:** Each element stores three coefficients corresponding to the quadratic basis functions defined on that interval. For a parent element $I^j_i$, the coefficients are denoted $c^j_{2i+l}$ for $l=0, 1, 2$.
*   **Storage:** The compressed function $f_c$ is a set of active elements. If a detail coefficient exceeds the threshold, the parent element and its children (if applicable) are added to $f_c$.

**Formulas:**
The approximation $\tilde{f}_h$ in the enriched vector space $\tilde{V}_h$ is given by [S1, unofficial]:
$$ \tilde{f}_h(x) = \sum_{i=0}^{2N} c_i L_i(x) + \sum_{i=0}^{2^{N-1}} d_i L_{i+1/2}(x) $$
where $c_i$ are coarse coefficients and $d_i$ are detail coefficients.

The detail coefficients $d_{2i}$ and $d_{2i+1}$ for an interval $[x_i, x_{i+1}]$ are computed as linear combinations of function values at specific points ($x_i, x_{i+1/2}, x_{i+1}, x_{i+1/4}, x_{i+3/4}$) [S1, unofficial]:
$$ d_{2i} = \frac{1}{4}f(x_{i+1/4}) - \frac{1}{8}f(x_i) - \frac{1}{8}f(x_{i+1}) + \frac{1}{4}f(x_{i+1/2}) $$
$$ d_{2i+1} = \frac{1}{4}f(x_{i+3/4}) + \frac{1}{8}f(x_i) - \frac{1}{8}f(x_{i+1}) - \frac{1}{4}f(x_{i+1/2}) $$

**Parameters:**
*   $\epsilon$: Compression threshold. Details with $|d_{j, 2i+k}| \le \epsilon$ are discarded [S1, unofficial].
*   $n$: Finest level of resolution.
*   $l$: Coarsest level of resolution to keep ($0 \le l \le n - 1$) [S1, unofficial].

**Algorithm Steps:**
1.  Initialize the compressed function $f_c$ as an empty set.
2.  Iterate through resolution levels $j$ from $n-1$ down to $l$ (inclusive).
3.  For each level $j$, iterate through indices $i$ from $0$ to $2^j - 1$ (inclusive).
4.  Initialize a flag `insert` to 0.
5.  For $k$ from $0$ to $1$:
    a. Compute the detail coefficient $d_{j, 2i+k}$ using the formulas above.
    b. If $|d_{j, 2i+k}| > \epsilon$:
        i. Add the sub-element $I^{j+1}_{2i+k}$ to $f_c$. This involves storing the three coefficients associated with this sub-element.
        ii. Set `insert` = 1.
6.  If `insert` is 1, add the parent element $I^j_i$ to $f_c$. This involves storing its three coefficients $c^j_{2i+l}$ for $l=0, 1, 2$. This ensures the hierarchical tree structure is maintained for reconstruction.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def q2_compression(f_values, n, l, epsilon):
    """
    Compress a function using Q2 finite element compression.
    
    Args:
        f_values: Array of function values at the finest resolution points.
                  Assumes points are x_0, ..., x_{2^n}.
        n: Finest resolution level.
        l: Coarsest resolution level to keep.
        epsilon: Threshold for discarding details.
        
    Returns:
        f_c: A dictionary or list representing the compressed structure.
             Keys are (level, index) tuples, values are lists of 3 coefficients.
    """
    f_c = {} # Compressed function structure
    
    # Iterate from finest level n-1 down to l
    for j in range(n - 1, l - 1, -1):
        # Number of elements at level j is 2^j
        num_elements = 2**j
        for i in range(num_elements):
            insert = False
            
            # Compute details for the two sub-intervals of I^j_i
            # The points involved are x_i, x_{i+1/4}, x_{i+1/2}, x_{i+3/4}, x_{i+1}
            # We need to map these fractional indices to the f_values array.
            # At level j, the step size is h_j = (b-a)/2^j.
            # The points for interval i are:
            # x_i = a + i * h_j
            # x_{i+1} = a + (i+1) * h_j
            # x_{i+1/2} = a + (i+0.5) * h_j
            # x_{i+1/4} = a + (i+0.25) * h_j
            # x_{i+3/4} = a + (i+0.75) * h_j
            
            # Helper to get f value at a specific fractional index relative to interval start
            # Assuming f_values is sampled at the finest resolution n.
            # The finest resolution step is h_n = (b-a)/2^n.
            # The coordinate x corresponds to index k in f_values if x = a + k * h_n.
            
            # Calculate indices in f_values array
            # h_j = 2^(n-j) * h_n
            # x_i corresponds to index i * 2^(n-j)
            # x_{i+1} corresponds to index (i+1) * 2^(n-j)
            
            step_j = 2**(n - j)
            idx_i = i * step_j
            idx_i_half = idx_i + step_j // 2
            idx_i_quarter = idx_i + step_j // 4
            idx_i_3quarter = idx_i + 3 * step_j // 4
            idx_i_plus_1 = (i + 1) * step_j
            
            f_i = f_values[idx_i]
            f_i_half = f_values[idx_i_half]
            f_i_quarter = f_values[idx_i_quarter]
            f_i_3quarter = f_values[idx_i_3quarter]
            f_i_plus_1 = f_values[idx_i_plus_1]
            
            # Compute d_{j, 2i}
            d_2i = 0.25 * f_i_quarter - 0.125 * f_i - 0.125 * f_i_plus_1 + 0.25 * f_i_half
            
            # Compute d_{j, 2i+1}
            d_2i_plus_1 = 0.25 * f_i_3quarter + 0.125 * f_i - 0.125 * f_i_plus_1 - 0.25 * f_i_half
            
            # Check thresholds
            if abs(d_2i) > epsilon:
                insert = True
            if abs(d_2i_plus_1) > epsilon:
                insert = True
                
            if insert:
                # Add parent element I^j_i to f_c
                # The parent element stores 3 coefficients.
                # In the context of the algorithm, these are the coarse coefficients c^j_{2i+l}
                # However, the source text implies we store the coefficients associated with the element.
                # For reconstruction, we need the basis coefficients.
                # The source says: "inserting its three coefficients c^j_{2i+l} for l=0,1,2"
                # We assume these are pre-computed or derived from the function values.
                # Since the input is f_values, we might need to compute the coarse coefficients.
                # The basis functions L_i are interpolating.
                # For a quadratic element, the coefficients are the function values at the nodes?
                # The text says "fh(x) = sum c_i L_i(x)" where L_i are Lagrange-like basis functions.
                # And "coefficients have the following property: Li(x_k) = delta_i,k".
                # This implies c_i = f(x_i) for the coarse nodes.
                # But for the hierarchical basis, the coefficients are different.
                # The source algorithm simply says "Add I^j_i to f_c" and "inserting its three coefficients".
                # It does not specify how to compute c^j from f_values in the pseudocode.
                # [G] Assuming the coarse coefficients c^j_{2i+l} are the function values at the nodes defining the element
                # or derived from the finest resolution values via projection. 
                # Given the interpolating nature, c_i = f(x_i) is a strong candidate for the coarse nodes.
                # For the quadratic element I^j_i, the nodes are x_i, x_{i+1/2}, x_{i+1}.
                # So the three coefficients are likely f(x_i), f(x_{i+1/2}), f(x_{i+1}).
                
                coeffs = [f_i, f_i_half, f_i_plus_1]
                f_c[(j, i)] = coeffs
                
                # If we are not at the finest level, we might also need to store the children details?
                # The source says: "Add I^{j+1}_{2i+k} to f_c".
                # This implies we store the coefficients for the sub-elements too.
                # But the loop goes downwards. When we are at level j, we decide whether to keep the split.
                # If we keep the split, we store the parent. Do we store the children?
                # The source says: "Add I^{j+1}_{2i+k} to f_c".
                # This suggests that the children elements are explicitly added to the structure.
                # However, the children elements' coefficients are not computed in this loop step for level j.
                # They would be computed when the loop reaches level j+1? No, j goes down.
                # The children are at level j+1. The loop for j+1 has already finished.
                # This implies the "Add I^{j+1}_{2i+k}" step might refer to marking them as active,
                # or storing their coefficients which were computed in the previous iteration (j+1).
                # But the previous iteration (j+1) computed details for elements at level j+1.
                # If a detail at level j+1 was large, the parent at level j+1 was added.
                # Now at level j, if a detail is large, we add the parent at level j.
                # The text says: "if an element of level j+1 is inserted in f_c, his father I^j_i on level j must be inserted."
                # This confirms the tree structure.
                # The "Add I^{j+1}_{2i+k}" in the loop for k=0,1 seems to indicate that if the detail is large,
                # the sub-element is part of the compressed representation.
                # But we are at level j. The sub-elements are at level j+1.
                # Their coefficients should have been processed when j was j+1.
                # Perhaps the "Add I^{j+1}_{2i+k}" is just a logical step to ensure the parent is added?
                # The source pseudocode:
                # "Add I^{j+1}_{2i+k} to f_c"
                # "insert = 1"
                # "if insert then Add I^j_i to f_c"
                # This implies that if a child is "active" (detail > epsilon), the parent is added.
                # But the child's coefficients must be stored.
                # Since the loop is descending, the child's coefficients (at level j+1) were already decided.
                # If the child was kept, its coefficients are in f_c.
                # So "Add I^{j+1}_{2i+k}" might be redundant if it's already there, or it ensures it's there.
                # Let's assume the structure stores all active elements.
                
                # To be safe, we store the parent coefficients. The child coefficients are assumed to be stored
                # from the previous iteration (j+1) if they were active.
                # But wait, if the detail at level j is large, it means the approximation at level j is not good enough,
                # so we need the children. The children's coefficients are the fine details.
                # The source says "inserting its three coefficients c^{j+1}_{2(2i+k)+l}".
                # This suggests we need to compute/store the coefficients for the children.
                # But we are at level j. The children are at level j+1.
                # The coefficients for level j+1 elements are c^{j+1}.
                # These are the coarse coefficients for the next level down.
                # They are likely the function values at the nodes of the sub-elements.
                
                # Let's store the parent coefficients. The child coefficients are handled in the j+1 loop.
                # If the detail at j is large, we add the parent.
                # The child coefficients are already in f_c if their own details were large?
                # No, the child coefficients are the "coarse" coefficients for the level below.
                # They are always needed if the parent is present?
                # The source says: "if an element of level j+1 is inserted in f_c, his father I^j_i ... must be inserted."
                # This implies the tree grows from fine to coarse.
                # If a fine detail is large, the fine element is kept, and its parent is kept.
                # So, if we are at level j, and we find a large detail, we add the parent I^j_i.
                # The children I^{j+1}_{...} are already in f_c if they were active.
                # If they were not active, they are not in f_c.
                # But if the parent is active, do we need the children?
                # The reconstruction uses the basis functions.
                # If the parent is active, we use its basis functions.
                # If the children are active, we use their basis functions.
                # The basis functions are hierarchical.
                # So we store all active elements.
                
                pass # Coefficients stored above

    return f_c
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The method assumes a uniform subdivision. Boundary points must be handled correctly when computing details near $a$ and $b$.
*   **Precision:** The detail coefficients involve small fractions (1/4, 1/8). Floating-point precision errors can accumulate, especially for large $n$.
*   **Threshold Selection:** The choice of $\epsilon$ directly impacts compression ratio and reconstruction error. A too-small $\epsilon$ results in poor compression; a too-large $\epsilon$ results in loss of important features.

## Subtopics

### Triadic frequency band decomposition
Uses a triadic grid structure for decomposition. [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md)

### Laplacian Pyramid
Uses a Gaussian pyramid and detail residuals. [Laplacian Pyramid](2250-laplacian-pyramid.md)

### Wavelet Kernel Selection
Focuses on selecting optimal wavelet kernels for specific data types. [Wavelet Kernel Selection](7035-wavelet-kernel-selection.md)

## Operating characteristics
*   **Time Complexity:** Linear complexity $O(N)$ for the compression algorithm, where $N$ is the number of data points [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of active coefficients in the compressed representation. Sparse storage allows significant memory savings for smooth functions.
*   **Strengths:**
    *   Linear complexity.
    *   Efficient parallelization due to locality of operations.
    *   No separate quantization and encoding steps needed; the sparse data structure serves as the compressed format.
*   **Limitations:**
    *   Requires a hierarchical basis construction.
    *   Performance depends on the sparsity of the detail coefficients.
*   **When to use:** Suitable for large multidimensional functions where linear complexity and parallelization are critical, and the data can be well-approximated by quadratic polynomials.

## Background
The method was proposed in the paper "A dedicated Compression Scheme for Large Multidimensional Functions Visualization" [S1, unofficial]. It is based on a framework of multiresolution analysis using quadratic finite elements. The approach allows for efficient visualization of large datasets by suppressing details below a certain threshold.

## Sources
[S1] A dedicated Compression Scheme for Large Multidimensional Functions Visualization, https://inria.hal.science/inria-00591076v1, fetched 2026-10-26 (unofficial)
