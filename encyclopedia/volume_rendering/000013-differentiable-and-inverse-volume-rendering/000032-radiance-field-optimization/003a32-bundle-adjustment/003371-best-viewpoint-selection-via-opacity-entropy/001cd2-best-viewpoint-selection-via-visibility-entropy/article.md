# Best Viewpoint Selection via Visibility Entropy

Also called: Viewpoint Entropy (VE), Visibility Channel, Visibility Entropy

## Overview
Best Viewpoint Selection via Visibility Entropy (VE) is a method for selecting representative camera viewpoints in volume visualization by maximizing the uniformity of the visibility distribution across the volume's voxels [S1, unofficial]. It models the relationship between viewpoints and volume data as an information channel $V \rightarrow Z$, where visibility is treated as a normalized contribution to the final image rather than raw opacity accumulation [S1, unofficial]. The measure is robust to any type of discretisation or resolution of the volume data set, and behaves similarly for polygonal data [S1, unofficial].

## How it works
The method models the relationship between viewpoints and volume data as an information channel $V \rightarrow Z$, where $V$ is the set of candidate viewpoints and $Z$ is the set of voxels (or objects) in the volume [S1, unofficial].

**Inputs and Outputs**
*   **Input:** A set of candidate viewpoints $\mathcal{V}$, a volume dataset represented as a set of voxels $\mathcal{Z}$, and a visibility function $vis(z|v)$ for each voxel $z$ and viewpoint $v$ [S1, unofficial].
*   **Output:** The single viewpoint $v^* \in \mathcal{V}$ that maximizes the Viewpoint Entropy $H(Z|v)$ [S1, unofficial].

**Data Structures**
*   **Visibility Matrix:** A matrix or map storing $vis(z|v)$ for all $z \in \mathcal{Z}, v \in \mathcal{V}$ [S1, unofficial].
*   **Probability Distributions:** $p(v)$ for viewpoint importance, $p(z|v)$ for conditional voxel visibility, and $p(z)$ for average voxel visibility across the channel [S1, unofficial].

**Steps**

1.  **Calculate Captured Visibility:** For each viewpoint $v \in \mathcal{V}$, compute the total captured visibility $vis(v)$ by summing the visibility contributions of all voxels:
    $$vis(v) = \sum_{z \in \mathcal{Z}} vis(z|v)$$
    [S1, unofficial]

2.  **Compute Input Probability Distribution:** Calculate the importance $p(v)$ of each viewpoint by normalizing its captured visibility over the sum of captured visibilities of all viewpoints:
    $$p(v) = \frac{vis(v)}{\sum_{v' \in \mathcal{V}} vis(v')}$$
    [S1, unofficial] In [Viola 2006], uniform importance was assigned to each viewpoint [S1, unofficial].

3.  **Compute Conditional Probabilities:** For each viewpoint $v$ and voxel $z$, calculate the transition probability $p(z|v)$:
    $$p(z|v) = \frac{vis(z|v)}{vis(v)}$$
    [S1, unofficial] This ensures the normalization constraint $\sum_{z \in \mathcal{Z}} p(z|v) = 1$ is fulfilled for every $v$ [S1, unofficial].

4.  **Compute Output Probability Distribution (Optional for VE calculation):** Calculate the average visibility $p(z)$ of each voxel across all viewpoints, weighted by viewpoint importance:
    $$p(z) = \sum_{v \in \mathcal{V}} p(v)p(z|v)$$
    [S1, unofficial] This step characterizes the channel but is not strictly required for the VE formula itself.

5.  **Calculate Viewpoint Entropy:** For each viewpoint $v$, compute the entropy of the conditional visibility distribution:
    $$H(Z|v) = - \sum_{z \in \mathcal{Z}} p(z|v) \log p(z|v)$$
    [S1, unofficial]

6.  **Select Best Viewpoint:** Choose the viewpoint $v^*$ with the maximum VE:
    $$v^* = \arg \max_{v \in \mathcal{V}} H(Z|v)$$
    [S1, unofficial]

