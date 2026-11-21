# Budgeted Gaussian Splatting

## Overview
Budgeted Gaussian Splatting (BCT) is a training and deployment method for large-scale 3D Gaussian Splatting that enables continuous, variable primitive counts (Level of Detail) without retraining [S1, unofficial]. It solves the problem of fixed-budget compression by constructing a Morton-ordered tree of primitives and a coverage-preserving prefix cut, allowing for near-lossless quality at low compression ratios [S1, unofficial]. It distinguishes itself from sibling methods like [Hierarchical 3D Gaussian Splatting (H-3DGS)](10279-hierarchical-3d-gaussian-splatting-h-3dgs.md) and [Iterative Gaussian Synopsis](9289-iterative-gaussian-synopsis.md) by providing a single, global primitive ordering whose prefixes define successive approximations of the scene, rather than camera-dependent hierarchical selectors or discrete hierarchy methods [S1, unofficial].

## How it works
**Inputs:** A trained 3D Gaussian Splatting model (means, covariances/scales, opacities, spherical harmonics coefficients) and a set of training views.
**Outputs:** A compact binary file containing primitives in Morton order, and a runtime scheduler that selects a subset of primitives for rendering based on a budget parameter $c$.

**Data Structures:**
1.  **Morton-Ordered Tree:** Primitives are sorted by their 3D Morton code. The tree structure allows for efficient prefix traversal.
2.  **Compact Binary Format:** Stores raw primitives in Morton order, per-band equivalent planks, and SH first moments of all degrees.
    *   **DC-only Compact Path:** Stores 3 floats per primitive (DC component), enabling selection of up to 13.1M primitives in 3.9 GB [S1, unofficial].
    *   **Full Export:** Standard 3DGS PLY format with SH zero-padded to degree 3, scales/opacities in log/logit domain [S1, unofficial].
3.  **Residuals (for Adaptive Selection):** Per cluster, two residuals are stored:
    *   Geometric spread: $\sigma_{geo} = \frac{p \cdot \text{tr}(T^*)}{W}$ (world units), projected to pixels by $\sigma \cdot f / d$ [S1, unofficial].
    *   SH residual ratio: $\rho_{sh}$ (cluster-internal SH variance vs. global RMS) [S1, unofficial].

**Parameters:**
*   $c$: Compression factor or LOD tier parameter. Typical values: $c \in \{1, 2, 4, 8, 16, 32\}$ [S1, unofficial].
*   $K$: Capacity or maximum primitive count for the trajectory (e.g., $K=2\text{M}$) [S1, unofficial].

**Algorithm Steps:**

1.  **Preprocessing (Offline):**
    1.  Construct a Morton-order tree on the trained primitives.
        *   *Performance:* Takes 1.5 s with 19.4 GiB peak memory on an A100 GPU for 8.56M primitives [S1, unofficial].
    2.  (Optional) Compute LightGaussian significance scores for pure primitive-count budgets:
        *   Score every primitive with accumulated per-pixel $T^*\alpha$ over all training views via a one-pass forward scan [S1, unofficial].
        *   Validate CUDA accumulation using the telescoping identity $\sum_i s_i = \sum_p (1 - T^*_{\text{final}})$, matched to $10^{-5}$ relative [S1, unofficial].
        *   Select top-$N$ primitives based on this score for a given budget [S1, unofficial].

