# LOD Generation for 3DGS

Also called: 3DGS LOD Generation, Gaussian Clustering for LOD

## Overview
This method reduces the number of 3D Gaussians copied, stored, sorted, and rendered by merging Gaussians into clusters based on similar attributes, minimizing distant appearance impact without altering scene creation [S1]. It operates on a virtual memory system where Gaussians are organized into pages; unlike virtual texturing where mip levels are uniform-sized pages, this method reduces the Gaussian count with each LOD level, allowing multiple virtual pages to reside within one physical page [S1]. It is distinct from [K-wise moment-matched merging](5440-k-wise-moment-matched-merging-with-l2-optimal-amplitude.md) by using arithmetic averaging and fixed scaling factors rather than L2-optimal moment matching, and from [Hierarchical Gaussian Merging](8358-hierarchical-gaussian-merging.md) by its specific integration with page-based virtual memory management and dynamic thresholding [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $G$ assigned to spatial pages. Each Gaussian $G_i$ has position $\mathbf{t}_i$, rotation $\mathbf{q}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and spherical harmonic coefficients $c_i$ [G].
*   **Output:** A set of merged representative Gaussians $G'$ for each LOD level, and a page table mapping virtual pages to physical memory with associated LOD indices [G].

**Data Structures:**
*   **Page Table:** An array of entries. Each entry contains:
    *   `page_id`: Identifier for the virtual page.
    *   `lod_level`: Integer index indicating the current detail level (0 = highest detail/raw Gaussians, 1+ = merged) [S1].
    *   `last_use`: Timestamp or frame counter for LRU eviction [S1].
    *   `physical_address`: Pointer to the GPU memory buffer containing the merged Gaussians for this LOD [S1].
*   **Gaussian Buffer:** Stores the merged Gaussians [G]. Since the count decreases with LOD, buffers for higher LODs are smaller [S1].

**Algorithm Steps:**

1.  **Clustering:** For each page, identify clusters of Gaussians using the k-means algorithm in a multidimensional attribute space [S1].
    *   The feature vector for each Gaussian includes position $\mathbf{t}_i$, rotation $\mathbf{q}_i$, and color $c_i$ [G].
    *   **Priority:** Position attributes are prioritized during clustering to prevent dispersed clusters [S1]. This is implemented by weighting the position components higher in the distance metric used by k-means [G].
    *   **Constraint:** Only Gaussians residing on the same page can be merged into a cluster [S1].

2.  **Merging:** Replace each cluster $C_j$ with a single representative Gaussian $G'_j$ [S1].
    *   **Translation:** Arithmetic mean: $\mathbf{t}'_j = \frac{1}{|C_j|} \sum_{i \in C_j} \mathbf{t}_i$ [S1].
    *   **Rotation:** Arithmetic mean of quaternions (normalized) [S1].
    *   **Opacity:** Arithmetic mean: $\alpha'_j = \frac{1}{|C_j|} \sum_{i \in C_j} \alpha_i$ [S1].
    *   **Color:** Arithmetic mean of spherical harmonic coefficients [S1].
    *   **Scale:** Arithmetic mean is **not** used directly. Merging scales directly causes the resulting Gaussian to be too small, creating visual gaps [S1]. Instead, the merged Gaussian is scaled up by a fixed factor $k_{scale}$ to fill gaps [S1]. The factor $k_{scale}$ is determined during training using average pixel coverage [S1].
        *   $\mathbf{s}'_j = k_{scale} \cdot \frac{1}{|C_j|} \sum_{i \in C_j} \mathbf{s}_i$ [S1].

3.  **LOD Threshold Management:**
    *   Assign an initial LOD level to each page based on distance from the camera [G].
    *   **Dynamic Adjustment:** Thresholds move away from the camera when GPU memory usage exceeds a limit, and closer when memory is available [S1].
    *   **Step Size Adaptation:** To prevent rapid flipping (popping/flickering) between levels, the step size for threshold changes is dynamically adjusted based on previous moves [S1]. If a move caused overshooting (drastic memory change), the step size is reduced; if adaptation is too slow, it is increased [S1].

4.  **Page Table Update:** Mark page table entries with their current LOD level [S1]. When a page's LOD changes, the corresponding merged Gaussian buffer is loaded or evicted [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from math import sqrt, pi

class Gaussian:
    t: tuple[float, float, float]
    q: tuple[float, float, float, float]
    s: tuple[float, float, float]
    alpha: float
    sh: tuple[float, float, float]
    color: tuple[float, float, float]

def normalize_quaternion(q: tuple[float, float, float, float]) -> tuple[float, float, float, float]:
    norm = sqrt(sum(c*c for c in q))
    return tuple(c / norm for c in q)

def compute_cluster_mean(cluster: list[Gaussian]) -> Gaussian:
    n = len(cluster)
    sum_t = (0.0, 0.0, 0.0)
    sum_q = (1.0, 0.0, 0.0, 0.0)
    sum_alpha = 0.0
    sum_sh = (0.0, 0.0, 0.0)
    sum_s = (0.0, 0.0, 0.0)
    for g in cluster:
        sum_t = tuple(sum_t[i] + g.t[i] for i in range(3))
        sum_q = tuple(sum_q[i] + g.q[i] for i in range(4))
        sum_alpha += g.alpha
        sum_sh = tuple(sum_sh[i] + g.sh[i] for i in range(3))
        sum_s = tuple(sum_s[i] + g.s[i] for i in range(3))
    mean_t = tuple(v / n for v in sum_t)
    mean_q = normalize_quaternion(tuple(v / n for v in sum_q))
    mean_alpha = sum_alpha / n
    mean_sh = tuple(v / n for v in sum_sh)
    raw_mean_s = tuple(v / n for v in sum_s)
    return Gaussian(mean_t, mean_q, raw_mean_s, mean_alpha, mean_sh, (0.0, 0.0, 0.0))

def standard_kmeans(points: list, k: int, max_iters: int = 10) -> list[list[int]]:
    """Standard Lloyd's k-means algorithm [G]. Returns list of cluster indices."""
    n = len(points)
    centroids = points[:k]
    assignments = [0] * n
    for _ in range(max_iters):
        new_assignments = []
        for p in points:
            min_dist = float('inf')
            best_c = 0
            for i, c in enumerate(centroids):
                dist = sum((p[j] - c[j])**2 for j in range(len(p)))
                if dist < min_dist:
                    min_dist = dist
                    best_c = i
            new_assignments.append(best_c)
        if new_assignments == assignments:
            break
        assignments = new_assignments
        for i in range(k):
            cluster_pts = [points[j] for j in range(n) if assignments[j] == i]
            if cluster_pts:
                centroids[i] = tuple(sum(pts[d] for pts in cluster_pts) / len(cluster_pts) for d in range(len(points[0])))
    clusters = [[] for _ in range(k)]
    for idx, c in enumerate(assignments):
        clusters[c].append(idx)
    return clusters

def generate_lod_gaussians(page_gaussians: list[Gaussian], cluster_count: int, scale_factor: float) -> list[Gaussian]:
    features = [tuple(g.t + g.q + g.color) for g in page_gaussians]
    clusters = standard_kmeans(features, k=cluster_count)
    merged_gaussians = []
    for cluster_indices in clusters:
        cluster_gs = [page_gaussians[i] for i in cluster_indices]
        t, q, alpha, sh, raw_s = compute_cluster_mean(cluster_gs).t, compute_cluster_mean(cluster_gs).q, compute_cluster_mean(cluster_gs).alpha, compute_cluster_mean(cluster_gs).sh, compute_cluster_mean(cluster_gs).s
        s = tuple(v * scale_factor for v in raw_s)
        merged_gaussians.append(Gaussian(t, q, s, alpha, sh, (0.0, 0.0, 0.0)))
    return merged_gaussians

def update_lod_thresholds(current_memory_usage: float, memory_limit: float, current_thresholds: list[float], prev_step_direction: int) -> tuple[list[float], int]:
    new_thresholds = current_thresholds.copy()
    step = 1.0
    if prev_step_direction != 0:
        step = 0.8 if prev_step_direction == 1 else 1.2
    if current_memory_usage > memory_limit:
        for i in range(len(new_thresholds)):
            new_thresholds[i] += step
        return new_thresholds, 1
    else:
        for i in range(len(new_thresholds)):
            new_thresholds[i] = max(0.0, new_thresholds[i] - step)
        return new_thresholds, -1
```

## Subtopics

### Memory-Efficient 3D GS via Pruning and Clustering
This variant combines strategic pruning of low-impact Gaussians (e.g., volume-based masking) with clustering [S2]. Neighboring Gaussians are represented using properties stored within a "local anchor" obtained via clustering [S2]. It specifically employs a hash-grid for clustering to improve efficiency in large-scale scenes [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Enables real-time rendering (60+ FPS) by reducing sorting and rendering overhead for distant geometry [S1].
*   **Strengths:** Arithmetic averaging is computationally cheap [S1]. Dynamic thresholding adapts to varying memory constraints [S1].
*   **Limitations:** Arithmetic averaging can introduce geometric artifacts if clusters are large or heterogeneous [S1]. Fixed scale factor may not be optimal for all scenes [S1]. Rapid flipping (popping) can occur if threshold adaptation is poorly tuned [S1].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
This method was introduced in "Virtual memory for 3D Gaussian Splatting" [S1]. It addresses the scalability and memory demands of 3DGS in large-scale scenes [S2]. The approach is inspired by virtual texturing techniques used in computer graphics, where mip levels are used to manage memory bandwidth and capacity [S1]. The specific use of k-means clustering with position prioritization and fixed scale adjustment was proposed to minimize distant appearance impact while keeping the implementation simple [S1].

## Sources
[S1] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
[S2] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