**Parameters**
*   **Visibility Function $vis(z|v)$:** Defined as the contribution of voxel $z$ to the final image from $v$ [S1, unofficial].
*   **Logarithm Base:** Not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Zero Visibility:** If $vis(z|v) = 0$, then $p(z|v) = 0$ [G]. By standard mathematical convention, $0 \log 0 = 0$ in the entropy sum [G].
*   **Degenerate Viewpoints:** If $vis(v) = 0$ (no voxels visible), $p(z|v)$ is undefined [G]. Such viewpoints should be excluded from $\mathcal{V}$ or handled with a guard [G].
*   **Maximum VE:** Achieved when a viewpoint sees all voxels with equal projected visibility (uniform distribution) [S1, unofficial].
*   **Minimum VE:** Achieved when most of the visibility is captured from only a few voxels (sparse distribution) [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_visibility_entropy(viewpoints, voxels, visibility_function):
    """
    Selects the best viewpoint by maximizing Visibility Entropy.
    
    Args:
        viewpoints: List of viewpoint objects/indices.
        voxels: List of voxel objects/indices.
        visibility_function: Callable(vis_z, vis_v) -> float, 
                             returns vis(z|v), the contribution of voxel z to image from v.
                             
    Returns:
        The best viewpoint index.
    """
    # Step 1: Calculate captured visibility vis(v) for each viewpoint
    vis_v = {}
    for v in viewpoints:
        total_vis = 0.0
        for z in voxels:
            total_vis += visibility_function(z, v)
        vis_v[v] = total_vis
        
    # Step 2: Compute input probability distribution p(v)
    total_vis_all = sum(vis_v.values())
    if total_vis_all == 0:
        return viewpoints[0] # [G] Arbitrary fallback for degenerate case
        
    p_v = {}
    for v in viewpoints:
        p_v[v] = vis_v[v] / total_vis_all
        
    # Step 5: Calculate VE for each viewpoint and find maximum
    best_v = None
    max_ve = -math.inf
    
    for v in viewpoints:
        current_vis_v = vis_v[v]
        if current_vis_v == 0:
            continue # [G] Skip degenerate viewpoints where p(z|v) is undefined
            
        ve = 0.0
        for z in voxels:
            vis_z_v = visibility_function(z, v)
            # Step 3: Conditional probability p(z|v)
            p_z_v = vis_z_v / current_vis_v
            
            # Step 5: Entropy term -p(z|v) * log(p(z|v))
            if p_z_v > 0:
                ve -= p_z_v * math.log(p_z_v)
                
        if ve > max_ve:
            max_ve = ve
            best_v = v
            
    return best_v
```

## Subtopics

### Importance-Driven Strategy
A strategy presented by Viola et al. to emphasize objects of interest within the contextual volume, exploiting intelligent viewpoint selection and alternative visual representations of the volume [S2].

## Operating characteristics
*   **Scaling:** Requires computing visibility contributions for all voxel-viewpoint pairs, scaling linearly with $|\mathcal{V}| \cdot |\mathcal{Z}|$ [S1, unofficial].
*   **Strengths:** Robust to any type of discretisation or resolution of the volume data set, and behaves similarly for polygonal data [S1, unofficial].
*   **Time complexity:** Not given in the sources [S1, unofficial].
*   **Memory complexity:** Not given in the sources [S1, unofficial].
*   **When to use:** Not given in the sources [S1, unofficial].
*   **When not to use:** Not given in the sources [S1, unofficial].

## Background
The concept of the visibility channel and viewpoint entropy was introduced in the context of volume visualization to address the problem of selecting the most representative views of a volume dataset [S1, unofficial]. The specific formulation of Viewpoint Entropy (VE) and its relationship to Viewpoint Mutual Information (VMI) is detailed in works by González [2008] and Feixas [2009] [S1, unofficial]. Viola [2006] originally proposed using mutual information for view selection, with uniform viewpoint importance [S1, unofficial]. The framework is applicable to both voxel-based and polygonal data [S1, unofficial].

## Sources
[S1] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
