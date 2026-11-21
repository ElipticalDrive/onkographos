# Adaptive Algebraic Reconstruction Technique (AART)
Also called: AART (Adaptative ART), Adaptative ART
## Overview
Adaptive Algebraic Reconstruction Technique (AART) is an optimization of the Algebraic Reconstruction Technique (ART) that employs adaptive updates to improve reconstruction quality [S1, unofficial]. It corrects pixel values based on the error measured between the initial sinogram and the sinogram recalculated from the image obtained at the previous iteration [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Sinogram data $y$, system matrix $A$ (where $A_{ij}$ represents the intersection length of ray $i$ with voxel $j$), initial estimate $x^{(0)}$, and iteration count $K$. [G]
- Output: Reconstructed volume $x^{(K)}$. [G]

**Data Structures:**
- $x$: Vector of voxel values, size $N$. [G]
- $y$: Vector of projection measurements, size $M$. [G]
- $A$: Sparse matrix $M \times N$. [G]

**Algorithm Steps:**
1. Initialize $x^{(0)}$ (e.g., zero or FBP result) [G].
2. For each iteration $k = 0$ to $K-1$:
   a. For each projection ray $i = 1$ to $M$:
      i. Calculate the forward projection: $p_i = \sum_{j=1}^{N} A_{ij} x_j^{(k)}$ [G].
      ii. Compute the error: $e_i = y_i - p_i$ [G].
      iii. Compute the adaptive step size $\alpha_i$. The sources do not specify the exact adaptive formula [S1, unofficial]. A standard implementation uses $\alpha_i = \frac{e_i}{\sum_{j=1}^{N} A_{ij}^2}$ [G].
      iv. Update the voxel values: $x_j^{(k+1)} = x_j^{(k)} + \alpha_i A_{ij}$ for all $j$ intersected by ray $i$ [G].
   b. The update is sequential. The order of rays $i$ can influence the result [S1, unofficial].

**Parameters:**
- Iteration count $K$: Not given in the sources. [S1, unofficial]
- Adaptive scaling factor: Not given in the sources. [S1, unofficial]

**Edge Cases:**
- Ray order sensitivity: The sequential nature means the order of projection access affects convergence and final quality [S1, unofficial].
- Precision: Standard floating-point arithmetic is assumed; no specific bit-packing is mandated. [G]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aart(y, A, x_init, num_iterations):
    x = x_init.copy()
    M, N = A.shape
    for k in range(num_iterations):
        for i in range(M):
            row_i = A.getrow(i)
            # Forward projection for ray i
            p_i = sum(val * x[idx] for idx, val in zip(row_i.indices, row_i.data))
            # Error between measured and recalculated sinogram
            e_i = y[i] - p_i
            # Row norm squared
            norm_sq = sum(val * val for val in row_i.data)
            if norm_sq == 0:
                continue
            # Adaptive step size (formula not specified in sources [S1, unofficial])
            alpha = e_i / norm_sq
            # Sequential voxel update
            for idx, val in zip(row_i.indices, row_i.data):
                x[idx] += alpha * val
    return x
```
## Subtopics

### [Multiplicative ART](14391-multiplicative-art.md)
Multiplicative ART (MART) is an optimization of ART that uses multiplicative updates to solve the linear system $P = Af$. It addresses the non-negativity constraint inherent in many volume rendering and tomographic problems by ensuring reconstructed voxel values remain positive through multiplicative correction factors rather than additive adjustments. Unlike its sibling [Adaptive Algebraic Reconstruction Technique (AART)](14387-adaptive-algebraic-reconstruction-technique-aart.md), which uses adaptive step sizes, MART relies on a fixed multiplicative model derived from maximum likelihood estimation principles.

## Operating characteristics
**Time and Memory Complexity:**
- Not given in the sources. [S1, unofficial]

**Measured Performance:**
- Not given in the sources. [S1, unofficial]

**Scaling:**
- Not given in the sources. [S1, unofficial]

**Strengths and Limitations:**
- Strengths: Adaptive updates adjust corrections based on projection errors [S1, unofficial].
- Limitations: Sequential update order affects results [S1, unofficial].

**When to use:**
- Use when adaptive updates are needed for tomographic reconstruction [S1, unofficial].
- Avoid when parallel performance is critical or when global consistency is preferred over sequential refinement [G].
## Background
**Origin:**
- Developed as an optimization of ART to improve reconstruction quality by adapting updates based on projection errors [S1, unofficial].

**History:**
- Emerged alongside MART as part of the evolution from basic ART to more sophisticated iterative solvers like SIRT and SART [S1, unofficial].

**Usage:**
- Used in tomographic reconstruction for precision and quality improvement [S1, unofficial].
## Sources
[S1] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
