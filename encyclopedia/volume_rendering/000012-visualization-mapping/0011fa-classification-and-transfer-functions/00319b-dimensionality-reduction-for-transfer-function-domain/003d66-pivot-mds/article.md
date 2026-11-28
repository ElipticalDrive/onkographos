# Pivot MDS
Also called: FastMap, FastMap Dimensionality Reduction, Landmarks MDS, Pivot Multidimensional Scaling

## Overview
Pivot MDS (also known as FastMap) is a linear-time variant of Multidimensional Scaling (MDS) used to project high-dimensional voxel data into a low-dimensional space (typically 2D or 3D) while preserving relative distances between points [S1, unofficial]. It solves the computational bottleneck of classical MDS, which has $O(n^3)$ time complexity, by using a heuristic to select only two pivot points per dimension and projecting all other points via interpolation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A dataset $P$ of $n$ instances (voxels), where each instance has a feature vector of dimension $d$, and a target dimension $k$ (typically $k < d$) [S1, unofficial].
- **Output:** A set of projected points in a $k$-dimensional space.

**Data Structures:**
- $P$: List of $n$ points, where each point is a vector of size $d$.
- $O_a, O_b$: Indices of the two pivot points selected from $P$.
- $X$: Matrix of size $n \times k$ storing projected coordinates.
- $f(a, b)$: Distance function between points $a$ and $b$.

**Numbered Steps:**
1. **Pivot Selection:** Select two pivots $O_a$ and $O_b$ from the dataset $P$ that are approximately furthest apart using a linear-time heuristic [S1, unofficial].
2. **Projection:** For each point $p \in P$, compute its coordinate $x$ along the axis defined by the segment $O_a O_b$ using the Law of Cosines [S1, unofficial].
3. **Distance Update:** For subsequent dimensions ($i \ge 2$), adjust the distance metric to remove the variance explained by previously projected dimensions [S1, unofficial].
4. **Recursion:** Repeat steps 1–3 for $k$ dimensions [S1, unofficial].

**Formulas:**
The projected coordinate $x$ for a point $p$ given pivots $O_a$ and $O_b$ is calculated as [S1, unofficial]:

$$x = \frac{f(O_a, p)^2 + f(O_a, O_b)^2 - f(O_b, p)^2}{2 f(O_a, O_b)}$$

where:
- $x$ is the projected coordinate.
- $p$ is the original point.
- $O_a$ and $O_b$ are the two pivot points.
- $f(a, b)$ is the distance function (e.g., Euclidean distance) [S1, unofficial].

For dimensions $i \ge 2$, the distance between two points $p_a$ and $p_b$ is adjusted to remove the variance explained by previous projections [S1, unofficial]:

$$f(p_a, p_b) = \sqrt{\text{dist}_d(p_a, p_b)^2 - \text{dist}_{i-1}(\text{proj}_a, \text{proj}_b)^2}$$

where:
- $\text{dist}_d$ is the metric distance operator in the original $d$-dimensional space.
- $\text{dist}_{i-1}$ is the distance between the projections of $p_a$ and $p_b$ in the first $i-1$ dimensions.
- $\text{proj}_a, \text{proj}_b$ are the projected coordinates of $p_a$ and $p_b$ in the previous dimensions [S1, unofficial].

**Pivot Selection Heuristic (Algorithm 2):**
To avoid the quadratic cost of finding the true furthest pair, the following heuristic is used [S1, unofficial]:
1. Select an arbitrary point $p \in P$ as temporary pivot $O_a$.
2. Select point $p \in P$ furthest from $O_a$ as pivot $O_b$.
3. Select point $p \in P$ furthest from $O_b$ as final pivot $O_a$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pivot_mds(points, target_dim):
    """
    Projects points into target_dim dimensions using Pivot MDS (FastMap).
    
    Args:
        points: List of vectors (n x d).
        target_dim: Integer k, the number of output dimensions.
        
    Returns:
        projected: List of vectors (n x k).
    """
    n = len(points)
    d = len(points[0])
    projected = [[0.0] * target_dim for _ in range(n)]
    
    def residual_dist(idx_p, idx_q, current_dim):
        """Computes distance in the residual space after removing previous projections."""
        orig_dist_sq = sum((points[idx_p][j] - points[idx_q][j])**2 for j in range(d))
        proj_dist_sq = sum((projected[idx_p][j] - projected[idx_q][j])**2 for j in range(current_dim))
        val = orig_dist_sq - proj_dist_sq
        return val**0.5 if val > 0 else 0.0

    for dim in range(target_dim):
        # Step 1: Pivot Selection Heuristic
        idx_Oa = 0
        max_dist = -1.0
        idx_Ob = -1
        for i in range(n):
            dist = residual_dist(idx_Oa, i, dim)
            if dist > max_dist:
                max_dist = dist
                idx_Ob = i
                
        idx_Oa_new = -1
        max_dist = -1.0
        for i in range(n):
            dist = residual_dist(idx_Ob, i, dim)
            if dist > max_dist:
                max_dist = dist
                idx_Oa_new = i
        idx_Oa = idx_Oa_new
        
        dist_Oa_Ob = residual_dist(idx_Oa, idx_Ob, dim)
        
        # Step 2: Projection
        if dist_Oa_Ob == 0:
            for i in range(n):
                projected[i][dim] = 0.0
            continue
            
        for i in range(n):
            dist_Oa_p = residual_dist(idx_Oa, i, dim)
            dist_Ob_p = residual_dist(idx_Ob, i, dim)
            x = (dist_Oa_p**2 + dist_Oa_Ob**2 - dist_Ob_p**2) / (2 * dist_Oa_Ob)
            projected[i][dim] = x

    return projected
```

**Parameters:**
- $n$: Total number of voxels (instances) [S1, unofficial].
- $d$: Original dimension of the domain [S1, unofficial].
- $k$: Target dimension (typically 2 or 3) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Pivot Distance:** If the two selected pivots are identical (distance 0), the denominator in the projection formula becomes zero. A guard assigns a coordinate of 0 and skips the dimension [G].
- **Numerical Precision:** The residual distance calculation involves subtracting squared distances, which can lead to negative values due to floating-point errors. A guard clamps the residual to 0 before taking the square root [G].
- **Heuristic Accuracy:** The pivot selection heuristic does not guarantee the true furthest pair, which may slightly reduce distance preservation accuracy compared to exhaustive search, but is necessary for linear time complexity [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(nk)$, where $n$ is the number of instances and $k$ is the target dimension [S1, unofficial]. Some sources claim $O(n)$ complexity per dimension [S2, unofficial].
- **Memory Complexity:** $O(nk)$ to store the projected points; distances are computed on the fly without a precomputed matrix [S1, unofficial].
- **Strengths:** Linear time complexity makes it suitable for large datasets; preserves global distances reasonably well [S1, unofficial].
- **Limitations:** The heuristic pivot selection may not preserve distances as accurately as exhaustive methods; assumes the data can be well-approximated by a low-dimensional Euclidean space [S1, unofficial].
- **When to Use:** When dealing with large datasets where classical MDS is too slow, and global distance preservation is important [S1, unofficial].

## Background
Pivot MDS, also known as FastMap, was introduced by Faloutsos and Lin in 1995 [S1, unofficial]. It was developed as a linear-time variant of Multidimensional Scaling to address the computational infeasibility of classical MDS for large datasets [S1, unofficial]. The method has been applied in volume rendering for transfer function design, where it helps reduce high-dimensional feature vectors to a manageable 2D or 3D space for visualization [S1, unofficial].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
