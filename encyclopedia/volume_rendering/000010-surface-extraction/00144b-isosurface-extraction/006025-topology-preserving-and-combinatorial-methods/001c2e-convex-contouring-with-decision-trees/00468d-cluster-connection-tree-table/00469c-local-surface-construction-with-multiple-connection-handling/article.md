# Local Surface Construction with Multiple Connection Handling

Also called: Eigen-based cluster division, Local convexity construction

## Overview
This method constructs surfaces from local convexities defined by two connected clusters in adjacent slices, specifically addressing the failure of standard 3D hull algorithms on non-convex objects [S1]. It handles 1:n branching or n:1 merging connections by dividing them into multiple 1:1 connections using eigen-analysis, thereby preventing surface smoothing of details or unnecessary duplication of surface patches in common areas [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of data points $X$ belonging to a branched cluster (a cluster with multiple connections to clusters in the next slice) [S1]. Each point is represented as $\tilde{t}_{ij} = [u, v]^T$ [S1].
*   **Output:** A division of the source cluster into $n$ subclusters, creating $n$ distinct 1:1 connections [S1].

**Data Structures:**
*   **Data Matrix $X$:** A collection of $m$ data points in the branched cluster [S1].
*   **Covariance Matrix $C$:** A $2 \times 2$ matrix computed from the data points [S1].
*   **Eigenvectors:** Orthonormal column vectors $\tilde{e}_1, \tilde{e}_2$ [S1].

**Steps:**
1.  **Identify Multiple Connections:** Detect 1:n branching or n:1 merging connections in the Cluster Connection Tree Table [S1].
2.  **Compute Covariance Matrix:** Calculate the covariance matrix $C$ of the data points $X$ in the branched cluster [S1]:
    $$C = \sum_j \sum_i (\tilde{t}_{ij} - \tilde{m})(\tilde{t}_{ij} - \tilde{m})^T$$
    where $\tilde{m}$ is the mean of $X$ [S1].
3.  **Eigen Analysis:** Perform eigen-decomposition on $C$. The projection of $C$ onto orthonormal vectors $\tilde{e}_1, \tilde{e}_2$ is:
    $$C[\tilde{e}_1 \tilde{e}_2] = [\tilde{e}_1 \tilde{e}_2] \begin{bmatrix} \lambda_1 & 0 \\ 0 & \lambda_2 \end{bmatrix}$$
    where $\lambda_i$ is the eigenvalue corresponding to eigenvector $\tilde{e}_i$ [S1].
4.  **Select Best Representative Vector:** Identify the eigenvector corresponding to the maximum eigenvalue as the best representative vector [S1].
5.  **Project and Normalize:** Project each point $\tilde{t}_{ij}$ onto the best eigenvector $\tilde{e}_1$ to find the distribution:
    $$l_{ij} = \tilde{t}_{ij}^T \tilde{e}_1$$
    Determine the minimum and maximum locations $(l_{min}, l_{max})_j$ for each target cluster $j$ [S1]. Normalize these data distributions [S1].
6.  **Divide Source Cluster:** Project the source cluster data onto the best eigenvector and divide it into subclusters according to the normalized distributions to create $n$ 1:1 connections [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def divide_cluster(source_points, target_clusters):
    """
    source_points: list of 2D vectors [u, v] representing the source cluster
    target_clusters: list of lists of 2D vectors, one list per target cluster j
    Returns: list of n subclusters (lists of points)
    """
    m = len(source_points)
    mean = [sum(p[0] for p in source_points) / m, sum(p[1] for p in source_points) / m]
    
    # Step 2: Compute Covariance Matrix C (2x2)
    C = [[0.0, 0.0], [0.0, 0.0]]
    for p in source_points:
        dx = p[0] - mean[0]
        dy = p[1] - mean[1]
        C[0][0] += dx * dx
        C[0][1] += dx * dy
        C[1][0] += dy * dx
        C[1][1] += dy * dy
        
    # Step 3: Eigen decomposition of 2x2 symmetric matrix
    a, b, d = C[0][0], C[0][1], C[1][1]
    trace = a + d
    discriminant = (a - d)**2 + 4 * b**2
    sqrt_disc = discriminant**0.5
    lambda1 = (trace + sqrt_disc) / 2.0
    lambda2 = (trace - sqrt_disc) / 2.0
    
    # Step 4: Select eigenvector for max eigenvalue (lambda1)
    if abs(b) < 1e-9:
        if abs(lambda1 - a) < 1e-9:
            e1 = [1.0, 0.0]
        else:
            e1 = [0.0, 1.0]
    else:
        e1 = [b, lambda1 - a]
        norm_e1 = (e1[0]**2 + e1[1]**2)**0.5
        e1 = [e1[0] / norm_e1, e1[1] / norm_e1]
        
    # Step 5: Project source points onto e1
    source_projections = [(p[0] * e1[0] + p[1] * e1[1], p) for p in source_points]
    source_projections.sort(key=lambda x: x[0])
    
    # Project target clusters onto e1 to find bounds (l_min, l_max)_j
    n = len(target_clusters)
    bounds = []
    for t_clusters in target_clusters:
        t_projs = [p[0] * e1[0] + p[1] * e1[1] for p in t_clusters]
        bounds.append((min(t_projs), max(t_projs)))
        
    # Normalize bounds to [0, 1] range
    min_all = bounds[0][0]
    max_all = bounds[-1][1]
    range_all = max_all - min_all
    if range_all < 1e-9:
        range_all = 1.0
    thresholds = [(b[0] - min_all) / range_all for b in bounds]
    thresholds.append(1.0)
        
    # Step 6: Divide source cluster according to normalized distributions
    subclusters = [[] for _ in range(n)]
    source_idx = 0
    for j in range(n):
        threshold = thresholds[j]
        while source_idx < len(source_projections) and source_projections[source_idx][0] <= threshold * range_all + min_all:
            subclusters[j].append(source_projections[source_idx][1])
            source_idx += 1
    while source_idx < len(source_projections):
        subclusters[n-1].append(source_projections[source_idx][1])
        source_idx += 1
        
    return subclusters
```

**Parameters:**
*   **Slice Size:** Should be "reasonably small" to ensure clusters are locally convex where possible [S1].
*   **Number of Targets ($n$):** The number of clusters in the adjacent slice that the source cluster connects to [S1].

**Edge Cases and Pitfalls:**
*   **Surface Expansion/Smoothing:** Applying a simple 3D hull to multiple connections (1:n or n:1) causes unexpected surface expansion (smoothing details) or unnecessary duplication of surface patches in common areas [S1].
*   **Arbitrary Distant Branching:** If a cluster branches to several clusters in an arbitrary distant position, the 3D hull algorithm may create unexpected connections that expand the surface construction [S1].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources [S1].

**Measured Performance:**
The algorithm was evaluated on four objects with various shapes [S1]. Specific performance metrics are not provided in the notes [S1].

**Strengths and Limitations:**
*   **Strengths:** Prevents surface smoothing and duplication in non-convex objects by correctly handling branching and merging connections [S1].
*   **Limitations:** Requires eigen-analysis, which adds computational overhead compared to simple 1:1 connections [S1]. Relies on the assumption that the principal component (best representative vector) adequately separates the branches [S1].

**When to Use:**
Use when the Cluster Connection Tree Table indicates multiple connections (1:n or n:1) between clusters in adjacent slices, particularly for non-convex objects [S1].

## Background
**Origin:**
The method was proposed in the paper "Local hull-based surface construction of volumetric data from silhouettes" [S1].

**History and Development:**
The method addresses limitations of the 3D hull algorithm when applied to local convexities that are not truly convex due to multiple connections [S1]. It is part of a pipeline that includes Cluster Connection Tree Table construction [S1].

## Sources
[S1] Local hull-based surface construction of volumetric data from silhouettes, https://doi.org/10.1109/tip.2008.926149, fetched 2026-10-30
