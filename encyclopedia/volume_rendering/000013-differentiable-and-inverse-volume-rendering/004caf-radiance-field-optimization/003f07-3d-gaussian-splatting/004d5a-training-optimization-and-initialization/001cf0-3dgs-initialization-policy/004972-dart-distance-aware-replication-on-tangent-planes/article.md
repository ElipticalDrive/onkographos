# DART (Distance-Aware Replication on Tangent-planes)
## Overview
DART is a pre-processing initialization method for 3D Gaussian Splatting that enriches a sparse Structure-from-Motion (SfM) point cloud by generating additional samples on local tangent planes [S1]. It addresses the problem of sparse seeding in scene peripheries and low-texture regions by assigning disk radii that grow with the point’s distance from a robust scene center [S1]. Unlike dynamic densification strategies that spawn primitives during training based on view-space gradients, DART acts once before optimization using only the initial SfM geometry, making it independent of the backbone’s densification dynamics [S1].

## How it works
**Inputs:**
- Initial SfM point cloud $P = \{p_i\}_{i=1}^N$, where each $p_i$ has a 3D position and color.
- Multiplier $m$: total number of points generated per original SfM point (including the original).
- Radius bounds $\rho_{min}, \rho_{max}$: lower and upper bounds for the sampling disk radius, expressed as fractions of the robust scene radius $R$.
- $K_{nn}$: number of nearest neighbors for PCA normal estimation.
- $T_w$: number of Weiszfeld iterations for center computation.

**Outputs:**
- Augmented point cloud $P_{aug}$ containing the original points and $m-1$ new samples per original point.

**Data Structures:**
- Points are represented as tuples $(x, y, z, r, g, b)$.
- Vectors are 3D Euclidean vectors.

**Steps:**

1. **Compute Robust Scene Center:**
   Calculate the L1-median (geometric median) $c$ of the point cloud $P$ using the Weiszfeld iteration [S1]. Initialize $c^{(0)}$ as the arithmetic mean of $P$. Iterate for $T_w$ steps:
   $$ c^{(k+1)} = \frac{\sum_{i=1}^N \frac{p_i}{\|p_i - c^{(k)}\|}}{\sum_{i=1}^N \frac{1}{\|p_i - c^{(k)}\|}} $$
   [S1]

2. **Compute Robust Scene Radius:**
   Calculate the robust scene radius $R$ as the 95th-percentile distance from the center $c$ [S1]:
   $$ R = \text{Percentile}_{95}\left(\{ \|p_i - c\| : p_i \in P \}\right) $$
   [S1]

3. **Initialize Augmented Cloud:**
   Set $P_{aug} \leftarrow P$.

