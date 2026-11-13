# Hierarchical Gaussian Merging

Also called: Gaussian Merging for LOD, LOD Generation, LOD Generation via Merging, LOD Level Generation, Unified LOD Approach

## Overview
Hierarchical Gaussian Merging generates multiple levels of detail (LOD) for 3D Gaussian Splatting by iteratively reducing the number of Gaussians per page [S1]. It builds on [Gaussian clustering and merging](8436-gaussian-clustering-and-merging.md) by applying arithmetic averaging to create simplified proxies at each level [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ Gaussians $\{G_i\}_{i=1}^N$ organized into pages/clusters, where each Gaussian $G_i$ has position $\mathbf{t}_i$, covariance $\mathbf{\Sigma}_i$ (or scale/rotation), opacity $\alpha_i$, and color coefficients $c_i$ [S1].
*   **Output:** A hierarchy of LOD levels $L_0, L_1, \dots, L_k$, where $L_0$ contains the original Gaussians and each subsequent level $L_{j+1}$ contains approximately half the number of Gaussians per page as $L_j$ [S1].

**Data Structures:**
The method uses a "unified buffer approach" where LOD splats use the same memory format as original splats to facilitate switching between levels [S2, unofficial]. Each Gaussian retains its full attribute set [S1].

**Steps:**
1.  **Initialization:** Assign Level 0 ($L_0$) to be the original set of Gaussians [G]. Group Gaussians into pages based on spatial proximity to mesh faces, subdividing faces if a page exceeds a maximum capacity [S1].
2.  **Iteration:** For each level $j$ from 0 to $k-1$ [G]:
    a.  Iterate through each page/cluster $C$ in level $L_j$ [G].
    b.  If the cluster contains more than one Gaussian, merge them into a single representative Gaussian for level $L_{j+1}$ [S1].
    c.  Compute the parameters of the merged Gaussian $G_{merged}$ using arithmetic averaging for all properties except scale [S1].
3.  **Scale Handling:** The scale (or covariance matrix) is not averaged [S1]. The specific method for determining the scale of the merged Gaussian is not detailed in the sources [S1].
4.  **Halving:** The number of Gaussians per page is halved for each subsequent level [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_lod_levels(gaussians, max_page_size, num_levels):
    levels = [gaussians]
    current_page_size = max_page_size
    for level in range(1, num_levels):
        next_level = []
        pages = [gaussians[i:i + current_page_size] for i in range(0, len(gaussians), current_page_size)]
        for page in pages:
            if len(page) == 0:
                continue
            merged_t = sum(g.t for g in page) / len(page)
            merged_alpha = sum(g.alpha for g in page) / len(page)
            merged_c = sum(g.c for g in page) / len(page)
            # Scale is not averaged; exact fallback not specified in sources [S1]
            merged_scale = page[0].scale
            next_level.append(Gaussian(merged_t, merged_scale, merged_alpha, merged_c))
        levels.append(next_level)
        gaussians = next_level
        current_page_size = current_page_size // 2
    return levels
```

**Formulas:**
For a cluster $C$ of Gaussians, the merged position $\mathbf{t}_{m}$, opacity $\alpha_{m}$, and color $c_{m}$ are computed as simple averages:

$$ \mathbf{t}_{m} = \frac{1}{|C|} \sum_{i \in C} \mathbf{t}_i $$

$$ \alpha_{m} = \frac{1}{|C|} \sum_{i \in C} \alpha_i $$

$$ c_{m} = \frac{1}{|C|} \sum_{i \in C} c_i $$

where $|C|$ is the number of Gaussians in the cluster [S1].

**Parameters:**
*   **Page Size:** The maximum number of Gaussians per page. Not given in the sources.
*   **LOD Depth:** The number of levels to generate. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Scale Ambiguity:** The sources explicitly state that scale is *not* averaged but do not specify the alternative method (e.g., maximum, minimum, or weighted average) [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Requires storing all LOD levels, increasing memory usage by a factor of $k$ (number of levels) [S1].
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.

## Background
The method is described in the context of page assignment for Gaussian datasets, where mesh faces are subdivided to manage page capacity [S1].

## Sources
[S1] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
[S2] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
