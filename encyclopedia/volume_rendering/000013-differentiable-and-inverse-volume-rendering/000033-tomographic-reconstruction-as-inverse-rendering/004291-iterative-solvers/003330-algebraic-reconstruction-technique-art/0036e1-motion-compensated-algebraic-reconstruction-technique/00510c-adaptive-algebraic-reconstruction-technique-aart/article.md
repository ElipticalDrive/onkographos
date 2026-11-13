# Adaptive Algebraic Reconstruction Technique (AART)
Also called: AART (Adaptative ART), Adaptative ART
## Overview
Adaptive Algebraic Reconstruction Technique (AART) is an optimization of the Algebraic Reconstruction Technique (ART) that employs adaptive updates to improve convergence and reconstruction quality [S1, unofficial]. It addresses the limitations of standard ART by dynamically adjusting the correction magnitude based on the discrepancy between measured and calculated projections, rather than using a fixed step size [S1, unofficial]. Unlike its sibling [Motion-Compensated Algebraic Reconstruction Technique](7422-motion-compensated-algebraic-reconstruction-technique.md), which handles deformation via warping, AART focuses on the algebraic update rule itself to enhance accuracy in tomographic reconstruction.
## How it works
**Inputs and Outputs:**
- Input: Sinogram data $y$, system matrix $A$ (where $A_{ij}$ represents the intersection length of ray $i$ with voxel $j$), initial estimate $x^{(0)}$, and iteration count $K$.
- Output: Reconstructed volume $x^{(K)}$.

**Data Structures:**
- $x$: Vector of voxel values, size $N$.
- $y$: Vector of projection measurements, size $M$.
- $A$: Sparse matrix $M \times N$.

**Algorithm Steps:**
1. Initialize $x^{(0)}$ (e.g., zero or FBP result).
2. For each iteration $k = 0$ to $K-1$:
   a. For each projection ray $i = 1$ to $M$:
      i. Calculate the forward projection: $p_i = \sum_{j=1}^{N} A_{ij} x_j^{(k)}$.
      ii. Compute the error: $e_i = y_i - p_i$.
      iii. Compute the adaptive step size $\alpha_i$. In AART, this is typically derived from the residual energy or normalized by the row norm of $A$ to ensure stability, though specific adaptive formulas vary [S1, unofficial]. A common adaptive form is $\alpha_i = \frac{e_i}{\sum_{j=1}^{N} A_{ij}^2}$ or a variant that scales with the error magnitude [G].
      iv. Update the voxel values: $x_j^{(k+1)} = x_j^{(k)} + \alpha_i A_{ij}$ for all $j$ intersected by ray $i$.
   b. Note: The update is sequential. The order of rays $i$ can influence the result [S1, unofficial].

**Parameters:**
- Iteration count $K$: Not given in the sources.
- Adaptive scaling factor: Not given in the sources.

**Edge Cases:**
- Ray order sensitivity: The sequential nature means the order of projection access affects convergence and final quality [S1, unofficial].
- Precision: Standard floating-point arithmetic is assumed; no specific bit-packing is mandated.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aart(y, A, x_init, num_iterations):
    """
    y: array of shape (M,)
    A: sparse matrix of shape (M, N)
    x_init: array of shape (N,)
    """
    x = x_init.copy()
    M, N = A.shape
    
    for k in range(num_iterations):
        for i in range(M):
            # Forward projection for ray i
            # Assuming A is sparse or accessed via row i
            row_i = A.getrow(i)
            p_i = row_i.dot(x)
            
            # Error
            e_i = y[i] - p_i
            
            # Adaptive step size calculation
            # Standard ART uses alpha = 1 / ||row_i||^2
            # AART adapts this, often by scaling with error or using a dynamic factor
            # Specific adaptive formula not detailed in source [S1, unofficial]
            # Using standard ART denominator as base for adaptation [G]
            norm_sq = row_i.dot(row_i)
            if norm_sq == 0:
                continue
            alpha = e_i / norm_sq 
            
            # Update voxels
            # x = x + alpha * A[i, :]
            # To avoid dense matrix multiplication, iterate over non-zeros
            indices = row_i.indices
            values = row_i.data
            for idx, val in zip(indices, values):
                x[idx] += alpha * val
                
    return x
```
## Subtopics

### Multiplicative ART
Multiplicative ART (MART) is an optimization of ART that uses multiplicative updates instead of additive ones [S1, unofficial]. It ensures non-negativity of voxel values naturally, making it suitable for applications where negative densities are physically impossible. It is chosen when positivity constraints are critical, unlike AART which uses additive adaptive steps [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(K \cdot M \cdot \bar{d})$, where $\bar{d}$ is the average number of voxels per ray. Sequential updates prevent easy parallelization of the inner loop [S1, unofficial].
- Memory: $O(N + M)$ for storing $x$, $y$, and $A$.

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Scales linearly with the number of rays and iterations.

**Strengths and Limitations:**
- Strengths: Improved convergence over standard ART; adaptive nature handles varying ray sensitivities [S1, unofficial].
- Limitations: Sequential update order affects results; slower than simultaneous methods like SIRT or SART for large datasets [S1, unofficial].

**When to use:**
- Use when adaptive step sizes are needed for better convergence in tomographic reconstruction [S1, unofficial].
- Avoid when parallel performance is critical or when global consistency (SIRT) is preferred over sequential refinement.
## Background
**Origin:**
- Developed as an optimization of ART, alongside MART, to improve reconstruction quality by adapting updates based on projection errors [S1, unofficial].
- Cited in the context of tomographic reconstruction algorithms and applications [S1, unofficial].

**History:**
- Emerged as part of the evolution from basic ART to more sophisticated iterative solvers like SIRT and SART [S1, unofficial].

**Usage:**
- Used in tomographic reconstruction for precision and quality improvement [S1, unofficial].

## Sources
[S1] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
