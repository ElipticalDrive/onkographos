# Geometric Consistency Metric

Also called: Normalized symmetric nearest-surface distance

## Overview
The Geometric Consistency Metric evaluates the geometric consistency between a reconstructed surface and a reference surface, serving as a scale-normalized measure of shape fidelity rather than an absolute metrological error [S1]. It differs from standard Chamfer Distance by normalizing the bidirectional nearest-surface distances by the reference bounding box diagonal, making it robust to scale variations and suitable for comparing objects of different sizes [S1].

## How it works
The method computes a symmetric, normalized error between two point sets sampled from the reconstructed surface ($P$) and the reference surface ($Q$).

**Inputs:**
*   Reconstructed surface mesh.
*   Reference surface mesh.
*   Number of sample points per surface ($N = 100,000$) [S1].

**Outputs:**
*   Geometric consistency error $E_{geo}$ [S1].

**Data Structures:**
*   $P$: Set of $N$ uniformly sampled points from the reconstructed surface.
*   $Q$: Set of $N$ uniformly sampled points from the reference surface.
*   $D$: Diagonal length of the reference surface's bounding box.

**Steps:**
1.  **Sampling:** Uniformly sample $N$ points from the reconstructed surface to form set $P$, and $N$ points from the reference surface to form set $Q$ [S1].
2.  **Alignment:** Align $P$ to $Q$ using Iterative Closest Point (ICP) registration [S1].
    *   Maximum correspondence distance: $0.01 \times D$ (1% of the reference bounding-box diagonal) [S1].
    *   Convergence tolerance: $10^{-6}$ [S1].
    *   Maximum iterations: 50 [S1].
3.  **Distance Calculation:** Compute the bidirectional nearest-surface distance sum [S1]:
    $$ E_{geo} = \frac{1}{2D} \left[ \frac{1}{|P|} \sum_{p \in P} \min_{q \in Q} \|p - q\|^2 + \frac{1}{|Q|} \sum_{q \in Q} \min_{p \in P} \|q - p\|^2 \right] $$
    Where $\| \cdot \|$ denotes the Euclidean distance [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometric_consistency_metric(reconstructed_mesh, reference_mesh, N=100000):
    # 1. Sample points
    P = uniform_sample_points(reconstructed_mesh, N)
    Q = uniform_sample_points(reference_mesh, N)
    
    # 2. Compute reference bounding box diagonal
    ref_bbox = compute_bounding_box(reference_mesh)
    D = compute_diagonal_length(ref_bbox)
    
    # 3. ICP Alignment
    # Parameters from [S1]
    max_corr_dist = 0.01 * D
    tol = 1e-6
    max_iter = 50
    
    # Align P to Q using ICP
    # Note: ICP implementation details are not provided in [S1], 
    # standard ICP logic is assumed [G].
    P_aligned = icp_register(P, Q, 
                             max_correspondence_distance=max_corr_dist,
                             convergence_tolerance=tol,
                             max_iterations=max_iter)
    
    # 4. Compute bidirectional nearest neighbor distances
    # Forward: min distance from each p in P to any q in Q
    dist_p_to_q = sum(min_distance(p, Q) for p in P_aligned) / len(P_aligned)
    
    # Backward: min distance from each q in Q to any p in P
    dist_q_to_p = sum(min_distance(q, P_aligned) for q in Q) / len(Q)
    
    # 5. Normalize and return
    E_geo = (dist_p_to_q + dist_q_to_p) / (2 * D)
    return E_geo

def min_distance(point, point_set):
    return min(sqrt(sum((point - q)**2 for q in point_set)))
```

**Edge Cases and Pitfalls:**
*   **Reference Uncertainty:** The reference surfaces are generated via photogrammetric alignment and dense surface fusion, not laser scanning, so they may retain photogrammetric uncertainty [S1]. The metric measures relative consistency, not absolute truth [S1].
*   **Scale Sensitivity:** Without normalization by $D$, the metric would be scale-dependent [S1].
*   **Alignment Dependency:** The result depends on the success of ICP registration; poor initialization or large deformations may lead to incorrect alignment [S1].

## Subtopics

## Operating characteristics
*   **Complexity:** The complexity is dominated by the ICP registration and nearest-neighbor searches, which are $O(N \log N)$ or $O(N^2)$ depending on the search structure, where $N=100,000$ [S1].
*   **Performance:** Not given in the sources.
*   **Strengths:** Scale-normalized; robust to global scale differences; provides a single scalar consistency measure [S1].
*   **Limitations:** Relies on accurate surface reconstruction and sampling; sensitive to ICP convergence; reference data may contain errors [S1].
*   **When to use:** When comparing geometric fidelity of reconstructed surfaces across different scales or when absolute metrological accuracy is not the primary concern [S1].

## Background
The metric was introduced in the context of evaluating high-fidelity 3D reconstruction and interactive visualization of public art sculptures using neural radiance fields [S1]. It was used to examine differences between explicit surface reconstruction, depth-based reprojection, and implicit volumetric representation [S1]. The reference surfaces were generated through quality-checked multiview alignment and dense surface fusion [S1].

## Sources
[S1] High fidelity three dimensional reconstruction and interactive visualization of public art sculptures using neural radiance fields, https://link.springer.com/article/10.1007/s44163-026-02441-3, fetched 2026-10-26
