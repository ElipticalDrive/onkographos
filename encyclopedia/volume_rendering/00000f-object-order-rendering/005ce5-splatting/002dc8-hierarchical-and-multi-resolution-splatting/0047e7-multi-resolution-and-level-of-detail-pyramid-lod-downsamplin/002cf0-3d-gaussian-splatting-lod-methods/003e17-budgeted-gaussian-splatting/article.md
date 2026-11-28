# Budgeted Gaussian Splatting


## Overview
Budgeted Gaussian Splatting (BCT) is a training and deployment method for large-scale 3D Gaussian Splatting that enables continuous, variable primitive counts (Level of Detail) without retraining [S1, unofficial]. It solves the problem of fixed-budget compression by constructing a Morton-ordered tree of primitives and a coverage-preserving prefix cut, allowing for near-lossless quality at low compression ratios [S1, unofficial]. It distinguishes itself from sibling methods like [Hierarchical 3D Gaussian Splatting (H-3DGS)](10279-hierarchical-3d-gaussian-splatting-h-3dgs.md) by providing a single, global primitive ordering whose prefixes define successive approximations of the scene, rather than camera-dependent hierarchical selectors or discrete hierarchy methods [S1, unofficial].

## How it works
**Inputs:** A trained 3D Gaussian Splatting model (means, scales, rotations, opacities, spherical harmonics coefficients) and a set of training views [S1, unofficial].
**Outputs:** A compact binary file containing primitives in Morton order, and a runtime scheduler that selects a subset of primitives for rendering based on a budget parameter $c$ [S1, unofficial].

**Data Structures:**
1. **Morton-Ordered Tree:** Primitives are sorted by their 3D Morton code, enabling efficient prefix traversal [S1, unofficial].
2. **Compact Binary Format:** Stores raw primitives in Morton order, per-band equivalent planks, and SH first moments of all degrees [S1, unofficial].
   - **DC-only Compact Path:** Stores 3 floats per primitive (DC component), enabling selection of up to 13.1M primitives in 3.9 GB [S1, unofficial].
   - **Full Export:** Standard 3DGS PLY format with SH zero-padded to degree 3, scales/opacities in log/logit domain [S1, unofficial].
3. **Residuals (for Adaptive Selection):** Per cluster, two residuals are stored:
   - Geometric spread: $\sigma_{geo} = \frac{p \cdot \text{tr}(T^*)}{W}$ (world units), projected to pixels by $\sigma \cdot f / d$ [S1, unofficial].
   - SH residual ratio: $\rho_{sh}$ (cluster-internal SH variance vs. global RMS) [S1, unofficial].

**Parameters:**
* $c$: Compression factor or LOD tier parameter. Typical values: $c \in \{1, 2, 4, 8, 16, 32\}$ [S1, unofficial].
* $K$: Capacity or maximum primitive count for the trajectory (e.g., $K=2\text{M}$) [S1, unofficial].

**Algorithm Steps:**
1. **Preprocessing (Offline):**
   1. Construct a Morton-order tree on the trained primitives. Takes 1.5 s with 19.4 GiB peak memory on an A100 GPU for 8.56M primitives [S1, unofficial].
   2. (Optional) Compute LightGaussian significance scores for pure primitive-count budgets: score every primitive with accumulated per-pixel $T^*\alpha$ over all training views via a one-pass forward scan [S1, unofficial]. Validate CUDA accumulation using the telescoping identity $\sum_i s_i = \sum_p (1 - T^*_{\text{final}})$, matched to $10^{-5}$ relative [S1, unofficial]. Select top-$N$ primitives based on this score for a given budget [S1, unofficial].
