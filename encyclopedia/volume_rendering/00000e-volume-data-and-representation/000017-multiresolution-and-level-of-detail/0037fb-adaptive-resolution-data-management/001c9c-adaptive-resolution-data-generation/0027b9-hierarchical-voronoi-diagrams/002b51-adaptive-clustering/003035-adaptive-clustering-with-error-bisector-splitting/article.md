# Adaptive Clustering with Error-Bisector Splitting

Also called: Adaptive clustering with quadratic polynomials, Cluster subdivision by positive/negative error centers, Quadratic Polynomial Approximation Clustering, Quadratic clustering, Splitting along bisector of c+ and c-

## Overview
Adaptive clustering with error-bisector splitting is a method for generating a data-dependent hierarchy of clusters from scattered 2D points with associated scalar values, using quadratic polynomial approximations to guide subdivision [S1, unofficial]. It solves the problem of efficiently approximating smooth surfaces with a minimal number of clusters for a given error tolerance, outperforming PCA-based splitting by producing long, skinny clusters when curvature is directional [S1, unofficial]. Unlike its sibling [PCA-based Clustering](5048-pca-based-clustering.md), which ignores function values, this method weights spatial distribution by approximation residuals [S1, unofficial].

## How it works
**Inputs:** A set of points $D = \{\mathbf{p}_i\}_{i=1}^N$ in the plane with associated scalar function values $F = \{f_i\}_{i=1}^N$, a prescribed global error bound, and a prescribed maximum number of tiles [S1, unofficial].

**Outputs:** A hierarchical set of clusters (tiles) $\tau_j^k$ at various levels $j$, each associated with a quadratic polynomial approximation $\tilde{f}_k^j$ and a residual error $\epsilon_j^k$ [S1, unofficial].

