# Hierarchical Gaussian Merging

Also called: Gaussian Merging for LOD, LOD Generation, LOD Generation via Merging, LOD Level Generation, Unified LOD Approach

## Overview
Hierarchical Gaussian Merging generates Level-of-Detail (LOD) representations for 3D Gaussian Splatting by recursively halving the number of Gaussians per page to reduce memory bandwidth and rendering overhead at distance [S1]. It differs from sibling methods by using a simple arithmetic average for most properties (excluding scale) within a unified buffer structure, rather than complex moment-matching or L2-optimal amplitude optimization [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of original Gaussians $\{G_i\}_{i=1}^N$ organized into pages, where each page has a maximum capacity $P$.
*   **Output:** A hierarchy of LOD levels $L_0, L_1, \dots, L_k$, where $L_0$ contains the original Gaussians and subsequent levels contain merged Gaussians.

**Data Structures:**
Each Gaussian $G$ is stored with fields: position $\mathbf{t}$, covariance $\mathbf{\Sigma}$ (or rotation/scale), opacity $\alpha$, and color/amplitude coefficients $c$. The "Unified LOD Approach" implies that LOD splats use the same memory format as original splats [S2, unofficial].

**Steps:**
1.  **Page Assignment:** Assign each Gaussian to a page based on the nearest mesh face [S1]. If a face exceeds the page size $P$, subdivide the face by adding a vertex at an edge’s midpoint, repeating until capacity is met. Rotate vertex order to prevent thin triangles [S1].
2.  **LOD Level 0:** Store the original Gaussians in $L_0$ [S1].
3.  **Recursive Merging:** For each subsequent level $L_{j+1}$ from $L_j$:
    a. Partition the Gaussians in $L_j$ into clusters. The target number of Gaussians per page in $L_{j+1}$ is $\lfloor P / 2 \rfloor$ [S1].
    b. For each cluster, compute a single representative Gaussian $G_{merged}$ [S1].
    c. The parameters for $G_{merged}$ are derived as follows:
        *   Position $\mathbf{t}_{merged}$: Arithmetic average of cluster positions.
        *   Opacity $\alpha_{merged}$: Arithmetic average of cluster opacities.
        *   Color $c_{merged}$: Arithmetic average of cluster colors.
        *   Scale/Covariance: Not averaged; handled separately (exact method not given in sources, but noted as an exception to the averaging rule) [S1].
4.  **Repeat:** Continue until the desired number of levels is reached or the minimum Gaussian count is met [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_hierarchical_lod(gaussians, page_size):
    """
    Generates LOD levels by halving page size and averaging properties.
    gaussians: List of Gaussian objects with attributes t, alpha, c, scale.
    page_size: Maximum Gaussians per page for Level 0.
    """
    lod_levels = []
    current_gaussians = gaussians
    current_page_size = page_size
    
    # Level 0 is the original set
    lod_levels.append(current_gaussians)
    
    while current_page_size > 1:
        next_page_size = current_page_size // 2
        next_gaussians = []
        
        # Group into clusters of size next_page_size
        # Note: Source [S1] implies halving per page, so we cluster 
        # to fit the new smaller page size.
        for i in range(0, len(current_gaussians), next_page_size):
            cluster = current_gaussians[i : i + next_page_size]
            if not cluster:
                break
                
            # Merge cluster
            merged = merge_gaussians(cluster)
            next_gaussians.append(merged)
            
        lod_levels.append(next_gaussians)
        current_gaussians = next_gaussians
        current_page_size = next_page_size
        
    return lod_levels

def merge_gaussians(cluster):
    """
    Merges a cluster of Gaussians using arithmetic average for 
    position, opacity, and color. Scale handling is not specified 
    in sources [S1].
    """
    if not cluster:
        return None
        
    # Arithmetic average for position
    t_avg = sum(g.t for g in cluster) / len(cluster)
    
    # Arithmetic average for opacity
    alpha_avg = sum(g.alpha for g in cluster) / len(cluster)
    
    # Arithmetic average for color
    c_avg = sum(g.c for g in cluster) / len(cluster)
    
    # Scale/Covariance: Not given in sources [S1]
    # [G] Typically, scale might be preserved from the largest member 
    # or averaged in log-space, but this is implementation-specific.
    scale_avg = cluster[0].scale 
    
    return Gaussian(t=t_avg, alpha=alpha_avg, c=c_avg, scale=scale_avg)
```

**Edge Cases and Pitfalls:**
*   **Scale Handling:** The sources explicitly state that scale is *not* averaged like other properties [S1], but do not provide the formula for how scale is derived during merging. This is a critical implementation gap [G].
*   **Page Subdivision:** Repeated subdivision can create thin triangles; rotating vertex order is required to mitigate this [S1].
*   **Precision:** Arithmetic averaging of opacity and color may lead to visual artifacts if clusters contain high-contrast features.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians per level, as it involves simple averaging.
*   **Memory Complexity:** Reduces memory footprint by half at each LOD level [S1].
*   **Strengths:** Simple to implement; unified buffer format simplifies memory management [S2, unofficial].
*   **Limitations:** Lack of scale averaging detail; simple averaging may not preserve visual fidelity as well as moment-matched methods [S1][S2, unofficial].

## Background
This method is associated with the "Virtual memory for 3D Gaussian Splatting" approach, which uses page-based memory management and LOD generation to optimize rendering [S1]. It is also referenced in unofficial implementations like OpenSplat4D [S2, unofficial].

## Sources
[S1] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
[S2] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