4. **Process Each Point:**
   For each point $p_i \in P$:
   
   a. **Distance Calculation:**
      Compute the Euclidean distance $d_i$ from the center:
      $$ d_i = \|p_i - c\| $$
      [S1]
      
      Compute the normalized, clipped distance $\hat{d}_i$:
      $$ \hat{d}_i = \text{clip}\left(\frac{d_i}{R}, 0, 1\right) $$
      [S1]
      Normalizing by $R$ prevents distant outliers from collapsing the radius schedule toward $\rho_{min}$ [S1].

   b. **Tangent-Plane Estimation:**
      Identify the $K_{nn}$ nearest neighbors $N_i$ of $p_i$ in $P$ using Euclidean distance [S1].
      Compute the centroid $\bar{p}_i$ of $N_i$:
      $$ \bar{p}_i = \frac{1}{K_{nn}} \sum_{j \in N_i} p_j $$
      [S1]
      
      Compute the $3 \times 3$ sample covariance matrix $\text{Cov}(N_i)$:
      $$ \text{Cov}(N_i) = \frac{1}{K_{nn}} \sum_{j \in N_i} (p_j - \bar{p}_i)(p_j - \bar{p}_i)^\top $$
      [S1]
      
      Estimate the local normal $n_i$ as the unit eigenvector of $\text{Cov}(N_i)$ associated with the smallest eigenvalue:
      $$ n_i = \arg\min_{\|u\|_2=1} u^\top \text{Cov}(N_i) u $$
      [S1]
      
      Construct an orthonormal tangent basis $(e_i^{(1)}, e_i^{(2)})$ by choosing any unit vector $e_i^{(1)}$ perpendicular to $n_i$ and setting:
      $$ e_i^{(2)} = n_i \times e_i^{(1)} $$
      [S1]
      The normal $n_i$ is used only to construct the frame and is not stored on the replicas [S1].

   c. **Radius Assignment:**
      Assign the disk radius $\rho_i$ as a linear function of $\hat{d}_i$:
      $$ \rho_i = R \cdot \left[ \rho_{min} + \hat{d}_i (\rho_{max} - \rho_{min}) \right] $$
      [S1]
      A point at the scene center receives $\rho_i = \rho_{min} R$; a point at or beyond the 95th-percentile shell receives $\rho_i = \rho_{max} R$ [S1].

   d. **Disk Sampling:**
      For $k$ from $1$ to $m-1$:
      1. Draw two independent uniform random variables $\xi_1, \xi_2 \sim \text{Uniform}[0, 1]$ [S1].
      2. Compute polar coordinates:
         $$ r = \rho_i \xi_1 $$
         $$ \theta = 2\pi \xi_2 $$
         [S1]
         *Note: The source explicitly defines $r = \rho_i \xi_1$ [S1].*
      3. Compute the new sample position $q$:
         $$ q = p_i + r \cos(\theta) e_i^{(1)} + r \sin(\theta) e_i^{(2)} $$
         [S1]
      4. Assign the color of the parent point to $q$: $q.\text{color} \leftarrow p_i.\text{color}$ [S1].
      5. Add $q$ to $P_{aug}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def dart(P, m, rho_min, rho_max, Knn, Tw):
    """
    P: list of points, each point is a dict or object with 'pos' (np.array, shape (3,)) and 'color' (np.array, shape (3,))
    m: multiplier
    rho_min: lower bound fraction
    rho_max: upper bound fraction
    Knn: number of neighbors for PCA
    Tw: Weiszfeld iterations
    Returns: list of augmented points
    """
    N = len(P)
    positions = np.array([p['pos'] for p in P])
    
    # Step 1: Compute robust scene center c (L1-median via Weiszfeld)
    c = np.mean(positions, axis=0) # Initialize at arithmetic mean
    for _ in range(Tw):
        dists = np.linalg.norm(positions - c, axis=1)
        # Handle potential division by zero if a point coincides with c
        dists = np.maximum(dists, 1e-8) 
        weights = 1.0 / dists
        c = np.sum(positions * weights[:, np.newaxis], axis=0) / np.sum(weights)
        
    # Step 2: Compute robust scene radius R (95th-percentile distance)
    dists_from_c = np.linalg.norm(positions - c, axis=1)
    R = np.percentile(dists_from_c, 95)
    
    # Step 3: Initialize augmented cloud
    Paug = list(P) # Copy original points
    
    # Precompute KNN indices for efficiency if needed, but doing per-point for clarity
    # Using a simple brute-force or KDTree for KNN
    from scipy.spatial import KDTree
    tree = KDTree(positions)
    
    for i, p_i in enumerate(P):
        pos_i = p_i['pos']
        
        # Stage (i): Distance
        d_i = np.linalg.norm(pos_i - c)
        d_hat_i = np.clip(d_i / R, 0, 1)
        
        # Stage (ii): Tangent-plane estimation
        # Find Knn nearest neighbors
        # Note: KDTree query returns distances and indices. The first neighbor is the point itself.
        # We need Knn neighbors excluding the point itself or including? 
        # The source says "Knn nearest neighbors Ni". Usually, for PCA, self is included or excluded.
        # Standard practice often excludes self to avoid degenerate covariance, but source doesn't specify.
        # Assuming we take Knn neighbors from the cloud. If Knn includes self, it's fine as long as N >= Knn.
        # Let's assume we query Knn+1 to exclude self, or just Knn if self is negligible.
        # Given the source says "Ni subset of P", and PCA on neighbors, let's take Knn neighbors.
        # To be safe and standard, let's take Knn neighbors excluding self.
        dists_knn, idx_knn = tree.query(pos_i, k=Knn + 1)
        # idx_knn[0] is the point itself.
        neighbor_indices = idx_knn[1:]
        neighbors = positions[neighbor_indices]
        
        # Compute centroid of neighborhood
        p_bar_i = np.mean(neighbors, axis=0)
        
        # Compute covariance matrix
        diff = neighbors - p_bar_i
        Cov_Ni = (diff.T @ diff) / Knn
        
        # Compute normal: eigenvector corresponding to smallest eigenvalue
        eigenvalues, eigenvectors = np.linalg.eigh(Cov_Ni)
        # Smallest eigenvalue is at index 0
        n_i = eigenvectors[:, 0]
        n_i = n_i / np.linalg.norm(n_i)
        
        # Construct orthonormal tangent basis
        # Choose any unit vector e1 perpendicular to n
        # A robust way: if n is not parallel to [1,0,0], use cross with [1,0,0]
        if np.abs(n_i[0]) < 0.9:
            e1 = np.cross(n_i, np.array([1.0, 0.0, 0.0]))
        else:
            e1 = np.cross(n_i, np.array([0.0, 1.0, 0.0]))
        e1 = e1 / np.linalg.norm(e1)
        
        e2 = np.cross(n_i, e1)
        
        # Stage (iii): Radius assignment
        rho_i = R * (rho_min + d_hat_i * (rho_max - rho_min))
        
        # Stage (iv): Disk sampling
        for k in range(m - 1):
            xi1 = np.random.uniform(0, 1)
            xi2 = np.random.uniform(0, 1)
            
            r = rho_i * xi1
            theta = 2 * np.pi * xi2
            
            q_pos = pos_i + r * np.cos(theta) * e1 + r * np.sin(theta) * e2
            q_color = p_i['color']
            
            Paug.append({'pos': q_pos, 'color': q_color})
            
    return Paug