**Data Structures:**
*   **Cluster:** Contains a list of point indices, the current level $j$, and the fitted quadratic polynomial coefficients [S1, unofficial].
*   **Quadratic Polynomial:** Defined by coefficients $c_{l,m}$ for $l+m \le 2$. In 2D, this is $\tilde{f}(x_1, x_2) = \sum_{l, m \geq 0; l+m \leq 2} c_{l,m} x_1^l x_2^m$ [S1, unofficial].
*   **Residual:** The maximum norm error $\epsilon = \max_i | \tilde{f}(\mathbf{p}_i) - f_i |$ [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization (Level $L_0$):**
    *   Create a single root cluster $\tau_0^1 = D$ containing all points [S1, unofficial].
    *   Fit a quadratic polynomial $\tilde{f}_0^1$ to all points in $\tau_0^1$ using least-squares regression [S1, unofficial].
    *   Compute the residual for $\tau_0^1$ using the maximum norm: $\epsilon_0^1 = \max_{i \in \tau_0^1} | \tilde{f}_0^1(\mathbf{p}_i) - f_i |$ [S1, unofficial].

2.  **Refinement Loop:**
    *   Identify the cluster $\tau_j^m$ in the current level $L_j$ with the maximal residual $\epsilon_j^m$ such that $\epsilon_j^m \ge \epsilon_j^k$ for all other clusters $k$ in $L_j$ [S1, unofficial].
    *   If $\epsilon_j^m$ satisfies the prescribed global error bound or the total number of clusters reaches the prescribed maximum, terminate [S1, unofficial].
    *   **Splitting:**
        *   Compute the positive error weights for each point $\mathbf{p}_i$ in $\tau_j^m$:
            $$ w_i^+ = \begin{cases} \tilde{f}_k^j(\mathbf{p}_i) - f_i & \text{if } \tilde{f}_k^j(\mathbf{p}_i) - f_i > 0 \\ 0 & \text{otherwise} \end{cases} $$
        *   Compute the negative error weights:
            $$ w_i^- = \begin{cases} f_i - \tilde{f}_k^j(\mathbf{p}_i) & \text{if } \tilde{f}_k^j(\mathbf{p}_i) - f_i < 0 \\ 0 & \text{otherwise} \end{cases} $$
        *   Calculate the weighted centers $\mathbf{c}^+$ and $\mathbf{c}^-$:
            $$ \mathbf{c}^+ = \frac{\sum_{i=1}^n w_i^+ \mathbf{p}_i}{\sum_{i=1}^n w_i^+}, \quad \mathbf{c}^- = \frac{\sum_{i=1}^n w_i^- \mathbf{p}_i}{\sum_{i=1}^n w_i^-} $$
            where $n$ is the number of points in the cluster [S1, unofficial].
        *   Define the splitting line as the perpendicular bisector of the segment connecting $\mathbf{c}^+$ and $\mathbf{c}^-$ [S1, unofficial].
        *   If $\mathbf{c}^+ = \mathbf{c}^-$, the direction of subdivision is arbitrary [S1, unofficial].
        *   Partition the points in $\tau_j^m$ into two new clusters $\tau_{j+1}^{2m-1}$ and $\tau_{j+1}^{2m}$ based on which side of the bisector they fall [S1, unofficial].
    *   **Re-fitting:**
        *   For each new cluster, fit a new quadratic polynomial using least-squares [S1, unofficial].
        *   Compute the new residuals $\epsilon_{j+1}^{2m-1}$ and $\epsilon_{j+1}^{2m}$ using the maximum norm [S1, unofficial].
    *   Replace $\tau_j^m$ with the two new clusters in the hierarchy. Increment level $j$ and repeat [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_clustering_error_bisector(points, values, epsilon_max, max_tiles):
    """
    points: list of (x, y) tuples
    values: list of scalar floats
    epsilon_max: float, global error bound
    max_tiles: int, maximum number of clusters
    Returns: list of clusters, each containing indices and polynomial coeffs
    """
    current_level_clusters = []
    root_indices = list(range(len(points)))
    
    # Fit initial polynomial
    coeffs = fit_quadratic_least_squares(points, values, root_indices)
    residual = compute_max_residual(points, values, coeffs, root_indices)
    
    current_level_clusters.append({
        'indices': root_indices,
        'coeffs': coeffs,
        'residual': residual,
        'level': 0
    })
    
    total_clusters = 1
    
    while total_clusters < max_tiles:
        # Find cluster with max residual
        max_idx = -1
        max_residual = -1.0
        
        for i, cluster in enumerate(current_level_clusters):
            if cluster['residual'] > max_residual:
                max_residual = cluster['residual']
                max_idx = i
                
        # Termination check
        if max_residual <= epsilon_max:
            break
            
        cluster_to_split = current_level_clusters[max_idx]
        indices = cluster_to_split['indices']
        coeffs = cluster_to_split['coeffs']
        
        # Compute weights
        w_plus = []
        w_minus = []
        for idx in indices:
            p = points[idx]
            f_val = values[idx]
            approx = evaluate_quadratic(p, coeffs)
            error = approx - f_val
            
            if error > 0:
                w_plus.append(error)
                w_minus.append(0.0)
            elif error < 0:
                w_plus.append(0.0)
                w_minus.append(-error)
            else:
                w_plus.append(0.0)
                w_minus.append(0.0)
                
        # Compute centers
        sum_w_plus = sum(w_plus)
        sum_w_minus = sum(w_minus)
        
        if sum_w_plus == 0:
            c_plus = None
        else:
            c_plus = (sum(w_plus[k] * points[indices[k]][0] for k in range(len(indices))) / sum_w_plus,
                      sum(w_plus[k] * points[indices[k]][1] for k in range(len(indices))) / sum_w_plus)
                      
        if sum_w_minus == 0:
            c_minus = None
        else:
            c_minus = (sum(w_minus[k] * points[indices[k]][0] for k in range(len(indices))) / sum_w_minus,
                       sum(w_minus[k] * points[indices[k]][1] for k in range(len(indices))) / sum_w_minus)
                       
        # Determine split line
        if c_plus is None or c_minus is None or c_plus == c_minus:
            # Arbitrary split direction when centers are identical or undefined [S1, unofficial]
            sorted_indices = sorted(indices, key=lambda i: points[i][0])
            mid = len(sorted_indices) // 2
            left_indices = sorted_indices[:mid]
            right_indices = sorted_indices[mid:]
        else:
            # Perpendicular bisector of c_plus and c_minus
            dx = c_plus[0] - c_minus[0]
            dy = c_plus[1] - c_minus[1]
            mx = (c_plus[0] + c_minus[0]) / 2.0
            my = (c_plus[1] + c_minus[1]) / 2.0
            
            left_indices = []
            right_indices = []
            for idx in indices:
                x, y = points[idx]
                val = dx * (x - mx) + dy * (y - my)
                if val > 0:
                    left_indices.append(idx)
                else:
                    right_indices.append(idx)
                    
            # Handle empty sets if all points fall on one side
            if not left_indices:
                left_indices = indices[:len(indices)//2]
                right_indices = indices[len(indices)//2:]
            elif not right_indices:
                right_indices = indices[len(indices)//2:]
                left_indices = indices[:len(indices)//2]

        # Remove old cluster
        current_level_clusters.pop(max_idx)
        total_clusters -= 1
        
        # Add new clusters
        for new_indices in [left_indices, right_indices]:
            new_coeffs = fit_quadratic_least_squares(points, values, new_indices)
            new_residual = compute_max_residual(points, values, new_coeffs, new_indices)
            current_level_clusters.append({
                'indices': new_indices,
                'coeffs': new_coeffs,
                'residual': new_residual,
                'level': cluster_to_split['level'] + 1
            })
            total_clusters += 1
            
    return current_level_clusters

def fit_quadratic_least_squares(points, values, indices):
    # [G] Standard least-squares solver for quadratic polynomial in 2D
    # Constructs design matrix A and target vector b, solves (A^T A) c = A^T b
    n = len(indices)
    A = [[1.0, points[i][0], points[i][1], points[i][0]**2, points[i][0]*points[i][1], points[i][1]**2] for i in indices]
    b = [values[i] for i in indices]
    ATA = [[sum(A[k][r]*A[k][c] for k in range(n)) for c in range(6)] for r in range(6)]
    ATb = [sum(A[k][r]*b[k] for k in range(n)) for r in range(6)]
    M = [ATA[i] + [ATb[i]] for i in range(6)]
    for i in range(6):
        pivot = M[i][i]
        for j in range(i, 7): M[i][j] /= pivot
        for k in range(6):
            if k != i:
                factor = M[k][i]
                for j in range(i, 7): M[k][j] -= factor * M[i][j]
    return [M[i][6] for i in range(6)]

def evaluate_quadratic(point, coeffs):
    x, y = point
    c00, c10, c01, c20, c11, c02 = coeffs
    return c00 + c10*x + c01*y + c20*x**2 + c11*x*y + c02*y**2

def compute_max_residual(points, values, coeffs, indices):
    max_err = 0.0
    for idx in indices:
        err = abs(evaluate_quadratic(points[idx], coeffs) - values[idx])
        if err > max_err:
            max_err = err
    return max_err
```

**Parameters:**
*   **Error Norm:** The maximum norm ($p=\infty$) is used for residual estimation to ensure the approximation stays within a prescribed tolerance [S1, unofficial].
*   **Error Bound ($\epsilon_{max}$):** The threshold for termination, defined as a prescribed global error bound [S1, unofficial].
*   **Max Tiles ($N_{max}$):** Limit on cluster count, defined as a prescribed number of tiles [S1, unofficial].

**Pitfalls:**
*   **Affine Invariance Issues:** An affine-invariant variant of this approach may provide poor approximation behavior in regions where clusters are recursively split along similar directions [S1, unofficial].
*   **Degenerate Centers:** If $\mathbf{c}^+$ and $\mathbf{c}^-$ are identical (or one is undefined because all errors are zero or have the same sign), the splitting direction is arbitrary [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Typically produces a lower number of clusters than PCA-based splitting for the same approximation error, particularly when the function is curved in only one direction, resulting in long and skinny clusters [S1, unofficial].
*   **Strengths:** Efficient triangulation of scattered data; better representation of smooth surfaces via quadratic polynomials compared to piecewise linear approximations; data-dependent hierarchy [S1, unofficial].
*   **Limitations:** Requires 2D plane assumption (generalization to higher dimensions requires local projection via PCA) [S1, unofficial].
*   **When to use:** When approximating smooth surfaces from scattered 2D data with a strict error bound and minimizing cluster count is critical [S1, unofficial].
*   **When not to use:** When affine invariance is strictly required and data is not densely sampled/smooth; when 3D volumetric data is processed directly without projection [S1, unofficial].

## Background
This method was introduced as an adaptive clustering approach for efficient triangulation of scattered data in the plane [S1, unofficial]. It generalizes the concept of error-weighted splitting to quadratic polynomial approximations, offering an improvement over piecewise linear approximations for smooth surfaces [S1, unofficial]. The approach is part of a broader framework for multiresolution modeling in scientific visualization, where hierarchical data structures are used to generate optimal approximations with triangles [S1, unofficial].

## Sources
[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
