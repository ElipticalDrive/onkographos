# Statistical Outlier Removal

Also called: Open3D Statistical Outlier Removal, Outlier Vertex Removal, Outlier Vertex Removal and Poisson Surface Reconstruction, Point Cloud Denoising

## Overview
Statistical Outlier Removal identifies and removes outlier vertices in a point cloud based on their proximity to other vertices [S1, unofficial]. It solves the problem of unwanted points (e.g., backgrounds or floors) being incorporated into generated meshes during point cloud to mesh conversion [S1, unofficial]. Unlike sibling methods that rely on learned opacity or fixed distance metrics, this method leverages local neighborhood density statistics [Parent](8464-point-cloud-filtering-and-noise-removal.md)[G].

## How it works
**Inputs:** A point cloud $P = \{p_1, p_2, ..., p_N\}$ where each $p_i \in \mathbb{R}^3$ [G].
**Outputs:** A cleaned point cloud $P_{clean} \subset P$ [G].
**Parameters:**
- $k$: Number of nearest neighbors to consider for statistical analysis. [S2] specifies $K=64$.
- $\sigma_{threshold}$: Standard deviation threshold for outlier removal. Determined heuristically via experimentation [S1, unofficial].

**Data Structures:**
- Array of mean distances $D$ of size $N$ [G].

**Steps:**
1. For each point $p_i \in P$, find the $k$ nearest neighbors in $P$ [G].
2. Calculate the mean distance $\bar{d}_i$ from $p_i$ to its $k$ neighbors [S2].
3. Calculate the global mean $\mu_D$ and standard deviation $\sigma_D$ of all values in $D$ [G].
4. Identify outliers: A point $p_i$ is an outlier if $\bar{d}_i > \mu_D + \sigma_{threshold} \cdot \sigma_D$ [S1, unofficial].
5. Remove all identified outliers from $P$ to form $P_{clean}$ [S1, unofficial].
6. [G] Apply Poisson surface reconstruction to $P_{clean}$ to generate a mesh [S1, unofficial].
7. [G] Remove mesh parts with density lower than a certain quantile to acquire the final 3D mesh [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def statistical_outlier_removal(points, k, sigma_threshold):
    """
    points: List of 3D coordinates (N, 3)
    k: Number of nearest neighbors
    sigma_threshold: Standard deviation threshold multiplier
    Returns: Cleaned points list
    """
    n = len(points)
    mean_distances = []

    # Compute mean distance to k nearest neighbors for each point
    for i in range(n):
        dists = []
        for j in range(n):
            if i == j:
                continue
            dx = points[i][0] - points[j][0]
            dy = points[i][1] - points[j][1]
            dz = points[i][2] - points[j][2]
            dists.append(math.sqrt(dx*dx + dy*dy + dz*dz))
        dists.sort()
        # Take mean of k nearest neighbors
        k_nearest = dists[:k]
        mean_d = sum(k_nearest) / len(k_nearest)
        mean_distances.append(mean_d)

    # Calculate global statistics
    mu_D = sum(mean_distances) / n
    variance = sum((d - mu_D)**2 for d in mean_distances) / n
    sigma_D = math.sqrt(variance)

    # Identify and remove outliers
    threshold = mu_D + sigma_threshold * sigma_D
    cleaned_points = [points[i] for i in range(n) if mean_distances[i] <= threshold]

    return cleaned_points
```

**Edge Cases and Pitfalls:**
- **Boundaries:** Points on the boundary of the object may have fewer neighbors or larger distances, potentially being incorrectly removed if $k$ is not chosen carefully [G].
- **Degenerate Input:** If the point cloud is too sparse, the statistical distribution may not be well-defined [G].

## Subtopics

### [Alpha-based Point Cleaning](19085-alpha-based-point-cleaning.md)
Alpha-based Point Cleaning is a post-processing and iterative optimization strategy used in neural point rendering to remove outlier points from a point cloud based on their learned opacity (alpha) values. It solves the problem of noisy initial reconstructions (e.g., from COLMAP) or spurious points generated during training (e.g., from spawning mechanisms) by explicitly identifying points with low confidence. Unlike sibling methods that rely on geometric density or distance metrics, this method leverages the neural optimization process itself, where valid geometry converges to high opacity while outliers converge to zero. It is a child of Statistical Outlier Removal, specifically utilizing the alpha parameter rather than statistical distance [Parent](17819-statistical-outlier-removal.md).

### [Point Cloud Regularization](20581-point-cloud-regularization.md)
Point Cloud Regularization is a post-processing pipeline applied to an initial colored point cloud to remove noise, duplicates, and outliers before training a 3D Gaussian Splatting (3DGS) model. It solves the problem of unreliable initialization caused by merging Structure-from-Motion (SfM) points with lightweight 3DGS primitive centers, which introduces depth ambiguity, duplicated points from densification, and unstable tracks. It sets itself apart from its sibling [Alpha-based Point Cleaning](19085-alpha-based-point-cleaning.md) by explicitly enforcing multi-view consistency and geometric normal agreement rather than relying solely on alpha-blending reliability metrics.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Effective at removing isolated noise points and background artifacts [S1, unofficial]. Simple to implement [G].
- **Limitations:** Requires heuristic tuning of parameters ($k$, $\sigma_{threshold}$) [S1, unofficial]. May remove valid points in sparse regions [G].
- **When to use:** When preprocessing point clouds for mesh generation to remove background noise [S1, unofficial].
- **When not:** When preserving sparse but valid geometry is critical, or when computational resources are extremely limited [G].

## Background
The method was used to circumvent the inclusion of unwanted points (backgrounds or floors) in generated meshes during point cloud to mesh conversion [S1, unofficial]. The values of the thresholds were determined heuristically via experimentation [S1, unofficial]. It is also referenced in the context of point cloud denoising for neural radiance field editing, where $K=64$ is used for nearest neighbor distance calculation [S2].

## Sources
[S1] DITTO-NeRF: Diffusion-based Iterative Text To Omni-directional 3D Model, https://arxiv.org/abs/2304.02827v1, fetched 2026-10-26 (unofficial)
[S2] NeuralEditor: Editing Neural Radiance Fields via Manipulating Point Clouds, https://doi.org/10.1109/cvpr52729.2023.01197, fetched 2026-10-26
