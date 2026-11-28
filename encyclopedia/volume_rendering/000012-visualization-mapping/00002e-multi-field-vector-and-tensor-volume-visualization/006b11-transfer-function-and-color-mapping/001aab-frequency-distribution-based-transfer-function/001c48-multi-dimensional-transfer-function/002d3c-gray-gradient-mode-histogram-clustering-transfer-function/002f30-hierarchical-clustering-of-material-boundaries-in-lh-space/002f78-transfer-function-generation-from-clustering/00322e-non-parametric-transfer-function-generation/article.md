# Non-Parametric Transfer Function Generation

Also called: Variable Kernel Density Estimation for TF Clustering

## Overview
Non-Parametric Transfer Function Generation is a clustering method for 2D histograms (conventional or abstracted) that groups data based on neighborhood information using variable kernel density estimation [S1, unofficial]. It provides users with a means to group data in the 2D histogram by areas of similar value [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N$ data points $\{X_1, X_2, ..., X_N\}$ representing voxels in a 2D feature space (e.g., Gray-Gradient Mode Histogram) [S1, unofficial].
*   **Output:** A probability density estimate $\hat{f}_h(x)$ over the feature space, which identifies clusters as local maxima in the density distribution [S1, unofficial].

**Data Structures**
*   **Samples:** An array of $N$ points, where each point $X_i$ is a vector in the 2D feature space [S1, unofficial].
*   **Parameters:**
    *   $N$: Total number of samples (voxels) [S1, unofficial].
    *   $k$: Index of the nearest neighbor used to scale the kernel radius [S1, unofficial].
    *   $d_{i,k}$: Multi-dimensional smoothing parameter for point $X_i$ [S1, unofficial].

**Steps**
1.  **Calculate Smoothing Parameters:** For each sample $X_i$, compute $d_{i,k}$, the distance from $X_i$ to its $k$-th nearest neighbor in the set of $N-1$ other data points [S1, unofficial].
2.  **Select Kernel:** Use the Epanechnikov kernel $K(u)$ for computational efficiency [S1, unofficial].
3.  **Compute Density Estimate:** Calculate the probability density $\hat{f}_h(x)$ at any location $x$ in the feature space using the variable kernel formula [S1, unofficial].
4.  **Identify Clusters:** Locate local maxima in the resulting density surface $\hat{f}_h(x)$ [S1, unofficial]. Each local maximum represents a potential volumetric feature cluster [S1, unofficial].

**Formulas**
The probability density estimate $\hat{f}_h(x)$ at location $x$ is:
$$ \hat{f}_h(x) = \frac{1}{N} \sum_{i=1}^{N} \frac{1}{d_{i,k}} K\left(\frac{x - X_i}{d_{i,k}}\right) $$

Where:
*   $N$ is the total number of samples in the histogram feature space [S1, unofficial].
*   $d_{i,k}$ is the multi-dimensional smoothing parameter, defined as the distance from the $i$-th sample $X_i$ to its $k$-th nearest neighbor in the set of $N-1$ data points [S1, unofficial].
*   $K(u)$ is the kernel function. The Epanechnikov kernel is used to reduce calculation time [S1, unofficial]:
    $$ K(u) = \frac{3}{4}(1 - \|u\|^2) \quad \text{for } \|u\| \le 1, \text{ and } 0 \text{ otherwise} $$

**Parameters**
*   $k$: The index of the nearest neighbor. Set to $k = \lfloor \sqrt{N} \rfloor$ as a rule of thumb approximation for optimal density estimation fitting [S1, unofficial].

**Pitfalls**
*   **Sparse Regions:** Data points in sparse regions require flatter kernels to maintain density estimation accuracy [S1, unofficial]. This is handled by scaling the kernel width proportional to $d_{i,k}$ (distance to $k$-th nearest neighbor) [S1, unofficial].
*   **Boundary Effects:** The method assumes the feature space is bounded; points near the edge may have underestimated density due to truncated kernel support [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def epanechnikov_kernel(u_sq_norm):
    """Epanechnikov kernel K(u) = 3/4 * (1 - ||u||^2) if ||u|| <= 1, else 0."""
    if u_sq_norm <= 1.0:
        return 0.75 * (1.0 - u_sq_norm)
    return 0.0

def compute_d_ik(samples, k):
    """Compute d_i,k for all samples: distance to k-th nearest neighbor."""
    N = len(samples)
    d_ik = []
    for i in range(N):
        dists = []
        for j in range(N):
            if i == j:
                continue
            dx = samples[i][0] - samples[j][0]
            dy = samples[i][1] - samples[j][1]
            dists.append(math.sqrt(dx*dx + dy*dy))
        dists.sort()
        # k is 1-indexed in the source; list is 0-indexed
        if k <= len(dists):
            d_ik.append(dists[k-1])
        else:
            d_ik.append(dists[-1])
    return d_ik

def compute_density_at_point(samples, d_ik, query_point):
    """Compute f_hat(x) at a single query point x."""
    N = len(samples)
    density = 0.0
    for i in range(N):
        d = d_ik[i]
        if d == 0.0:
            continue
        dx = query_point[0] - samples[i][0]
        dy = query_point[1] - samples[i][1]
        u_sq = (dx/d)**2 + (dy/d)**2
        density += (1.0 / d) * epanechnikov_kernel(u_sq)
    return density / N

def generate_density_grid(samples, k, resolution=100):
    """Evaluate density on a grid and return the grid and d_ik values."""
    d_ik = compute_d_ik(samples, k)
    min_vals = [min(p[dim] for p in samples) for dim in range(2)]
    max_vals = [max(p[dim] for p in samples) for dim in range(2)]
    
    grid = []
    for i in range(resolution):
        row = []
        for j in range(resolution):
            x = min_vals[0] + i * (max_vals[0] - min_vals[0]) / (resolution - 1)
            y = min_vals[1] + j * (max_vals[1] - min_vals[1]) / (resolution - 1)
            row.append(compute_density_at_point(samples, d_ik, [x, y]))
        grid.append(row)
    return grid, d_ik
```

## Subtopics

### [Topology-based Non-Parametric Clustering](7417-topology-based-non-parametric-clustering.md)
Topology-based Non-Parametric Clustering decomposes a 2D density plot feature space into non-overlapping valley cells separated by valley lines, where each cell contains a local maximum representing a potential volumetric feature. It differs from discrete histogram-based clustering by operating on continuous density plots, thereby avoiding spurious peaks/valleys and preserving the spatial relationships of neighboring data points. Unlike frequency binning methods that consider only density magnitude, this method incorporates position in feature space via topological information, preventing disjoint segments for the same density value.

## Operating characteristics
Not given in the sources.

## Background
The method originated from the work of Maciejewski et al. [12] in the context of abstract feature space representation for volumetric transfer function exploration [S1, unofficial]. It builds on variable kernel density estimation techniques [17, 19] cited in [S1, unofficial].

## Sources
[S1] Abstract Feature Space Representation for Volumetric Transfer Function Exploration, https://doi.org/10.4230/dfu.vol2.sciviz.2011.212, fetched 2026-10-26 (unofficial)
