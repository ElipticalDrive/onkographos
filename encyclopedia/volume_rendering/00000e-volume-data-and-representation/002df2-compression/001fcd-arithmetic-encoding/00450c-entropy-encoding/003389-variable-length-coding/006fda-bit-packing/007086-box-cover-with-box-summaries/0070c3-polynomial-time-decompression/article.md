# Polynomial Time Decompression

Also called: Decompression Algorithm
## Overview
This method reconstructs a voxelized vector field from a compressed box cover with box summaries data structure. It solves the inverse problem of the compression step by generating a reconstructed field $\hat{X}$ that satisfies the specified error bounds. It is distinguished from exact root-finding approaches by using a strict inequality constraint, which ensures the algorithm runs in polynomial time rather than being intractable or undecidable [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A box cover with box summaries data structure, defined as $\{(\hat{R}, \hat{f}_{\hat{R}}) : \hat{R} \in \mathcal{S}(X, f, \varepsilon)\}$, along with parameters $(\varepsilon^*, \varepsilon, f)$. Here, $\mathcal{S}(X, f, \varepsilon)$ is the set of boxes covering the support of the original field, $\hat{f}_{\hat{R}}$ is the summary value for box $\hat{R}$, $\varepsilon$ is the total error bound, and $\varepsilon^*$ is a strict upper bound used during compression such that $\varepsilon^* < \varepsilon$ [S1, unofficial].
*   **Output:** A reconstructed $(k, D)$-dimensional voxelized vector field $\hat{X}$, where $\hat{X}_i \in \mathbb{R}^D$ for each voxel index $i$ in the support [S1, unofficial].

**Data Structures**
The input structure consists of a list of tuples. Each tuple contains:
1.  $\hat{R}$: The geometric definition of the box (e.g., min/max coordinates).
2.  $\hat{f}_{\hat{R}}$: The summary vector or scalar value associated with the box.
The parameters $\varepsilon^*$, $\varepsilon$, and the function definition $f$ (piecewise defined as $f_j$) are global inputs.

**Steps**
1.  Initialize the output field $\hat{X}$ with dimensions matching the input support.
2.  Iterate through each box $\hat{R}$ in the cover $\mathcal{S}(X, f, \varepsilon)$:
    a.  Identify the polynomial function $f_j$ corresponding to the region of $\hat{R}$.
    b.  Formulate the polynomial inequality:
        $$ (f_j(x) - \hat{f}_{\hat{R}})^2 \leq (\varepsilon - \varepsilon^*)^2 $$
        where $x \in \bar{\mathbb{Q}}^D$ (the set of algebraic numbers in $D$ dimensions) [S1, unofficial].
    c.  If the coefficients of $f_j$ are not integers, rescale them to integers.
    d.  Run the algorithm $A$ from [GVJ88] on this inequality to compute a Representative Set $\tau$ [S1, unofficial].
    e.  For every voxel index $i$ contained within box $\hat{R}$:
        i. Select a vector $\hat{x}$ from the set $\tau$.
        ii. Assign $\hat{X}_i = \hat{x}$. The entries of $\hat{x}$ are algebraic numbers [S1, unofficial].
3.  Return $\hat{X}$.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompress(box_cover, epsilon_star, epsilon, f_functions):
    """
    Reconstructs voxelized vector field from box cover with box summaries.
    
    Args:
        box_cover: List of tuples (box_R, f_hat_R).
        epsilon_star: Strict upper bound used in compression (epsilon_star < epsilon).
        epsilon: Total allowed error bound.
        f_functions: Dictionary mapping box regions to polynomial function definitions f_j.
        
    Returns:
        X_hat: Reconstructed voxelized vector field.
    """
    X_hat = {} # Map from voxel index i to vector x_hat
    
    # Calculate the slack allowed for the inequality constraint
    slack = epsilon - epsilon_star
    
    for box_R, f_hat_R in box_cover:
        # Identify the polynomial function f_j for this box
        f_j = f_functions[box_R]
        
        # Define the inequality: (f_j(x) - f_hat_R)^2 <= (epsilon - epsilon_star)^2
        # Note: Coefficients must be rescaled to integers if not already
        inequality_poly = (f_j - f_hat_R)**2
        target_value = slack**2
        
        # Run algorithm A from [GVJ88] to find the Representative Set tau
        # This algorithm solves the polynomial inequality over algebraic numbers
        tau = algorithm_GVJ88_solve_inequality(inequality_poly, target_value)
        
        # Assign a value from tau to every voxel in the box
        for i in get_voxel_indices(box_R):
            # Select any vector from the representative set
            # In practice, a deterministic selection (e.g., first element) is required
            x_hat = tau[0] 
            X_hat[i] = x_hat
            
    return X_hat

def algorithm_GVJ88_solve_inequality(poly, target):
    """
    Placeholder for the specific algorithm from [GVJ88].
    This algorithm computes a representative set of algebraic numbers
    satisfying the polynomial inequality.
    """
    # Implementation details depend on [GVJ88]
    # It must handle rescaled integer coefficients
    pass
```

**Formulas and Correctness**
The correctness relies on the triangle inequality. For any voxel $i$ in box $\hat{R}$, the reconstruction $\hat{X}_i$ satisfies:
$$ |f(\hat{X}_i) - \hat{f}_{\hat{R}}|_2 \leq \varepsilon - \varepsilon^* $$
Since the compression step ensured $|\hat{f}_{\hat{R}} - f(X_i)|_2 \leq \varepsilon^*$, the total error is:
$$ |f(\hat{X}_i) - f(X_i)|_2 \leq |f(\hat{X}_i) - \hat{f}_{\hat{R}}|_2 + |\hat{f}_{\hat{R}} - f(X_i)|_2 \leq (\varepsilon - \varepsilon^*) + \varepsilon^* = \varepsilon $$
Thus, the reconstructed field satisfies the $(k, D)$-RectLossyVVFCompression error bound [S1, unofficial].

**Pitfalls**
*   **Intractability of Exact Roots:** If $\varepsilon^*$ were replaced by $\varepsilon$ in the decompression inequality, the constraint would become an equality requiring the exact root of a multivariate polynomial. In the Blum-Shub-Smale model, this is undecidable [S1, unofficial].
*   **Complexity of Root Finding:** Using Gröbner bases to solve for exact roots is asymptotically exponential in the number of variables [S1, unofficial]. The use of $\varepsilon^* < \varepsilon$ avoids this by allowing an inequality constraint, which is solvable in polynomial time using the method from [GVJ88] [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Polynomial time with respect to the input size, assuming the algorithm from [GVJ88] runs in polynomial time for the given inequality constraints [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of voxels in the support, as each voxel requires storing a $D$-dimensional algebraic number vector [S1, unofficial].
*   **Strengths:** Guarantees polynomial-time decompression; mathematically rigorous error bounds.
*   **Limitations:** Requires the compression step to strictly enforce $\varepsilon^* < \varepsilon$; relies on the availability and efficiency of the [GVJ88] algorithm for solving polynomial inequalities.

## Background
The method was introduced in the context of proving that $(k, D)$-RectLossyVVFCompression is NP-Hard [S1, unofficial]. It serves as the constructive inverse to the box compression algorithm, demonstrating that while finding the optimal compression is hard, decompression remains tractable if the compression step adheres to specific strict bounds [S1, unofficial]. The core computational subroutine relies on the algorithm described in [GVJ88] for handling polynomial inequalities over algebraic numbers.

## Sources
[S1] Compression of Voxelized Vector Field Data by Boxes is Hard, https://doi.org/10.48550/arxiv.2510.20801, fetched 2026-10-26 (unofficial)
