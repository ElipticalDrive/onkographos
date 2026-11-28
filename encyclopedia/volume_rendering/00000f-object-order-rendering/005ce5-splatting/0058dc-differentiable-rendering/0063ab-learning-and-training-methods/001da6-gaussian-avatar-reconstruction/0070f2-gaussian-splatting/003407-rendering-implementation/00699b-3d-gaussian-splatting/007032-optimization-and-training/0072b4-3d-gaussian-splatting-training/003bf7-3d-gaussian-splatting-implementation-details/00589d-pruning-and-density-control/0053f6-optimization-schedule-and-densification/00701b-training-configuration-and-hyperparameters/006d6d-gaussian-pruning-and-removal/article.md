# Gaussian Pruning and Removal

Also called: Adaptive Gaussian Pruning, Ray Contribution-Based Pruning, Adaptive Redundancy Culling, Low-Opacity Culling

## Overview
Gaussian Pruning and Removal is a family of optimization techniques in 3D Gaussian Splatting (3DGS) that reduces the number of Gaussian primitives by eliminating redundant or low-contribution splats. It solves the problem of excessive memory consumption and computational overhead caused by the densification process, which generates many auxiliary Gaussians with low geometric meaning [S1]. Unlike densification control which limits freedom, these methods are applied post-densification or during optimization to guarantee real-time rendering for large-scale scenes with nearly identical rendering quality while significantly reducing Gaussian count and storage [S1].

## Shared principles
All methods in this family share the goal of reducing the Gaussian set $\mathcal{G}$ to a subset $\mathcal{G}'$ by evaluating a "contribution" or "redundancy" metric for each Gaussian and removing those below a threshold or ranking. The core mechanism involves:
1.  **Metric Calculation:** Computing a scalar value for each Gaussian $g_i$ representing its visual importance (e.g., pixel contribution, ray contribution, or redundancy score) [S1, S2, S3].
2.  **Thresholding/Selection:** Comparing the metric against a threshold or ranking to identify Gaussians for removal [S1, S2, S3].
3.  **Removal:** Deleting the identified Gaussians from the scene representation [S1, S2, S3].
4.  **Post-Pruning Optimization:** Continuing training on the remaining Gaussians, allowing them to absorb the rendering workload of the deleted primitives [S1].

## Subtopics

### [Gaussian Pruning](18664-gaussian-pruning.md)
Gaussian Pruning is a post-densification strategy that removes redundant "auxiliary" Gaussians by retaining only the "dominant" Gaussians that contribute most significantly to the final pixel color for each ray. It is simpler than controlling split positions during densification and guarantees real-time rendering for large-scale scenes.

**Mechanism:**
For each pixel $p$ in each training image, the algorithm identifies the set of overlapping Gaussians $N_p$ sorted by depth (front-to-back). It calculates the contribution weight $weight_i^p$ for each Gaussian $g_i \in N_p$ using the alpha-blending formula:
$$ weight_i^p = \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
where $\alpha_i$ is the opacity of $g_i$, and the product is over all Gaussians $j$ preceding $g_i$ in depth order within $N_p$.

The top-$K$ Gaussians with the highest weights are identified as dominant for pixel $p$. A Gaussian $g$ is considered dominant for pixel $p$ if:
$$ weight_g^p \ge \text{sort}(\{-weight_j^p, g_j \in N_p\})[K] $$
where $\text{sort}$ returns the $K$-th largest weight.

A Gaussian $g$ is marked as retained globally if it is identified as dominant for at least one pixel across all training images. All Gaussians not marked as retained are deleted. The colors (and potentially other attributes) of the retained dominant Gaussians are then optimized to absorb the rendering work of the deleted auxiliaries.

**Parameters:**
- **$K$:** The number of top-weight Gaussians to retain per pixel as dominant.
  - **Typical Value:** 1.
  - **Effect:** Smaller $K$ enforces stricter filtering (more removal, potential quality degradation); larger $K$ retains more Gaussians (higher quality, higher storage/compute). The default/final value used in the source is 1.

**Edge Cases:**
- **Precision:** The weight calculation involves products of $(1-\alpha_j)$. If $\alpha_j$ is very close to 1, the product can underflow or become numerically unstable if not handled carefully, though standard alpha blending usually mitigates this.
- **Boundary Conditions:** If a pixel has fewer than $K$ overlapping Gaussians, all overlapping Gaussians are retained as dominant [G].
- **Degenerate Input:** If a Gaussian has zero opacity, its weight is zero, and it will likely be pruned unless $K$ is larger than the number of non-zero weight Gaussians [G].

### Adaptive Gaussian Pruning
A dynamic removal of superfluous Gaussians based on their contribution to the final image, integrating opacity, variance, and pixel contribution criteria to eliminate extra Gaussians in low-impact or flat areas [S1].

**Mechanism:**
After densification, Gaussians are assessed based on low opacity, high variance (redundant spatial information), and low pixel contribution [S1]. A Gaussian is removed if its contribution satisfies:
$$ w_i = \max_x (N(x; \mu_i, \Sigma_i)) \cdot \alpha_i < \tau $$
where $w_i$ is the contribution metric, $N$ is the Gaussian function, $\alpha_i$ is opacity, and $\tau$ is a predefined threshold [S1].

