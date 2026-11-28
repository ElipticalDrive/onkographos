# Subspace Clustering for Multivariate Volume Visualization

Also called: Dynamic Projections, Subspace-based TF Design

## Overview
This method is a visualization pipeline that uses subspace clustering to identify low-dimensional linear subspaces within high-dimensional multivariate volumetric data, enabling effective Transfer Function (TF) design through animated transitions (dynamic projections) between 2D views of these subspaces [S1]. It sets itself apart from static high-dimensional projection methods (like PCA) by mitigating structural illusions inherent in single-view projections; unlike random projection techniques (e.g., GGobi's grand tour), it uses data-driven subspace clustering to define meaningful transitional targets [S1].

## How it works
The pipeline consists of three main stages: Data Reduction, Subspace Clustering, and Dynamic Projections for TF Design.

### Inputs and Outputs
*   **Input:** Multivariate volumetric dataset $V$ with $N$ voxels, each having $D$ attributes.
*   **Output:** A Transfer Function (colormap) and volume rendering parameters, visualized via dynamic projections.

### Data Structures
1.  **Attribute Space Points:** A set of points $\{x_i \in \mathbb{R}^D\}_{i=1}^T$ representing the volumetric data.
2.  **Representatives:** A reduced set of points $R$ derived from clustering, serving as proxies for the full volume.
3.  **Coefficient Matrix $W$:** A $T \times T$ matrix where $W_{ik}$ describes the relationship between sample $i$ and sample $k$.
4.  **Adjacency Matrix $A$:** A $T \times T$ symmetric matrix used for spectral clustering.
5.  **Subspace Basis $U_j$:** A $D \times d_j$ matrix containing the basis vectors for subspace $S_j$.
6.  **Labels:** An array mapping each representative (and by extension, volume voxels) to a subspace index.

### Numbered Steps

#### 1. Data Reduction Stage
The goal is to reduce the computational load while preserving the structure of the attribute space.

1.  **Histogram Construction:** Construct a histogram of the data points in the attribute space based on their $\ell_2$-norm.
2.  **Sampling:** Sample a fixed percentage of points within each discrete interval of the histogram to ensure coverage of the attribute space.
    *   *Parameter:* Sampling rate. Typical values are 10% to 30%. [S1]
3.  **Clustering:** Perform k-means++ clustering on the sampled points.
    *   *Parameter:* Number of clusters $k$. Typical values are 1500 or 3000. [S1]
4.  **Approximation:** Obtain cluster centers during convergence. Approximate these cluster centers with their nearest neighbors within the initial volumetric dataset. These approximated centers serve as the **representatives** of the entire volume. [S1]

#### 2. Subspace Clustering Stage
Represent the high-dimensional attribute space as a collection of low-dimensional linear subspaces.

1.  **Reconstruction Model:** Solve for the coefficient matrix $W$ such that:
    $$ X \approx XW $$
    subject to $W_{ii} = 0$ for $i = 1 \dots T$.
    *   $X$ is the data matrix of representatives.
    *   Regularization penalties on $W$ include sparsity, low-rank, or L2 norms. [S1]
2.  **Adjacency Matrix:** Compute the adjacency matrix for spectral clustering:
    $$ A = |W| + |W^T| $$
    *   Here, $|W|$ denotes the element-wise absolute value. [S1]
3.  **Segmentation:** Perform spectral clustering on $A$ to segment the representatives into clusters $\Lambda_j$, where each cluster corresponds to a subspace $S_j$. [S1]
4.  **Basis Estimation:** For each subspace $S_j$ with index set $\Lambda_j$, estimate the basis $U_j \in \mathbb{R}^{D \times d_j}$ by solving:
    $$ U_j = \arg \min_U \sum_{i \in \Lambda_j} \left\| U^T x_i - \sum_{k \neq i, k \in \Lambda_j} W_{ik} U^T x_k \right\|_2^2 $$
    subject to $U^T U = I$.
    *   The solution is obtained via generalized eigenvalue decomposition of $X_j (I - W_j)^T (I - W_j) X_j^T$.
    *   $U_j$ consists of the eigenvectors corresponding to the $d_j$ smallest eigenvalues. [S1]

#### 3. View Generation and Dynamic Projections
1.  **View Definition:** Use the estimated basis vectors of each subspace to define viewpoints. Create projections onto pairs of basis vectors to generate a set of 2D views from each subspace.
    *   *Note:* A subspace of dimension $d_j$ produces $\binom{d_j}{2}$ 2D views. [S1]
2.  **Dynamic Projections:** Define dynamic projections as animated transitions between pairs of linear projections (views).
    *   Unlike methods that use random generation (e.g., grand tour) or projection pursuit, transitional targets are obtained via the subspace clustering framework. [S1]
3.  **Point Selection:** Provide two approaches for selecting points in the dynamic projection canvas:
    *   **Lasso Selection:** Paints individual points for sculpting TF regions.
    *   **Neighborhood Selection:** Paints a point and its neighbors within a chosen radius in the high-dimensional attribute space. [S1]

#### 4. Transfer Function (TF) Design
Three modes of TF design are supported, leveraging subspace labels assigned to representatives:

1.  **Automatic TF Design:**
    *   Assign colors based on subspace labels.
    *   During dynamic transitions, points in the subspace of the target view increase opacity, while all other subspaces increase transparency. This yields smooth color transitions in the volume visualization. [S1]
2.  **Semi-automatic TF Design:**
    *   Import subspace labels as an initial design.
    *   Modify and refine the TF by exploiting relations among data parts via animated transitions among multiple views. [S1]
3.  **Manual TF Design:**
    *   Sculpt the colormap manually across multiple views.
    *   Subspace labels are treated as latent information to provide freedom and flexibility. [S1]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def subspace_clustering_viz_pipeline(volume_data, sampling_rate=0.1, num_clusters=1500):
    """
    volume_data: List of D-dimensional vectors representing volumetric attributes.
    sampling_rate: Fraction of points to sample per histogram bin.
    num_clusters: Number of clusters for k-means++.
    """
    # 1. Data Reduction
    # Construct histogram based on L2-norm
    norms = [norm(x) for x in volume_data]
    histogram_bins = create_histogram(norms)
    
    # Sample points
    sampled_points = []
    for bin in histogram_bins:
        points_in_bin = get_points_in_norm_range(volume_data, bin.min, bin.max)
        num_to_sample = int(len(points_in_bin) * sampling_rate)
        sampled_points.extend(random_sample(points_in_bin, num_to_sample))
        
    # K-means++ Clustering
    cluster_centers = kmeans_pp(sampled_points, k=num_clusters)
    
    # Approximate centers with nearest neighbors in original data
    representatives = []
    for center in cluster_centers:
        nearest = find_nearest_neighbor(center, volume_data)
        representatives.append(nearest)
        
    # Map original volume points to nearest representative
    volume_labels = [find_nearest_rep_idx(pt, representatives) for pt in volume_data]
    
    # 2. Subspace Clustering
    X = matrix(representatives) # Shape: T x D
    T, D = X.shape
    
    # Solve Reconstruction Model: X ≈ XW, s.t. W_ii = 0
    # Regularization: Sparsity, Low-rank, or L2
    W = solve_reconstruction(X, regularization='sparsity')
    
    # Compute Adjacency Matrix
    A = abs(W) + abs(W.T)
    
    # Spectral Clustering to get clusters Lambda_j
    clusters = spectral_clustering(A)
    
    # 3. Basis Estimation
    subspace_bases = {}
    for j, cluster_indices in enumerate(clusters):
        X_j = X[cluster_indices, :]
        W_j = W[cluster_indices, :]
        
        # Compute Matrix for Eigenvalue Decomposition
        # M = X_j (I - W_j)^T (I - W_j) X_j^T
        I_j = identity(len(cluster_indices))
        diff = I_j - W_j
        M = X_j @ diff.T @ diff @ X_j.T
        
        # Get eigenvectors corresponding to smallest eigenvalues
        eigenvalues, eigenvectors = eig(M)
        sorted_indices = argsort(eigenvalues)
        
        # Determine dimension d_j (e.g., based on eigenvalue gap or fixed)
        d_j = estimate_dimension(eigenvalues)
        
        # Basis U_j is the first d_j eigenvectors
        subspace_bases[j] = eigenvectors[:, sorted_indices[:d_j]]
        
    # 4. View Generation
    views = []
    for j, basis in subspace_bases.items():
        # Generate 2D views from pairs of basis vectors
        for i in range(basis.shape[1]):
            for k in range(i + 1, basis.shape[1]):
                view_proj = basis[:, [i, k]]
                views.append((j, view_proj))
                
    return representatives, volume_labels, views, subspace_bases

def dynamic_projection_animation(views, current_view_idx, target_view_idx, duration=1.0):
    """
    Animates transition between two 2D views using interpolation.
    """
    # Interpolate projection matrices or view parameters
    # This relies on the infrastructure from GGobi-style frame interpolation
    pass

def tf_design_automatic(representatives, volume_labels, views, current_target_view_subspace):
    """
    Automatic TF: Opacity based on subspace membership relative to target view.
    """
    tf = {}
    for i, rep in enumerate(representatives):
        label = volume_labels[i]
        # If the label matches the subspace of the target view, high opacity
        if label == current_target_view_subspace:
            tf[i] = (color, high_opacity)
        else:
            tf[i] = (color, low_opacity)
    return tf
```

### Parameters
*   **Sampling Rate:** Fixed percentage of points sampled from histogram intervals. Typical value: 10% to 30%. [S1]
*   **Number of Clusters ($k$):** For k-means++. Typical values: 1500 or 3000. [S1]
*   **Regularization:** Choice of penalty on $W$ (sparsity, low-rank, L2). [S1]

### Edge Cases and Pitfalls
*   **Structural Illusions:** High dimensionality and complex attribute space structure mean no single projection faithfully represents structural relations. Dynamic projections across multiple views are required to mitigate errors caused by these illusions. [S1]
*   **Intersecting Subspaces:** Standard PCA on each subspace independently fails when subspaces intersect. The proposed method uses a generalized eigenvalue decomposition that accounts for inter-sample relationships via $W$. [S1]
*   **Scalability:** Hardware capacities limit the size of raw volumetric data and the number of points for smooth animation. Data reduction is essential. [S1]

## Operating characteristics
*   **Time Complexity:**
    *   k-means++: $O(n k^2)$ times the number of iterations, where $n$ is the number of data points and $k$ is the number of clusters. [S1]
    *   Subspace Clustering: $O(m^2 d)$ time to construct the affinity matrix, where $m$ is the number of samples (representatives) and $d$ is the dimension. [S1]
*   **Memory Complexity:** $O(m^2)$ to store the affinity matrix in subspace clustering. [S1]
*   **Strengths:**
    *   Reveals unseen structures (e.g., separated subspaces) that appear connected in static PCA views. [S1]
    *   Provides a multi-view canvas for effective TF design, reducing structural illusions. [S1]
    *   Automatic TF design yields smooth color transitions during dynamic projections. [S1]
*   **Limitations:**
    *   Computationally expensive for large datasets without reduction. [S1]
    *   Requires careful selection of sampling rate and number of clusters. [S1]
*   **When to use:** When exploring high-dimensional multivariate volumetric data where structural relationships are complex and static projections are insufficient. [S1]
*   **When not to use:** When real-time interaction on very large raw datasets is required without pre-processing, or when the attribute space does not exhibit low-dimensional subspace structure. [S1]

## Background
The method was introduced in the paper "Multivariate volume visualization through dynamic projections" [S1]. It builds upon infrastructure developed for high-dimensional data exploration [23] and adapts it for multivariate TF design and volume rendering. The dynamic projection technique uses mathematical formulation similar to GGobi [29] but replaces random generation with subspace clustering-derived targets. The implementation uses C++ and Qt for the interface, k-means++ from the Mobile Robot Programming Toolkit [1], and MATLAB for subspace clustering prototyping. [S1]

## Sources
[S1] Multivariate volume visualization through dynamic projections, https://doi.org/10.1109/ldav.2014.7013202, fetched 2026-10-26
