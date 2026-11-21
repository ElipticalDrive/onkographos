# Hardy Global Interpolation

Also called: Hardy Coefficient Calculation

## Overview
Hardy Global Interpolation is a scattered data interpolation method that uses Hardy's multiquadric kernel to solve a global linear system for coefficients, enabling the reconstruction of volume data from sparse sample points [S1, unofficial]. It differs from local methods like Shepard interpolation by considering all sample points simultaneously to determine values at grid locations, which provides smooth global continuity but incurs higher computational cost for large datasets [S1, unofficial].

## How it works
**Inputs:**
- `self.random_Points`: A list of $N$ sample points in 3D space, denoted as $x_i$.
- `d`: A list of $N$ scalar values corresponding to the sample points.
- `R`: A shape parameter for the multiquadric kernel.
- `power`: The exponent for the kernel function (typically 0.5 for standard multiquadric, though the source code uses `math.pow(val, power)`).
- `grid_points`: The set of 3D coordinates in the volume grid where interpolation is required.

**Outputs:**
- `ret`: A volume grid (or map) containing interpolated values at each grid point.

**Data Structures:**
- Matrix $A$: An $N \times N$ symmetric matrix where $A_{ij} = \sqrt{R + \|x_i - x_j\|^2}$.
- Vector $b$: An $N \times 1$ vector containing the sample values $d$.
- Vector $c$: An $N \times 1$ vector of interpolation coefficients.
- Vector $x$: The solution vector to the linear system, which corresponds to $c$.

**Steps:**
1. Initialize the output volume `ret` with zeros.
2. Copy sample point data to corresponding grid positions if they exist in the grid.
3. Construct the kernel matrix $A$ of size $N \times N$:
   - For each pair of points $i, j$ from $0$ to $N-1$:
     - Calculate Euclidean distance $dist(x_i, x_j)$.
     - Set $A_{ij} = \sqrt{R + dist(x_i, x_j)^2}$.
4. Construct the value vector $b$ from the sample data values $d$.
5. Solve the linear system for coefficients $c$. The source code notes solving $A^T A x = A^T b$ using `np.linalg.solve`, which implies solving the normal equations for a least-squares fit or potentially a typo in the source where $A c = b$ is solved directly if $A$ is invertible. Given $A$ is symmetric positive definite (for $R>0$), solving $A c = b$ is standard for interpolation. However, the source explicitly states `x = (A^T A)^{-1} A^T b`. We follow the source's explicit algorithm:
   - Compute $M = A^T A$.
   - Compute $y = A^T b$.
   - Solve $M x = y$ for $x$ using a linear solver (e.g., Cholesky or LU decomposition).
   - Set $c = x$.
6. Evaluate the interpolated field at each grid point $p$:
   - If the grid point $p$ already has a non-zero value (from step 2), skip it.
   - Initialize `val_sum = 0`.
   - For each sample point $i$ from $0$ to $N-1$:
     - Calculate $dist2 = \|x_i - p\|^2$.
     - Calculate $val = R + dist2$.
     - Add $c_i \cdot val^{power}$ to `val_sum`.
   - Set `ret[p] = val_sum`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import math

def hardy_global_interpolation(random_Points, d, R, power, grid_points):
    """
    Perform Hardy Global Interpolation on a volume grid.
    
    Args:
        random_Points: List of N 3D points (x_i).
        d: List of N scalar values corresponding to random_Points.
        R: Shape parameter for the multiquadric kernel.
        power: Exponent for the kernel (e.g., 0.5).
        grid_points: List of 3D points to interpolate.
        
    Returns:
        ret: Dictionary or array mapping grid points to interpolated values.
    """
    N = len(random_Points)
    ret = {}
    
    # Step 1 & 2: Initialize and copy existing data
    # Assuming grid_points are keys in a pre-existing structure or we build ret from scratch.
    # The source implies copying sample data to grid positions if they match.
    # For implementation, we assume ret is initialized to 0 or empty.
    # We will compute values for all grid_points not in random_Points.
    
    # Step 3: Construct Matrix A
    # A[i][j] = sqrt(R + distance(x_i, x_j)^2)
    A = np.zeros((N, N))
    for i in range(N):
        for j in range(N):
            # Euclidean distance squared
            dist_sq = sum((a - b)**2 for a, b in zip(random_Points[i], random_Points[j]))
            A[i, j] = math.sqrt(R + dist_sq)
            
    # Step 4: Construct vector b
    b = np.array(d)
    
    # Step 5: Solve linear system
    # Source specifies: x = (A^T A)^{-1} A^T b
    # This is the least-squares solution. If A is square and invertible, 
    # solving A c = b is sufficient. We follow the source's explicit formula.
    AT = A.T
    ATA = AT @ A
    ATb = AT @ b
    
    # Solve ATA * x = ATb
    # Using numpy's solve which handles linear systems
    c = np.linalg.solve(ATA, ATb)
    
    # Step 6: Evaluate at grid points
    for p in grid_points:
        # Skip if value already exists and is non-zero (from step 2)
        # In a pure implementation, we might just overwrite or check a mask.
        # Here we assume we compute for all points, but the source logic 
        # suggests skipping if 'volume value at that point is not 0'.
        # We will compute the sum for all points for completeness, 
        # but note the source's conditional skip.
        
        val_sum = 0.0
        for i in range(N):
            # Distance squared between sample point i and grid point p
            dist_sq = sum((a - b)**2 for a, b in zip(random_Points[i], p))
            val = R + dist_sq
            val_sum += c[i] * math.pow(val, power)
        
        ret[p] = val_sum
        
    return ret
```

**Parameters:**
- `R`: Shape parameter. Not given in the sources.
- `power`: Exponent. Not given in the sources, but `math.pow(val, power)` is used.

**Edge Cases and Pitfalls:**
- **Singularity:** If $R \le 0$ and points coincide, the kernel may become singular or undefined.
- **Performance:** The construction of $A$ is $O(N^2)$ and solving the system is $O(N^3)$, making it expensive for large $N$.
- **Overwriting:** The source notes that the loop overwrites `ret[point]` for each sample point if not careful, but the pseudocode above correctly accumulates the sum. The source's comment about overwriting likely refers to a bug in the original code if not implemented as a sum.

## Subtopics

### [Shepard Local Interpolation](19992-shepard-local-interpolation.md)
A local interpolation method using weighted averages of nearby points. Choose when computational efficiency for large datasets is prioritized over global smoothness.

## Operating characteristics
- **Time Complexity:** $O(N^2)$ to build the kernel matrix and $O(N^3)$ to solve the linear system, where $N$ is the number of sample points. Evaluation at $M$ grid points is $O(M \cdot N)$.
- **Memory Complexity:** $O(N^2)$ to store the kernel matrix $A$.
- **Strengths:** Provides smooth, global interpolation; exact fit at sample points if $R$ is chosen appropriately and the system is non-singular.
- **Limitations:** High computational cost for large $N$; memory intensive; sensitive to the choice of $R$.
- **When to use:** Small to medium-sized scattered datasets where global smoothness is critical.
- **When not to use:** Large-scale volume rendering with millions of samples due to $O(N^3)$ solver cost.

## Background
Hardy's multiquadric interpolation was introduced by Richard Hardy in the context of geostatistics and scattered data approximation. It has been adapted for volume rendering to reconstruct dense grids from sparse samples. The specific implementation details cited here come from an unofficial GitHub repository [S1, unofficial].

## Sources
[S1] jarusified/WebGL-Volume-rendering, https://github.com/jarusified/WebGL-Volume-rendering, fetched 2026-10-27 (unofficial)