In under-reconstructed regions, Gaussians are cloned with added noise to create diversity. In over-reconstructed areas, Gaussians are separated by lowering their scale to 0.8x the original value to preserve suitable sizes [S1]. Gaussians with opacity values below a given threshold or sizes above the standard are then deleted using the `prune_points` tool [S1].

### Ray Contribution-Based Pruning
A pruning technique applied during optimization to reduce the number of Gaussians by removing those that do not significantly contribute to any training view, based on exact ray contribution [S2].

**Mechanism:**
An importance score $h(\mathbf{p}_i)$ is defined by aggregating the ray contribution of Gaussian $p_i$ along all rays of all input images:
$$ h(\mathbf{p}_i) = \max_{I_f \in \mathcal{I}_f, r \in I_f} \alpha_i^r \tau_i^r $$
where $\alpha_i^r \tau_i^r$ indicates the ray contribution for the pixel's final color prediction in 4 of Gaussian $p_i$ along ray $r$ [S2]. This formulation uses the exact ray contribution (as opposed to e.g. the opacity) as well as the max operator (instead of e.g. the mean) which is independent of the number of input images, hence more robust to different types of scene coverage [S2].

A mask value is calculated as:
$$ m_i = m(\mathbf{p}_i) = \mathbb{1} (h(\mathbf{p}_i) < t_{\text{prune}}) $$
and all Gaussians with a mask value of one are removed [S2].

**Parameters:**
- **Pruning Threshold $t_{\text{prune}}$:**
  - Default model: 0.01 [S2].
  - Lightweight model: 0.25 [S2].
  - Large Zip-NeRF scenes (default): 0.005 [S2].
  - Large Zip-NeRF scenes (lightweight): 0.03 [S2].

**Timing:**
- Pruning is applied twice during optimization: once after 16,000 steps and once after 24,000 steps [S2].

### Adaptive Redundancy Culling
A method to reduce the number of 3D Gaussian primitives by removing those with high redundancy scores, which indicate they overlap significantly with other primitives [S3].

**Mechanism:**
1.  Calculate the redundancy score for all primitives [S3].
2.  Identify primitives with a redundancy score greater than the adaptive threshold $\mu_{th}$ [S3].
3.  From the identified primitives, delete exactly 50% of them [S3].
4.  Select the primitives to delete based on having the lowest opacity values, as opacity is an independent estimator of visual contribution [S3].
5.  Apply the culling process every 1000 iterations during optimization [S3].

**Formula:**
$$ \mu_{th} = \max(\mu + k\sigma, 3) $$
where $\mu$ and $\sigma$ are the mean and standard deviation of the redundancy score for all primitives, and $k=1$ [S3]. This ensures the threshold is at least 3 to prevent culling primitives that contribute to at least one region where they coincide with just two other primitives [S3].

**Pitfalls:**
- Culling primitives with a redundancy score less than 3 adversely affects quality because such primitives contribute to at least one region where they coincide with just two other primitives [S3].

**Comparison:**
- Simple low-opacity culling gives results close to redundancy culling, but performance varies by dataset; combining both yields consistently better performance [S3].

### Low-Opacity Culling
A method to remove primitives with very low opacity values to reduce memory footprint with minimal visual impact [S3].

**Mechanism:**
- Remove 3% of the primitives with the lowest opacity values during each culling step [S3].
- Continue culling low-opacity Gaussians (opacity < 1/255) after the 15K iteration mark of training, as they are neither rendered nor optimized and thus provide no contribution to image quality [S3].

**Parameters:**
- **Maximum opacity threshold:** 0.05. Primitives with opacity above this value are not removed, as removing higher opacity primitives degrades quality [S3].

## Background
3D Gaussian Splatting (3DGS) was introduced by Kerbl et al. [2023] as a real-time explicit 3D representation. The densification process, which splits and clones Gaussians to improve detail, often results in an excessive number of primitives, leading to high memory consumption and computational cost [S1]. To address this, various pruning and density control methods have been developed.

**Adaptive Gaussian Pruning** was proposed by Vuong Pham et al. [2025] to dynamically remove superfluous Gaussians based on their contribution to the final image, solving the problem of model "bloating" [S1].

**Ray Contribution-Based Pruning** was introduced in RadSplat [2025] to create a lightweight representation for robust real-time rendering, using exact ray contribution and max aggregation to be robust to scene coverage [S2].

**Adaptive Redundancy Culling** and **Low-Opacity Culling** were presented in "Reducing the Memory Footprint of 3D Gaussian Splatting" [2024] to significantly reduce the memory footprint by culling redundant and low-opacity primitives during training [S3].

## Sources
[S1] Optimization of 3D Gaussian Splatting for Accurate Image Reconstruction, https://doi.org/10.59232/dst-v4i2p101, fetched 2026-10-26
[S2] RadSplat: Radiance Field-Informed Gaussian Splatting for Robust Real- Time Rendering with 900+ FPS, https://doi.org/10.1109/3dv66043.2025.00018, fetched 2026-10-26
[S3] Reducing the Memory Footprint of 3D Gaussian Splatting, https://doi.org/10.1145/3651282, fetched 2026-10-26
