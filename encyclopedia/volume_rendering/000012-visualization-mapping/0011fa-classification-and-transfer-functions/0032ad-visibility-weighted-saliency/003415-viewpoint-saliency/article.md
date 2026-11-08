# Viewpoint Saliency

Also called: Saliency-based View Selection, Visibility-Weighted Saliency Metric

## Overview
Viewpoint Saliency is a metric used to select optimal viewpoints for volume visualization by maximizing the visible perceptual importance of the data [S1]. It builds on [Visibility-Weighted Saliency](6824-visibility-weighted-saliency.md) by aggregating voxel-level saliency values weighted by their geometric visibility from a candidate viewpoint [S1]. Unlike methods that rely solely on geometric complexity or camera distance, this approach prioritizes views where the most semantically or structurally significant features are unoccluded [S1].

## How it works
**Inputs:**
1.  A 3D volume dataset $V$.
2.  A precomputed voxel-wise saliency map $S(p)$ for each voxel $p$ in the volume [S1].
3.  A set of candidate viewpoints $v$ [S1].

**Outputs:**
1.  The viewpoint $v^*$ that maximizes the viewpoint saliency score [S1].

**Data Structures:**
*   **Saliency Map:** A 3D array (or sparse structure) where each element corresponds to a voxel $p$ and stores a scalar saliency value $S(p)$. The values are typically normalized or derived from local geometric/structural properties [S1].
*   **Visibility Map:** For a given viewpoint $v$, a 3D array (or sparse structure) where each element corresponds to a voxel $p$ and stores a binary or continuous visibility value $V(p)$ [S1].

**Steps:**
1.  **Compute Visibility:** For each candidate viewpoint $v$, determine the visibility $V(p)$ of every voxel $p$ in the volume. $V(p)$ is typically 1 if the voxel is visible (not occluded by other voxels along the ray from the camera through $p$) and 0 otherwise, or a continuous value representing the fraction of the voxel's contribution [S1].
2.  **Calculate Viewpoint Saliency:** For each viewpoint $v$, compute the total viewpoint saliency $S(v)$ using the formula:
    $$S(v) = \sum_{p \in P} S(p)V(p)$$
    where $P$ is the set of all voxels in the volume data [S1].
3.  **Select Optimal Viewpoint:** Choose the viewpoint $v^*$ that yields the maximum $S(v)$:
    $$v^* = \arg\max_{v} S(v)$$
    [S1]

**Parameters:**
*   **Saliency Computation Method:** The method used to generate $S(p)$ (e.g., based on local structure, intensity gradients, or mesh saliency proxies) is not specified in the core definition but is assumed to be precomputed [S1].
*   **Visibility Calculation:** The specific ray-casting or depth-buffering technique used to determine $V(p)$ is not detailed, but must account for occlusion [S1].

**Edge Cases and Pitfalls:**
*   **Occlusion:** If the most salient voxels are heavily occluded from all candidate viewpoints, the metric may select a view that shows less salient but more visible features [S1].
*   **Saliency Accuracy:** The quality of the result depends entirely on the accuracy of the precomputed voxel saliency $S(p)$ [S1].
*   **Computational Cost:** Summing over all voxels for every viewpoint can be expensive for large volumes and many viewpoints [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_viewpoint_saliency(volume_data, voxel_saliency_map, candidate_viewpoints):
    """
    Computes the viewpoint saliency for a set of candidate viewpoints.
    
    Args:
        volume_data: The 3D volume grid.
        voxel_saliency_map: 3D array of saliency values S(p) for each voxel p.
        candidate_viewpoints: List of camera viewpoints v.
        
    Returns:
        best_viewpoint: The viewpoint with the highest saliency score.
        scores: Dictionary mapping viewpoint to its saliency score.
    """
    scores = {}
    
    for v in candidate_viewpoints:
        total_saliency = 0.0
        
        # Determine visibility for all voxels from viewpoint v
        # V(p) is 1 if voxel p is visible from v, 0 otherwise
        # This typically involves ray-casting or depth buffer lookup
        visibility_map = compute_visibility(volume_data, v)
        
        # Sum the product of voxel saliency and visibility
        for p in volume_data.get_all_voxels():
            s_p = voxel_saliency_map[p]
            v_p = visibility_map[p]
            total_saliency += s_p * v_p
            
        scores[v] = total_saliency
        
    best_viewpoint = max(scores, key=scores.get)
    return best_viewpoint, scores

def compute_visibility(volume_data, viewpoint):
    """
    Computes the visibility of each voxel from a given viewpoint.
    Returns a 3D array where 1 indicates visible, 0 indicates occluded.
    Implementation detail: Standard ray-casting or depth-buffering.
    """
    # [G] Placeholder for specific visibility algorithm (e.g., ray-marching, depth buffer)
    # The core method assumes this function returns V(p) for all p.
    pass
```

## Subtopics

### Information-Theoretic Viewpoint Estimation
An algorithm that estimates optimal viewpoints for a set of features using an information-theoretic framework to maximize visual discrimination between focus features and context [S2, unofficial]. It defines a set of candidate viewpoints $(v_1, v_2, ..., v_n)$ and features $(o_1, o_2, ..., o_m)$, computes visibility probabilities $p(o_j|v_i)$ for each feature-viewpoint pair, and calculates an information metric $I(v_i, O)$ for each viewpoint [S2, unofficial]. The viewpoint that maximizes this metric is selected to visually pop out the focus feature while suppressing context [S2, unofficial]. The information metric is defined as:
$$I(v_i, O) = \sum_{j=1}^{m} p(o_j|v_i) \log \frac{p(o_j|v_i)}{p(o_j)}$$
where $p(o_j|v_i)$ is the probability of observing feature $o_j$ from viewpoint $v_i$, and $p(o_j)$ is the prior probability of feature $o_j$ [S2, unofficial]. This approach differs from basic Viewpoint Saliency by explicitly modeling the discrimination between focus and context rather than just maximizing total visible saliency [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of voxels and $V$ is the number of candidate viewpoints, assuming visibility computation is $O(1)$ per voxel per viewpoint (e.g., via precomputed depth buffers) [S1]. If visibility is computed via ray-casting, it may be higher [S1].
*   **Memory Complexity:** $O(N)$ to store the voxel saliency map and visibility maps [S1].
*   **Strengths:** Directly optimizes for perceptual importance; automated; guides viewer attention to significant structures [S1].
*   **Limitations:** Depends on accurate precomputed voxel saliency; may not account for aesthetic or compositional factors beyond saliency [S1].
*   **When to Use:** When the goal is to automatically present the most informative or significant parts of a volume dataset [S1].
*   **When Not to Use:** When specific user-defined regions of interest are more important than global saliency, or when aesthetic composition is prioritized over structural importance [S1].

## Background
The concept of using saliency for view selection in volume visualization was introduced by Viola et al. in 2008 [S1]. It draws inspiration from mesh saliency methods used for 3D mesh view selection [S1]. The method is part of a broader framework for obscurance-based volume rendering and automated visualization enhancement [S1]. It is used in medical visualization and scientific data exploration to guide user attention [S1].

## Sources
[S1] Obscurance-based Volume Rendering Framework, https://doi.org/10.2312/vg/vg-pbg08/113-120, fetched 2026-10-26
[S2] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