```

**Parameters:**
- $m$: Multiplier for the number of additional samples. Total points per original point becomes $m$ [S1].
- $\rho_{min}$: Lower bound for the disk radius, relative to $R$ [S1].
- $\rho_{max}$: Upper bound for the disk radius, relative to $R$ [S1].
- $K_{nn}$: Number of nearest neighbors for PCA [S1].
- $T_w$: Number of Weiszfeld iterations [S1].

**Edge Cases and Pitfalls:**
- If a point $p_i$ coincides with the center $c$, the Weiszfeld iteration denominator term $\|p_i - c\|$ becomes zero. The pseudocode handles this with a small epsilon [G].
- The source explicitly uses $r = \rho_i \xi_1$ for disk sampling, which does not produce a uniform distribution over the disk area (standard uniform sampling uses $\sqrt{\xi_1}$) [S1]. Implementers should follow the source formula exactly [S1].
- The normal estimation relies on $K_{nn}$ being sufficient to define a plane. In sparse regions, this may be unstable [G].

## Operating characteristics
- **Time Complexity:** The method is a pre-processing step. Computing the center and radius is $O(N)$ or $O(N \log N)$ depending on the median/percentile algorithm. For each of the $N$ points, KNN search is $O(K_{nn} \log N)$ or $O(K_{nn} N)$, PCA is $O(K_{nn})$, and sampling is $O(m)$. Total complexity is roughly $O(N (K_{nn} \log N + m))$.
- **Memory Complexity:** $O(N \cdot m)$ to store the augmented cloud.
- **Strengths:** It is independent of the backbone's densification dynamics, acting only once before training [S1]. It specifically targets sparse periphery regions by increasing disk radius with distance [S1].
- **Limitations:** It relies on the quality of the initial SfM point cloud. It does not address over-dense regions (which is handled by the sibling method AGE) [S1].
- **When to use:** When the initial SfM cloud is sparse in the periphery or low-texture areas, and a static initialization is preferred over dynamic densification for the initial spread.

## Background
DART was introduced in the paper "DART-3DGS: Distance-Aware Replication and Age-Gated Densification for 3D Gaussian Splatting" [S1]. It is designed to work with vanilla 3D Gaussian Splatting and anchor-based methods like Scaffold-GS [S1]. It is part of a pair of initialization and densification controls, with AGE (Age-Gated Eligibility) handling the opposite problem of uncontrolled cascading splits in dense regions [S1].

## Sources
[S1] DART-3DGS: Distance-Aware Replication and Age-Gated Densification for 3D Gaussian Splatting, https://europepmc.org/article/PMC/PMC13705904, fetched 2026-10-27
