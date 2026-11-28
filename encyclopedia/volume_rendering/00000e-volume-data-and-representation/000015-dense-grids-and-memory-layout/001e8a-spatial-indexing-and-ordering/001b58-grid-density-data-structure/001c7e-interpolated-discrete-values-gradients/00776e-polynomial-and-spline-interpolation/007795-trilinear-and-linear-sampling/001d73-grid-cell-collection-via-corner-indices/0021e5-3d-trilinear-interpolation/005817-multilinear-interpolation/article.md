# Multilinear Interpolation

Also called: Higher-Dimensional Trilinear Interpolation, Tensor Product Linear Interpolation, Tensor-product interpolation, n-dimensional linear interpolation

## Overview
Multilinear interpolation generalizes trilinear interpolation to $n$ dimensions, estimating function values within an $n$-dimensional hypercube defined by $2^n$ grid points at its vertices [S1]. It constructs the approximation via the tensor product of univariate linear interpolations across each dimension [S1]. Unlike [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md), which is limited to three spatial dimensions, this method applies the same weighted-sum logic to arbitrary dimensions, though it suffers from exponential complexity growth [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{x} = (x_1, \dots, x_n)$ located within a unit hypercube (where each coordinate $x_i \in [0, 1]$); a set of function values $f(\sigma)$ at the $2^n$ vertices $\sigma$ of the hypercube, where $\sigma$ is a binary index vector $(\sigma_1, \dots, \sigma_n)$ with $\sigma_i \in \{0, 1\}$.
*   **Output:** The interpolated scalar value $f(\mathbf{x})$.

**Data Structures:**
The method requires access to the $2^n$ vertex values. In implementation, these are typically stored in a flat array or accessed via bitwise indexing. For a vertex index $\sigma$, the value is retrieved as $f(\sigma_1, \dots, \sigma_n)$.

**Algorithm:**
The value is computed as a weighted sum of the vertex values. For each of the $2^n$ vertices, a weight is calculated as the product of linear weights in each dimension.

1.  Initialize `result` to 0.
2.  Iterate through all binary combinations $\sigma = (\sigma_1, \dots, \sigma_n)$ where $\sigma_i \in \{0, 1\}$. There are $2^n$ such combinations.
3.  For each combination $\sigma$:
    a. Calculate the vertex weight $W_\sigma = \prod_{i=1}^n w_i(\sigma_i)$.
    b. The linear weight $w_i(\sigma_i)$ for dimension $i$ is defined as:
       $$ w_i(0) = 1 - x_i $$
       $$ w_i(1) = x_i $$
    c. Retrieve the function value $f(\sigma)$ at the vertex defined by indices $\sigma$.
    d. Add $f(\sigma) \times W_\sigma$ to `result`.
4.  Return `result`.

**Formula:**
The general formulation for multilinear interpolation of a function $f(\mathbf{x})$ is:

$$ f(\mathbf{x}) = \sum_{\sigma \in \{0,1\}^n} f(\sigma) \prod_{i=1}^n w_i(\sigma_i) $$

Where:
*   $\mathbf{x} = (x_1, \dots, x_n)$ is the query point in the unit hypercube.
*   $\sigma = (\sigma_1, \dots, \sigma_n)$ iterates over all binary indices $\{0,1\}^n$.
*   $f(\sigma)$ is the function value at the vertex $\sigma$.
*   $w_i(\sigma_i)$ is the linear weight in dimension $i$: $w_i(0) = 1 - x_i$ and $w_i(1) = x_i$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multilinear_interpolation(x, vertex_values):
    """
    Performs multilinear interpolation for a point x in an n-dimensional unit hypercube.
    
    Args:
        x: List or array of n floats, where each x[i] is in [0, 1].
        vertex_values: List or array of 2^n floats, indexed by the integer representation 
                       of the binary vertex indices. 
                       Index k corresponds to binary representation of k.
    
    Returns:
        The interpolated scalar value.
    """
    n = len(x)
    num_vertices = 1 << n  # 2^n
    result = 0.0
    
    for sigma_int in range(num_vertices):
        # Calculate the weight for this vertex
        vertex_weight = 1.0
        
        # Iterate over each dimension to compute the product of linear weights
        for i in range(n):
            # Extract the i-th bit of sigma_int to determine sigma_i
            sigma_i = (sigma_int >> i) & 1
            
            if sigma_i == 0:
                w_i = 1.0 - x[i]
            else:
                w_i = x[i]
                
            vertex_weight *= w_i
            
        # Add the weighted contribution of this vertex
        result += vertex_values[sigma_int] * vertex_weight
        
    return result
```

**Edge Cases and Pitfalls:**
*   **Curse of Dimensionality:** Storage and computation scale as $2^n$. For $n > 5$ or $6$, full grid evaluation becomes prohibitive [S1].
*   **Boundary Conditions:** The formula assumes the query point is strictly within the unit hypercube $[0,1]^n$. If $x_i$ falls outside this range, standard multilinear interpolation does not define behavior; implementations like SciPy's `interpn` default to linear extrapolation [S1].
*   **Precision:** The tensor-product structure ensures exact reproduction of linear functions but can accumulate floating-point errors in very high dimensions due to the large number of multiplications [S1].

**Subtopics:**
*   [Sparse Grid Multilinear Interpolation](link): Uses hierarchical bases to reduce the number of required grid points from exponential to near-polynomial scaling, mitigating the curse of dimensionality for smooth functions [S1].
*   [Tensor Product Approximation](link): A broader class of methods using tensor products of basis functions; multilinear interpolation is the specific case using linear basis functions [S1].

## Operating characteristics
*   **Time Complexity:** $O(2^n)$ per query point, as it requires iterating over all $2^n$ vertices of the hypercube [S1].
*   **Memory Complexity:** $O(2^n)$ to store the vertex values for a single hypercube [S1].
*   **Strengths:** Exact reproduction of linear functions; simple tensor-product structure; generalizable to any dimension $n$ [S1].
*   **Limitations:** Exponential scaling makes it impractical for $n \gtrsim 6$ on dense grids [S1].
*   **When to use:** Low-dimensional interpolation ($n \le 4$); applications where linear accuracy is required and dimensions are manageable [S1].
*   **When not to use:** High-dimensional spaces ($n > 6$) without sparsification techniques; real-time applications with strict latency constraints in high dimensions [S1].

## Background
Multilinear interpolation is a generalization of trilinear interpolation, extending the tensor-product approach to $n$ dimensions [S1]. It is rooted in finite element methods and structural analysis, where similar basis functions (e.g., $N_i(\xi,\eta,\zeta) = \frac{1}{8}(1 \pm \xi)(1 \pm \eta)(1 \pm \zeta)$) are used to map distorted geometries to reference cubes [S1]. In modern computing, it is used in gravitational waveform modeling from numerical relativity simulations for constructing reduced-order models across black hole merger parameter spaces [S1]. It is also employed in machine learning for value function estimation in reinforcement learning on discretized state-action spaces [S1]. Standard libraries like SciPy provide implementations via `interpn` for regular or rectilinear grids [S1].

## Sources
[S1] trilinear interpolation, https://grokipedia.com/page/Trilinear_interpolation, fetched 2026-10-26
