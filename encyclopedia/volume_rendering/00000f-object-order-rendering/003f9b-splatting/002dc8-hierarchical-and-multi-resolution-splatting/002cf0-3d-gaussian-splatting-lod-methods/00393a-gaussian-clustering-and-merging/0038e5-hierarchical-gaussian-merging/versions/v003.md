# Hierarchical Gaussian Merging

Also called: Gaussian Merging for LOD, LOD Generation, LOD Generation via Merging, LOD Level Generation, Unified LOD Approach

## Overview
Hierarchical Gaussian Merging generates multiple levels of detail (LOD) for 3D Gaussian Splatting by iteratively reducing the number of Gaussians in each cluster. It solves the problem of rendering large Gaussian datasets efficiently by allowing the renderer to select a coarser representation based on distance or performance constraints [S1]. Unlike methods that optimize merged parameters via L2 error minimization or moment matching, this approach relies on simple arithmetic averaging of Gaussian properties to create simplified proxies [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ Gaussians $\{G_i\}_{i=1}^N$ organized into pages/clusters, where each Gaussian $G_i$ has position $\mathbf{t}_i$, covariance $\mathbf{\Sigma}_i$ (or scale/rotation), opacity $\alpha_i$, and color coefficients $c_i$.
*   **Output:** A hierarchy of LOD levels $L_0, L_1, \dots, L_k$, where $L_0$ contains the original Gaussians and each subsequent level $L_{j+1}$ contains approximately half the number of Gaussians as $L_j$ [S1].

**Data Structures:**
The method uses a "unified buffer approach" where LOD splats use the same memory format as original splats to facilitate easy switching between levels [S2, unofficial]. Each Gaussian is stored with its full attribute set.

**Steps:**
1.  **Initialization:** Assign Level 0 ($L_0$) to be the original set of Gaussians. Group Gaussians into pages based on spatial proximity to mesh faces, subdividing faces if a page exceeds a maximum capacity [S1].
2.  **Iteration:** For each level $j$ from 0 to $k-1$:
    a.  Iterate through each page/cluster $C$ in level $L_j$.
    b.  If the cluster contains more than one Gaussian, merge them into a single representative Gaussian for level $L_{j+1}$.
    c.  Compute the parameters of the merged Gaussian $G_{merged}$ using arithmetic averaging for all properties except scale [S1].
3.  **Scale Handling:** The scale (or covariance matrix) is not averaged. The specific method for determining the scale of the merged Gaussian is not detailed in the sources, though it implies a non-averaged selection or calculation [S1].
4.  **Halving:** The number of Gaussians per page is halved for each subsequent level [S1].

**Formulas:**
For a cluster $C$ of Gaussians, the merged position $\mathbf{t}_m$, opacity $\alpha_m$, and color $c_m$ are computed as simple averages:

$$ \mathbf{t}_{m} = \frac{1}{|C|} \sum_{i \in C} \mathbf{t}_i $$

$$ \alpha_{m} = \frac{1}{|C|} \sum_{i \in C} \alpha_i $$

$$ c_{m} = \frac{1}{|C|} \sum_{i \in C} c_i $$

where $|C|$ is the number of Gaussians in the cluster [S1].

**Parameters:**
*   **Page Size:** The maximum number of Gaussians per page. Not given in the sources.
*   **LOD Depth:** The number of levels to generate. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Scale Ambiguity:** The sources explicitly state that scale is *not* averaged but do not specify the alternative method (e.g., maximum, minimum, or weighted average). Implementers must define this behavior [S1].
*   **Precision:** Simple averaging may lead to visual artifacts if the cluster contains Gaussians with widely varying opacities or colors, as it does not account for visual weight [S1].

## Operating characteristics
*   **Time Complexity:** Generation is linear with respect to the number of Gaussians and LOD levels, as it involves simple summation and division.
*   **Memory Complexity:** Requires storing all LOD levels, increasing memory usage by a factor of $k$ (number of levels). However, the unified buffer format allows efficient streaming [S2, unofficial].
*   **Strengths:** Extremely fast to generate; simple implementation; compatible with standard splatting pipelines [S1].
*   **Limitations:** Visual quality may degrade faster than optimized merging methods because it ignores the spatial distribution and opacity weights of the Gaussians during merging [S1].

## Background
This method was introduced in the context of virtual memory systems for 3D Gaussian Splatting to enable efficient out-of-core rendering [S1]. It is part of a broader effort to manage the memory footprint of large-scale Gaussian datasets by providing hierarchical representations.

## Sources
[S1] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
[S2] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