2.  **Runtime Selection (Per Frame):**
    1.  Determine the target budget based on $c$ or $K$.
    2.  Traverse the coverage-preserving prefix cut on the GPU.
    3.  Gather the selected prefix directly into rasterizer buffers.
        *   This step fuses SH evaluation and opacity compensation.
        *   No CPU-side collection or host-to-device upload is required [S1, unofficial].
    4.  (Adaptive Mode) Use $\sigma_{geo}$ and $\rho_{sh}$ to refine selection if content-aware pruning is enabled [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess(primitives, training_views):
    """
    Offline preprocessing to build the Morton tree and compute residuals.
    primitives: List of Gaussian objects (mean, scale, rotation, opacity, SH_coeffs)
    training_views: List of camera poses and intrinsics
    """
    # 1. Compute Morton codes for all primitives
    morton_codes = [compute_morton_code(p.mean) for p in primitives]
    
    # 2. Sort primitives by Morton code
    sorted_indices = argsort(morton_codes)
    sorted_primitives = [primitives[i] for i in sorted_indices]
    
    # 3. Build Morton tree (conceptual: hierarchical grouping based on code prefixes)
    # In practice, this is a sorted array with implicit tree structure for prefix cuts.
    tree = build_morton_tree(sorted_primitives)
    
    # 4. Compute Adaptive Residuals (if enabled)
    # Cluster primitives (e.g., via spatial hashing or tree nodes)
    clusters = cluster_primitives(sorted_primitives)
    for cluster in clusters:
        # Geometric spread: p * tr(T*) / W
        # T* is transmittance, W is world volume/weight factor
        cluster.sigma_geo = compute_geometric_spread(cluster)
        
        # SH residual ratio: cluster-internal SH variance / global RMS
        cluster.rho_sh = compute_sh_residual_ratio(cluster)
        
    return tree, clusters

def render_frame(tree, clusters, camera, budget_c, capacity_k):
    """
    Select primitives and render for a single frame.
    """
    # 1. Determine selection criteria
    # If using fixed compression factor c:
    if budget_c is not None:
        # The prefix cut is determined by c. 
        # Higher c -> fewer primitives.
        selected_indices = get_prefix_cut(tree, compression_factor=budget_c)
    else:
        # If using capacity K:
        selected_indices = get_prefix_cut(tree, capacity=capacity_k)
        
    # 2. Gather primitives into rasterizer buffers
    # This is done on GPU, fusing SH evaluation and opacity compensation
    splats = []
    for idx in selected_indices:
        prim = tree.primitives[idx]
        
        # Evaluate SH color for camera direction
        color = evaluate_sh(prim.sh_coeffs, camera.direction)
        
        # Project covariance to 2D
        cov2d = project_covariance(prim.scale, prim.rotation, camera)
        
        splats.append({
            'mean': prim.mean,
            'cov2d': cov2d,
            'color': color,
            'opacity': prim.opacity
        })
        
    # 3. Rasterize
    image = rasterize(splats, camera.resolution)
    return image

def compute_geometric_spread(cluster):
    """
    Computes sigma_geo = p * tr(T*) / W
    """
    # p: number of primitives in cluster
    # T*: transmittance (accumulated from training views)
    # W: world volume/weight factor
    p = len(cluster.primitives)
    T_star = compute_transmittance(cluster.primitives)
    W = compute_world_weight(cluster)
    return (p * trace(T_star)) / W

def compute_sh_residual_ratio(cluster):
    """
    Computes rho_sh = cluster-internal SH variance / global RMS
    """
    cluster_variance = compute_variance(cluster.sh_coeffs)
    global_rms = compute_global_rms_all_primitives()
    return cluster_variance / global_rms
```

**Edge Cases and Pitfalls:**
*   **Geometric Redundancy Blind Spot:** A primitive can be geometrically redundant yet visually load-bearing (occludes neighbors, carries unique color) or geometrically isolated yet invisible (fully occluded). Aggressive geometry-only pruning removes primitives a contribution-aware score would keep [S1, unofficial].
*   **View-Direction Reversals:** Large-count outliers in per-frame selection coincide with view-direction reversals where leaves entering/leaving selection are back-facing and never rasterized [S1, unofficial].
*   **Zero-Score Primitives:** A significant portion of primitives (e.g., 24.3% on Block_A) may score exactly zero in LightGaussian significance, being never visible from any training view [S1, unofficial].

## Subtopics

### [Complexity-Aware Continuous Level of Detail for Gaussian Splatting](16064-complexity-aware-continuous-level-of-detail-for-gaussian-spl.md)
Also known as KoalaGS. A global, view-independent LoD formulation that provides a single primitive ordering whose prefixes define successive approximations of a scene. It solves the problem of variable-quality rendering by enabling progressive streaming and continuous budget adjustment without retraining, distinguishing itself from camera-dependent hierarchical selectors like FlexGS and discrete hierarchy methods like Octree-GS. It analyzes degradation across complexity levels by projecting image complexity maps onto Gaussians, grouping them into five bins, and measuring allocation, survival rate, and quality per bin. At 0.1% budget, it retains 2.02% of Gaussians in the simplest bin [0,1) vs 0.02% in the most complex bin [4,5], achieving higher PSNR (+2.25 dB at 0.1%) despite lower survival in complex bins.

## Operating characteristics
*   **Time Complexity:** Preprocessing (tree construction) is fast: 1.5 s for 8.56M primitives on A100 [S1, unofficial]. Per-frame selection and gather is 0.8 ms for 7.07M primitives at 1920×1080 [S1, unofficial]. LightGaussian scoring takes 234 s for 4565 views at 7.94M primitives on RTX 4080 [S1, unofficial].
*   **Memory Complexity:** Compact binary storage is efficient. DC-only path uses 3 floats per primitive, selecting 13.1M primitives in 3.9 GB on 8 GB GPUs [S1, unofficial]. Memory-quality Pareto on MatrixCity: 1240 MB → 34 MB at 28.4 → 17.5 dB [S1, unofficial]. Matching ~20.8 dB operating point needs 110.6 MB vs. 1926 MB (GPS full), a 17× saving [S1, unofficial].
*   **Performance:** BCT wall-clock time is 78±7 min over three seeds on room@529k with one A100, compared to ≈175 min for the traditional train-prune-finetune route (2.2× faster) [S1, unofficial]. BCT achieves higher quality (31.9 dB vs. 30.5 dB) [S1, unofficial].
*   **Quality:** BCT reaches near-lossless quality at low compression ratios. At c=4, quality loss is -0.05 dB [S1, unofficial]. On Lego scene, BCT stop rule reaches within 0.34 dB of Vanilla quality at 44% of primitives [S1, unofficial]. BCT beats random subsampling by +5.5/+7.5/+8.0 dB at matched budgets [S1, unofficial].
*   **Stability:** Per-frame count stability is high: median change of 1.1% between consecutive views on a 207-view trajectory at K=2M [S1, unofficial]. GPU selection accuracy matches offline Python reference exactly (5/5 views at c=1, 10/10 at c=8) [S1, unofficial].
*   **Strengths:** Fast preprocessing, low memory footprint, high quality at low budgets, renderer-agnostic export, continuous LOD without retraining.
*   **Limitations:** Geometric redundancy blind spot, view-direction reversal outliers, requires careful tuning of $c$ or $K$.

## Background
Budgeted Gaussian Splatting (BCT) was introduced in the paper "Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD" [S1, unofficial]. It addresses the limitations of traditional compression methods (train-prune-finetune) which are slow and lose quality. BCT provides a continuous budget-quality curve, allowing for variable primitive counts without retraining. It is used in large-scale Gaussian Splatting applications where real-time rendering and memory efficiency are critical.

## Sources
[S1] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
[S2] KoalaGS: Complexity-Aware Continuous Level of Detail for Gaussian Splatting, https://arxiv.org/abs/2610.28615, fetched 2026-10-26 (unofficial)