2. **Runtime Selection (Per Frame):**
   1. Determine the target budget based on $c$ or $K$.
   2. Traverse the coverage-preserving prefix cut on the GPU.
   3. Gather the selected prefix directly into rasterizer buffers, fusing SH evaluation and opacity compensation. No CPU-side collection or host-to-device upload is required [S1, unofficial].
   4. (Adaptive Mode) Use $\sigma_{geo}$ and $\rho_{sh}$ to refine selection if content-aware pruning is enabled [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess(primitives, training_views):
    # 1. Compute 3D Morton codes for each primitive's mean position
    morton_codes = [interleave_bits(p.mean.x, p.mean.y, p.mean.z) for p in primitives]
    
    # 2. Sort primitives by Morton code
    indices = sorted(range(len(primitives)), key=lambda i: morton_codes[i])
    sorted_primitives = [primitives[i] for i in indices]
    
    # 3. Build implicit Morton tree (sorted array enables prefix traversal)
    tree = sorted_primitives
    
    # 4. Compute Adaptive Residuals (if enabled)
    clusters = spatial_hash_cluster(sorted_primitives)
    for cluster in clusters:
        p_count = len(cluster.primitives)
        # T* is accumulated front-to-back transmittance over training views
        T_star_sum = sum(compute_transmittance(p, v) for p in cluster.primitives for v in training_views)
        W = cluster.bounds.volume
        cluster.sigma_geo = (p_count * T_star_sum) / W
        
        # SH residual ratio: cluster-internal SH variance / global RMS
        cluster_variance = variance(cluster.sh_coeffs)
        global_rms = rms(all_primitives.sh_coeffs)
        cluster.rho_sh = cluster_variance / global_rms
        
    return tree, clusters

def render_frame(tree, clusters, camera, budget_c, capacity_k, adaptive_mode=False):
    # 1. Determine selection criteria
    if budget_c is not None:
        selected_indices = get_prefix_cut_by_factor(tree, compression_factor=budget_c)
    else:
        selected_indices = get_prefix_cut_by_capacity(tree, capacity=capacity_k)
        
    # 2. Gather primitives into rasterizer buffers on GPU
    # Fuses SH evaluation and opacity compensation, no CPU upload
    splats = []
    for idx in selected_indices:
        prim = tree[idx]
        color = evaluate_sh(prim.sh_coeffs, camera.direction)
        cov2d = project_covariance(prim.scale, prim.rotation, camera)
        opacity = prim.opacity * opacity_compensation(prim, camera)
        splats.append({'mean': prim.mean, 'cov2d': cov2d, 'color': color, 'opacity': opacity})
        
    # 3. Rasterize
    image = alpha_blend_rasterize(splats, camera.resolution)
    return image

def compute_transmittance(primitive, view):
    # Accumulated per-pixel T*alpha contribution for scoring
    return primitive.opacity * gaussian_projection(primitive, view)

def get_prefix_cut_by_factor(tree, compression_factor):
    # Returns indices for a coverage-preserving prefix cut
    total = len(tree)
    length = total // compression_factor
    return list(range(length))

def get_prefix_cut_by_capacity(tree, capacity):
    return list(range(min(capacity, len(tree))))

def interleave_bits(x, y, z):
    # Standard 3D Morton code interleaving of x, y, z bits
    return (z << 2) | (y << 1) | x

def spatial_hash_cluster(points):
    # Standard grid-based spatial clustering
    return grid_cluster(points, cell_size=1.0)

def variance(coeffs):
    mean = sum(coeffs) / len(coeffs)
    return sum((c - mean)**2 for c in coeffs) / len(coeffs)

def rms(coeffs):
    mean = sum(coeffs) / len(coeffs)
    return (sum((c - mean)**2 for c in coeffs) / len(coeffs)) ** 0.5

def evaluate_sh(sh_coeffs, direction):
    # Evaluate spherical harmonics at given direction
    return evaluate_sh_function(sh_coeffs, direction)

def project_covariance(scale, rotation, camera):
    # Jacobian-based projection of 3D covariance to 2D
    return camera.jacobian @ rotation @ scale @ rotation.T @ camera.jacobian.T

def opacity_compensation(prim, camera):
    # Compensation factor for depth/size
    return 1.0

def alpha_blend_rasterize(splats, resolution):
    # Standard alpha-blending rasterization
    image = zeros(resolution)
    for s in sorted(splats, key=lambda x: x.depth):
        image = blend_pixel(image, s)
    return image
```

**Edge Cases and Pitfalls:**
* **Geometric Redundancy Blind Spot:** A primitive can be geometrically redundant yet visually load-bearing (occludes neighbors, carries unique color) or geometrically isolated yet invisible (fully occluded). Aggressive geometry-only pruning removes primitives a contribution-aware score would keep [S1, unofficial].
* **View-Direction Reversals:** Large-count outliers in per-frame selection coincide with view-direction reversals where leaves entering/leaving selection are back-facing and never rasterized [S1, unofficial].
* **Zero-Score Primitives:** A significant portion of primitives (e.g., 24.3% on Block_A) may score exactly zero in LightGaussian significance, being never visible from any training view [S1, unofficial].

## Subtopics

### [Complexity-Aware Continuous Level of Detail for Gaussian Splatting](16064-complexity-aware-continuous-level-of-detail-for-gaussian-spl.md)
Also known as KoalaGS. A global, view-independent LoD formulation that provides a single primitive ordering whose prefixes define successive approximations of a scene. It solves the problem of variable-quality rendering by enabling progressive streaming and continuous budget adjustment without retraining, distinguishing itself from camera-dependent hierarchical selectors like FlexGS and discrete hierarchy methods like Octree-GS. It analyzes degradation across complexity levels by projecting image complexity maps onto Gaussians, grouping them into five bins, and measuring allocation, survival rate, and quality per bin. At 0.1% budget, it retains 2.02% of Gaussians in the simplest bin [0,1) vs 0.02% in the most complex bin [4,5], achieving higher PSNR (+2.25 dB at 0.1%) despite lower survival in complex bins.

## Operating characteristics
* **Time Complexity:** Preprocessing (tree construction) is fast: 1.5 s for 8.56M primitives on A100 [S1, unofficial]. Per-frame selection and gather is 0.8 ms for 7.07M primitives at 1920×1080 [S1, unofficial]. LightGaussian scoring takes 234 s for 4565 views at 7.94M primitives on RTX 4080 [S1, unofficial].
* **Memory Complexity:** Compact binary storage is efficient. DC-only path uses 3 floats per primitive, selecting 13.1M primitives in 3.9 GB on 8 GB GPUs [S1, unofficial]. Memory-quality Pareto on MatrixCity: 1240 MB → 34 MB at 28.4 → 17.5 dB [S1, unofficial]. Matching ~20.8 dB operating point needs 110.6 MB vs. 1926 MB (GPS full), a 17× saving [S1, unofficial].
* **Performance:** BCT wall-clock time is 78±7 min over three seeds on room@529k with one A100, compared to ≈175 min for the traditional train-prune-finetune route (2.2× faster) [S1, unofficial]. BCT achieves higher quality (31.9 dB vs. 30.5 dB) [S1, unofficial].
* **Quality:** BCT reaches near-lossless quality at low compression ratios. At c=4, quality loss is -0.05 dB [S1, unofficial]. On Lego scene, BCT stop rule reaches within 0.34 dB of Vanilla quality at 44% of primitives [S1, unofficial]. BCT beats random subsampling by +5.5/+7.5/+8.0 dB at matched budgets [S1, unofficial].
* **Stability:** Per-frame count stability is high: median change of 1.1% between consecutive views on a 207-view trajectory at K=2M [S1, unofficial]. GPU selection accuracy matches offline Python reference exactly (5/5 views at c=1, 10/10 at c=8) [S1, unofficial].
* **Strengths:** Fast preprocessing, low memory footprint, high quality at low budgets, renderer-agnostic export, continuous LOD without retraining [S1, unofficial].
* **Limitations:** Geometric redundancy blind spot, view-direction reversal outliers, requires careful tuning of $c$ or $K$ [S1, unofficial].

## Background
Budgeted Gaussian Splatting (BCT) was introduced in the paper "Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD" [S1, unofficial]. It addresses the limitations of traditional compression methods (train-prune-finetune) which are slow and lose quality [S1, unofficial]. BCT provides a continuous budget-quality curve, allowing for variable primitive counts without retraining [S1, unofficial]. It is used in large-scale Gaussian Splatting applications where real-time rendering and memory efficiency are critical [S1, unofficial].

## Sources
[S1] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
[S2] KoalaGS: Complexity-Aware Continuous Level of Detail for Gaussian Splatting, https://arxiv.org/abs/2610.28615, fetched 2026-10-26 (unofficial)
