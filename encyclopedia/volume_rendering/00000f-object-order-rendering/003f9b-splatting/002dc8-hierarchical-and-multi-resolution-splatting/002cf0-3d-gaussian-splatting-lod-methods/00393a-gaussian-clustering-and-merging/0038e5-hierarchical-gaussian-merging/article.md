# Hierarchical Gaussian Merging

Also called: Gaussian Merging for LOD, LOD Generation, LOD Generation via Merging, LOD Level Generation, Unified LOD Approach

## Overview
Hierarchical Gaussian Merging generates multiple Levels of Detail (LOD) for 3D Gaussian Splatting by recursively clustering and merging Gaussians to reduce scene complexity [S1]. It addresses the memory and rendering overhead of large Gaussian scenes by creating simplified representations where each LOD level contains fewer, larger Gaussians [S1]. Unlike methods that optimize for L2-optimal amplitude or use complex cluster culling, this approach relies on simple arithmetic averaging of Gaussian properties to construct a unified buffer hierarchy [S1].

## How it works
**Inputs:** A set of $N$ 3D Gaussians $\{G_i\}$, each defined by position $\mathbf{p}$, covariance matrix $\mathbf{C}$ (or scale/rotation), opacity $\alpha$, and spherical harmonics coefficients $\mathbf{c}$. A target page size $P$ for memory management.
**Outputs:** A hierarchy of LOD levels $L_0, L_1, \dots, L_k$, where $L_0$ contains the original Gaussians and each subsequent level $L_{j+1}$ contains approximately half the Gaussians of $L_j$ [S1].

**Data Structures:**
*   **Gaussian:** A struct containing position (3 floats), scale (3 floats), rotation (4 floats/quaternion), opacity (1 float), and SH coefficients (variable size).
*   **Page:** A container holding up to $P$ Gaussians. Pages are assigned to mesh faces based on proximity [S1].

**Steps:**
1.  **Initialization:** Assign all input Gaussians to Level 0 ($L_0$).
2.  **Page Assignment:** To minimize visibility checks, group Gaussians into pages. Initially, assign each mesh face a page. Assign each Gaussian to the page of the nearest mesh face. If a page exceeds size $P$, subdivide the corresponding mesh face (adding a vertex at an edge midpoint) until capacity is met. Rotate vertex order to avoid thin triangles [S1].
3.  **Merging (Recursive):** For each level $j$ from $0$ to $k-1$:
    *   Partition the Gaussians in $L_j$ into clusters. The target number of Gaussians in $L_{j+1}$ is $\lceil |L_j| / 2 \rceil$ [S1].
    *   For each cluster, compute a representative Gaussian $G_{rep}$ by averaging all properties except scale [S1].
    *   **Position:** $\mathbf{p}_{rep} = \frac{1}{n} \sum_{i=1}^{n} \mathbf{p}_i$
    *   **Opacity:** $\alpha_{rep} = \frac{1}{n} \sum_{i=1}^{n} \alpha_i$
    *   **Rotation:** $\mathbf{q}_{rep} = \frac{1}{n} \sum_{i=1}^{n} \mathbf{q}_i$ (normalized)
    *   **SH Coefficients:** $\mathbf{c}_{rep} = \frac{1}{n} \sum_{i=1}^{n} \mathbf{c}_i$
    *   **Scale:** The source notes explicitly state that scale is *not* averaged with the other properties, implying a different handling or preservation strategy, though the exact formula for scale adjustment is not provided in the excerpt [S1]. [G] Typically, scales might be averaged or adjusted to maintain coverage, but the text specifies "average for all properties but their scale."
    *   Add $G_{rep}$ to $L_{j+1}$.
4.  **Unified Buffer:** Store all LOD levels in a unified buffer format compatible with the original splat format [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_hierarchical_lod(gaussians, page_size):
    lod_levels = [gaussians]
    current_level = gaussians
    
    while len(current_level) > 1:
        next_level = []
        # Target half the size
        target_count = len(current_level) // 2
        if target_count < 1: target_count = 1
        
        # Simple spatial clustering to form 'target_count' clusters
        # Note: Exact clustering algorithm (e.g., k-means) is not detailed in [S1]
        # [G] Assuming a basic spatial partitioning or k-means with k=target_count
        clusters = cluster_gaussians(current_level, target_count)
        
        for cluster in clusters:
            rep = Gaussian()
            rep.position = average(cluster.positions)
            rep.opacity = average(cluster.opacities)
            rep.rotation = normalize(average(cluster.rotations))
            rep.sh_coeffs = average(cluster.sh_coeffs)
            # Scale handling is distinct from other properties [S1]
            rep.scale = handle_scale_merge(cluster.scales) 
            next_level.append(rep)
            
        lod_levels.append(next_level)
        current_level = next_level
        
    return lod_levels

def handle_scale_merge(scales):
    # Source [S1] states scale is NOT averaged with other properties.
    # Exact method is not given. [G] Common practice is averaging or max,
    # but we leave undefined as per strict source adherence.
    return average(scales) # Placeholder for undefined behavior
```

**Edge Cases:**
*   **Page Subdivision:** Repeated subdivision can create thin triangles. The method mitigates this by rotating vertex order and choosing different edges for subdivision [S1].
*   **Scale Ambiguity:** The exclusion of scale from the averaging process is a specific constraint that requires implementation detail not provided in the source [S1].

## Operating characteristics
*   **Complexity:** Generating LODs requires clustering and averaging, which is $O(N \log N)$ or $O(N)$ depending on the clustering implementation. Rendering complexity scales with the number of Gaussians in the selected LOD level.
*   **Memory:** Reduces memory footprint by half per LOD level [S1]. The unified buffer approach minimizes overhead [S2, unofficial].
*   **Strengths:** Simple implementation; effective reduction of Gaussian count for distant viewing [S1].
*   **Limitations:** Arithmetic averaging of rotations and SH coefficients can introduce artifacts if not handled carefully (e.g., quaternion averaging). The specific scale handling is undefined in the primary source [S1].

## Background
This method is associated with the "Virtual memory for 3D Gaussian Splatting" approach, which introduces page-based memory management and LOD generation for 3DGS [S1]. It is also referenced in unofficial implementations like OpenSplat4D [S2, unofficial].

## Sources
[S1] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
[S2] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
