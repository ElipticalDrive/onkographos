# Hierarchical SGGX Aggregation

Also called: Hierarchical SGGX clustering, SGG-H, SGGX-H

## Overview
Hierarchical SGGX Aggregation is a method for progressively reducing the number of Scalable Generalized Gaussian eXtension (SGGX) distributions within a Level of Detail (LoD) block, preserving high-frequency orientation details better than naive averaging or single-representative fitting [S1]. It solves the problem of accurately representing complex orientation data (normals/tangents) and optical density in lower-resolution volumes by clustering similar distributions rather than collapsing them, outperforming naive aggregation [S1]. Unlike naive averaging which collapses tangents into perpendicular directions and loses accuracy, this method evaluates samples of multiple distributions without averaging, preserving anisotropy [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of SGGX distributions $D_L = \{S_1, \dots, S_L\}$ for a specific LoD level $L$ [S1].
*   **Output:** A reduced set of SGGX distributions $D_{L-1}$ for the next lower resolution level, containing $L-1$ distributions [S1].

**Data Structures:**
*   **SGGX Distribution ($S_i$):** Defined by a rotation matrix $O$ and a diagonal scaling matrix $\Sigma$ (derived from eigenvalues/eigenvectors of the covariance matrix) [S1]. The distribution is parameterized by eigenvectors $(\omega_1, \omega_2, \omega_3)$ and standard deviations $\sigma(\omega_k)$ [S1].
*   **Histogram ($H(S_i)$):** A 3D histogram representing the probability distribution of samples drawn from $S_i$ [S1].
    *   **Bins:** 5 bins in each spatial dimension ($5 \times 5 \times 5 = 125$ bins total) [S1].
*   **Distance Matrix:** A symmetric matrix storing the Wasserstein distance between all pairs of histograms in the current level [S1].

**Parameters:**
*   **Samples per SGGX ($N$):** 5000 samples drawn from each SGGX distribution to generate the histogram [S1].
*   **Maximum distributions per LoD block:** 3 [S1].
*   **Distance Metric:** $\chi$Wasserstein distance (denoted as xWasserstein in sources) [S1].

**Steps:**
1.  **Sampling:** For each SGGX distribution $S_i$ in the current set $D_L$, draw $N=5000$ samples from the distribution [S1]. The sampling process generates uniform samples in the whole sphere, which are projected from the whole space to the underlying ellipsoid to ensure distribution in the whole Normal Distribution Function (NDF) [S1].
2.  **Histogramming:** Bin the $N$ samples for each $S_i$ into a 3D histogram $H(S_i)$ with 5 bins per dimension [S1].
3.  **Distance Calculation:** Compute the distance between every pair of distinct SGGXs $(S_i, S_j)$ using the Wasserstein distance between their histograms:
    $$ \text{dist}(H(S_i), H(S_j)) $$
    Since the distance function is symmetric, only the upper (or lower) triangular part of the distance matrix needs to be computed [S1].
4.  **Pair Selection:** Find the pair $(i, j)$ with $i \neq j$ that minimizes the distance:
    $$ (i, j) = \arg \min_{i \neq j; S_i, S_j \in D_L} \text{dist}(H(S_i), H(S_j)) $$
    [S1]
5.  **Aggregation:** Aggregate the samples of the selected $S_i$ and $S_j$ by concatenating their sample sets [S1].
6.  **Re-fitting:** Fit the resulting histogram of the aggregated samples into a new SGGX distribution $S_n$ using the SGGX Distribution Fitting method [S1].
    *   *Corner Case Handling:* If all input orientations align to almost the exact same orientation, the SGGX becomes ill-defined. In this case, a uniform distribution is used to introduce a small deviation to the original data, yielding a noisier but unbiased SGGX [S1].
7.  **Update Set:** Form the next LoD set $D_{L-1}$ by removing $S_i$ and $S_j$ from $D_L$ and adding $S_n$:
    $$ D_{L-1} = (D_L \setminus \{S_i, S_j\}) \cup \{S_n\} $$
    [S1]
8.  **Iteration:** Repeat steps 1-7 until the desired LoD level is reached or the maximum number of distributions per block (3) is met [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sample_sggx(O, Sigma, n_samples):
    # [G] Standard SGGX sampling: generate uniform points on a sphere,
    # transform by O, scale by Sigma, and normalize to unit length.
    u = np.random.uniform(-1, 1, (n_samples, 3))
    norms = np.linalg.norm(u, axis=1, keepdims=True)
    u = u / norms
    v = u @ O.T
    v = v * Sigma
    v = v / np.linalg.norm(v, axis=1, keepdims=True)
    return v

def compute_histogram(samples, bins_per_dim=5):
    # Normalize samples from [-1, 1] to [0, 1] for binning
    samples_norm = (samples + 1.0) / 2.0
    hist, _ = np.histogramdd(samples_norm, bins=bins_per_dim, range=[[0,1], [0,1], [0,1]])
    return hist.flatten()

def wasserstein_distance_1d(p, q):
    p = p / np.sum(p)
    q = q / np.sum(q)
    cp = np.cumsum(p)
    cq = np.cumsum(q)
    return np.sum(np.abs(cp - cq))

def wasserstein_distance_3d(h1, h2, bins_per_dim=5):
    h1_3d = h1.reshape(bins_per_dim, bins_per_dim, bins_per_dim)
    h2_3d = h2.reshape(bins_per_dim, bins_per_dim, bins_per_dim)
    p1_x = np.sum(h1_3d, axis=(1, 2))
    p2_x = np.sum(h2_3d, axis=(1, 2))
    p1_y = np.sum(h1_3d, axis=(0, 2))
    p2_y = np.sum(h2_3d, axis=(0, 2))
    p1_z = np.sum(h1_3d, axis=(0, 1))
    p2_z = np.sum(h2_3d, axis=(0, 1))
    return (wasserstein_distance_1d(p1_x, p2_x) +
            wasserstein_distance_1d(p1_y, p2_y) +
            wasserstein_distance_1d(p1_z, p2_z))

def fit_sggx(samples):
    mean = np.mean(samples, axis=0)
    centered = samples - mean
    E = np.dot(centered.T, centered) / len(samples)
    eigenvalues, eigenvectors = np.linalg.eigh(E)
    # [G] Handle ill-defined case: if eigenvalues are near zero,
    # add small uniform noise to samples before recomputing covariance.
    sigmas = np.sqrt(eigenvalues)
    O = eigenvectors.T
    Sigma = np.diag(sigmas)
    return O, Sigma

def hierarchical_sggx_aggregation(D_L, max_dists=3, N_samples=5000, bins_per_dim=5):
    current_level = list(D_L)
    while len(current_level) > max_dists:
        n = len(current_level)
        min_dist = float('inf')
        best_pair = None
        histograms = []
        for sggx in current_level:
            samples = sample_sggx(sggx[0], sggx[1], N_samples)
            hist = compute_histogram(samples, bins_per_dim)
            histograms.append(hist)
        for i in range(n):
            for j in range(i + 1, n):
                dist = wasserstein_distance_3d(histograms[i], histograms[j], bins_per_dim)
                if dist < min_dist:
                    min_dist = dist
                    best_pair = (i, j)
        if best_pair is None:
            break
        i, j = best_pair
        samples_i = sample_sggx(current_level[i][0], current_level[i][1], N_samples)
        samples_j = sample_sggx(current_level[j][0], current_level[j][1], N_samples)
        aggregated_samples = np.vstack((samples_i, samples_j))
        new_sggx = fit_sggx(aggregated_samples)
        # Remove j first to preserve index i
        current_level.pop(j)
        current_level.pop(i)
        current_level.append(new_sggx)
    return current_level
```

## Subtopics

### [GPU-CPU Data Flow for SGGX-H](7839-gpu-cpu-data-flow-for-sggx-h.md)
This method defines the data transfer and processing pipeline required to construct the SGGX-H representation from GPU-computed leaf node data. It addresses the problem of converting GPU-resident SGGX distributions into a CPU-accessible hierarchical structure suitable for aggregation. It is distinguished from pure GPU implementations by the explicit requirement to move leaf node histograms to the CPU to enable the generation of aggregated SGGX distributions, a step necessary for the hierarchical clustering logic described in Hierarchical SGGX Aggregation.

### SGGX Distribution Fitting
Fits arbitrary orientation histograms to a single SGGX distribution at the leaf-voxel level to aggregate multiple samples per voxel into a better representation than a single random value [S1]. It extracts eigenvectors $(\omega_1, \omega_2, \omega_3)$ and eigenvalues $\{\lambda_1, \lambda_2, \lambda_3\}$ from the covariance matrix $E$ of the orientation data $X$:
$$ E = \frac{1}{n} \sum_{i=1}^{n} (x_i - \bar{x})(x_i - \bar{x})^\top $$
The eigenvalues correspond to the projected areas (standard deviations) on each eigenvector direction: $\lambda_k = \sigma(\omega_k)$ [S1]. The SGGX matrix $S$ is defined as:
$$ S = (\omega_1, \omega_2, \omega_3) \begin{pmatrix} \sigma(\omega_1) & 0 & 0 \\ 0 & \sigma(\omega_2) & 0 \\ 0 & 0 & \sigma(\omega_3) \end{pmatrix} $$
[S1] A pitfall is that if all input orientations align to almost the exact same orientation, the SGGX becomes ill-defined. This is handled by using a uniform distribution to introduce a small deviation to the original data, yielding a noisier but unbiased SGGX [S1].

## Operating characteristics
**Time and Memory Complexity:**
The method requires computing pairwise distances between all distributions in a block [S1]. Each distance computation involves sampling $N$ points and computing a histogram, followed by a Wasserstein distance calculation [S1]. The method is computationally more expensive than naive averaging due to sampling and histogramming [S1].

**Measured Performance:**
On the Helmet scene at 256x256 resolution, Hierarchical SGGX achieved lower FLIP errors than Naive SGGX at all LoD levels: Level 1 (64 res): 0.098 vs 0.100; Level 2 (32 res): 0.151 vs 0.163; Level 3 (16 res): 0.179 vs 0.201 [S1]. On the Scarf scene (approx 1024 res), Hierarchical SGGX achieved lower FLIP errors than Naive SGGX: Level 1 (approx 800 res): 0.0641 vs 0.0642; Level 2 (approx 400 res): 0.101 vs 0.103; Level 3 (approx 200 res): 0.202 vs 0.281 [S1]. The performance gap increases as resolution decreases, particularly on pixel-wise metrics, suggesting additional benefits when aggregating more orientations [S1].

**Strengths and Limitations:**
*   **Strengths:** Preserves high-frequency details and anisotropy better than naive averaging. Consistently outperforms naive SGGX fitting across all LoD levels [S1].
*   **Limitations:** Requires re-fitting SGGX distributions during aggregation, which may reduce the accuracy of the orientation distribution for that node [S1]. Voxelizing complex fabrics can produce artifacts due to limited per-voxel accuracy with complex geometries in density data [S1].

**When to use:**
Use when accurate representation of complex orientation data (normals/tangents) is required at lower resolutions [S1].

**When not to use:**
Avoid when real-time performance is critical and naive averaging provides sufficient visual quality, or when the input data is simple enough that a single representative SGGX is adequate [S1].

## Background
Hierarchical SGGX Aggregation was introduced in the paper "Fast Voxelization and Level of Detail for Microgeometry Rendering" [S1]. It builds on the SGGX-based Downsampling method introduced by Heitz et al. [HDCD15] [S1]. The method draws inspiration from classical unsupervised learning algorithms in machine learning, specifically hierarchical clustering, to build a dendrogram of LoDs [S1]. The method's evaluation employs a CPU-based volumetric path tracer using Multiple Importance Sampling (MIS) and Woodcock’s Delta Tracking for transmittance estimation [S1].

## Sources
[S1] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
