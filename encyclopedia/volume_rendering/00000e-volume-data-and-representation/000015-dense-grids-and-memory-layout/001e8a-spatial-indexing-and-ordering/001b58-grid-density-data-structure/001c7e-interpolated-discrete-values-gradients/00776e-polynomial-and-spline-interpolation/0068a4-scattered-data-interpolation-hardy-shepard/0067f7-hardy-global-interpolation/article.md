# Hardy Global Interpolation
Also called: Hardy Coefficient Calculation
## Overview
Hardy Global Interpolation is a scattered data interpolation method that uses Hardy's multiquadric function to solve a global linear system for coefficients, then evaluates the function at volume grid points [S1, unofficial]. It differs from local methods by computing values based on all sample points simultaneously [S1, unofficial].
## How it works
**Inputs:**
- `random_Points`: List of $N$ sample points $\mathbf{x}_i \in \mathbb{R}^3$.
- `d`: List of $N$ scalar values $d_i$ corresponding to `random_Points`.
- `R`: Shape parameter for the kernel.
- `power`: Exponent for the kernel evaluation.
- `grid_points`: List of query points $\mathbf{p}$ in the volume grid.

**Outputs:**
- `ret`: Volume grid mapping each query point to its interpolated value.

**Data Structures:**
- Matrix $A$: $N \times N$ matrix where $A_{ij} = \sqrt{R + \|\mathbf{x}_i - \mathbf{x}_j\|}$ [S1, unofficial].
- Vector $b$: $N \times 1$ vector containing sample values $d$ [S1, unofficial].
- Vector $c$: $N \times 1$ vector of interpolation coefficients [S1, unofficial].
- `ret`: Output grid initialized to zeros [S1, unofficial].

**Steps:**
1. Initialize `ret` with zeros and copy sample point data to corresponding grid positions if they exist [S1, unofficial].
2. Construct matrix $A$ of size $N \times N$ with zeros [S1, unofficial].
3. For each pair of points $i, j$ from $0$ to $N-1$, calculate the Euclidean distance $\|\mathbf{x}_i - \mathbf{x}_j\|$ and set $A_{ij} = \sqrt{R + \|\mathbf{x}_i - \mathbf{x}_j\|}$ [S1, unofficial].
4. Construct vector $b$ from the sample data values $d$ [S1, unofficial].
5. Solve the linear system $\mathbf{c} = (A^T A)^{-1} A^T \mathbf{b}$ using a linear solver (e.g., `np.linalg.solve`) [S1, unofficial].
6. For each query point $\mathbf{p}$ in `grid_points`, if `ret[\mathbf{p}]` is non-zero, skip it [S1, unofficial]. Otherwise, initialize a sum to zero. For each sample point $i$ from $0$ to $N-1$, calculate $val = R + \|\mathbf{x}_i - \mathbf{p}\|^2$ and accumulate $c_i \cdot val^{\text{power}}$ to the sum [S1, unofficial]. Set `ret[\mathbf{p}]` to the accumulated sum [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import math

def hardy_global_interpolation(random_Points, d, R, power, grid_points):
    N = len(random_Points)
    ret = {p: 0.0 for p in grid_points}
    for i, p in enumerate(random_Points):
        if p in ret:
            ret[p] = d[i]
    A = np.zeros((N, N))
    for i in range(N):
        for j in range(N):
            dist = math.sqrt(sum((a - b)**2 for a, b in zip(random_Points[i], random_Points[j])))
            A[i, j] = math.sqrt(R + dist)
    b = np.array(d)
    AT = A.T
    ATA = AT @ A
    ATb = AT @ b
    c = np.linalg.solve(ATA, ATb)
    for p in grid_points:
        if ret[p] != 0.0:
            continue
        val_sum = 0.0
        for i in range(N):
            dist_sq = sum((a - b)**2 for a, b in zip(random_Points[i], p))
            val = R + dist_sq
            val_sum += c[i] * math.pow(val, power)
        ret[p] = val_sum
    return ret
```

**Parameters:**
- `R`: Shape parameter. Not given in the sources [S1, unofficial].
- `power`: Exponent for the kernel function. Not given in the sources, but `math.pow(val, power)` is used [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Overwriting Bug:** The original source code overwrites `ret[point]` in the evaluation loop instead of accumulating; the corrected pseudocode uses `+=` to fix this [S1, unofficial].
- **Singularity:** If $R \le 0$ and sample points coincide, the kernel matrix may become singular or undefined [G].
- **Precision:** Solving the normal equations $(A^T A)^{-1} A^T \mathbf{b}$ can amplify numerical errors for ill-conditioned matrices; direct solvers or regularization may be required for large $N$ [G].
## Operating characteristics
- **Time Complexity:** Constructing $A$ takes $O(N^2)$ and solving the system takes $O(N^3)$ [G]. Evaluation at $M$ grid points takes $O(M \cdot N)$ [G].
- **Memory Complexity:** $O(N^2)$ to store the kernel matrix $A$ [G].
- **Strengths:** Provides exact interpolation at sample points and smooth global continuity [G].
- **Limitations:** High computational and memory cost for large $N$; sensitive to the choice of $R$ [G].
- **When to use:** Small to medium scattered datasets where global smoothness is required [G].
- **When not to use:** Large-scale volume rendering with millions of samples due to cubic solver cost [G].
## Background
Hardy's multiquadric interpolation was introduced by Richard Hardy in geostatistics and scattered data approximation [G]. The specific implementation details cited here come from an unofficial GitHub repository [S1, unofficial].
## Sources
[S1] jarusified/WebGL-Volume-rendering, https://github.com/jarusified/WebGL-Volume-rendering, fetched 2026-10-27 (unofficial)
