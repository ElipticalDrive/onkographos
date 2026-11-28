# Sorting and Rendering Pipeline Optimization


## Overview
This family addresses the computational bottlenecks in the 3D Gaussian Splatting (3DGS) rendering pipeline, specifically targeting the memory-bound front-end (projection and sorting) and the compute-bound rasterization stage [12180]. It sets itself apart from generic optimization by employing hardware-aware strategies, such as the Roofline model to identify resource constraints [S1, unofficial], and algorithmic innovations like comparison-free sorting to reduce synchronization overhead [S1, unofficial].

## Shared principles
The rendering pipeline of 3DGS consists of two primary stages, each with distinct optimization goals [12180]:

1.  **Front-End (Memory-Bound):**
    *   **Projection and Sorting:** 3D Gaussians are projected to 2D, generating a depth key for each. To ensure correct alpha compositing, Gaussians must be sorted front-to-back (increasing depth) within each image tile.
    *   **Attribute Fetching:** For each Gaussian-pixel pair, attributes (opacity, spherical harmonic coefficients) are fetched from global memory.
    *   **Optimization Goal:** Reduce global memory traffic and bandwidth usage. This is achieved by compressing data (quantization, key compression) and fusing kernels to avoid intermediate writes [12180].

2.  **Rasterization (Compute-Bound):**
    *   **Alpha Compositing:** The final pixel color $C$ is computed via alpha blending:
        $$C = \sum_{i=1}^N c_i \alpha_i T_i, \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$$
        where $c_i$ is the color, $\alpha_i$ is the opacity, and $T_i$ is the transmittance [12180].
    *   **Gaussian Evaluation:** The opacity $\alpha_i$ for a pixel at $(x,y)$ relative to a Gaussian center $(u,v)$ is computed as [12180]:
        $$\alpha_i = o \exp(p(x, y))$$
        where $o$ is the base opacity and $p(x, y)$ is the Gaussian power:
        $$p(x, y) = -\frac{1}{2} \left( a(x - w)^2 + b(x - u)(y - v) + c(y - v)^2 \right)$$
        Here, $a, b, c$ are derived from the projected 2D covariance matrix, and $(w, v)$ are the center coordinates [12180].
    *   **Optimization Goal:** Increase instruction-level parallelism (ILP) and reduce expensive operations (like `exp`) within the per-pixel loop [12180].

For distributed training, the global set of 3D Gaussians $\mathcal{G}$ is partitioned across $M$ GPUs. Each GPU maintains a local shard $\mathcal{G}^{(m)}$ and its optimizer state. The training loop involves local projection of the shard, communication (e.g., all-to-all exchange) to route Gaussians to responsible GPUs, filtering via importance scoring or pruning, and local rasterization with gradient updates [15108].

## Subtopics

### [RoofGS](9830-roofgs.md)
RoofGS is an end-to-end acceleration method for 3D Gaussian Splatting (3DGS) that optimizes the rendering pipeline by aligning software strategies with hardware bottlenecks identified via the Roofline model. It solves the problem of uncoordinated optimization in 3DGS, where different stages are constrained by different resources (memory bandwidth vs. compute throughput). It sets itself apart from generic optimization by explicitly using the Roofline model to identify the "knee" point and applying memory-centric optimizations to stages below it and compute-centric optimizations to stages near it. See [RoofGS](9830-roofgs.md).

### [Tile-Based Comparison-Free Sorting](9991-tile-based-comparison-free-sorting.md)
Tile-Based Comparison-Free Sorting restricts depth sorting operations to individual tiles to avoid global synchronization, reduce hardware complexity, and achieve fixed latency per output. Unlike global radix sort, which incurs high memory traffic and variable latency due to histograms and prefix sums, this method provides deterministic O(N) cycles suitable for low-power hardware accelerators. It serves as the sorting mechanism for 3D Gaussian Splatting, ensuring Gaussians are ordered front-to-back for correct alpha compositing within each tile. See [Tile-Based Comparison-Free Sorting](9991-tile-based-comparison-free-sorting.md).

### Optimization Strategy
- [S1, unofficial] Optimize static Gaussians for the initial 3000 iterations as a warm-up phase.
- [S1, unofficial] Jointly optimize both the Gaussians and the deformation field after the warm-up phase.
- [S1, unofficial] Total loss L = Lcolor + λ1 * Lscales, where Lcolor is the color loss and λ1 is a weighting factor.
- [S1, unofficial] λ1: Weight for the scale loss, set to 0.1.
- [S1, unofficial] λv: Weight for another loss term (value 0.2), though its specific definition is not detailed in this excerpt.

### Background Color Optimization
- [S2, unofficial] Changing the default white background (RGB 1.0) to a gray value to improve training speed while maintaining rendering quality.
- [S2, unofficial] Background value `bg`: The three same values for RGB background channels. Optimal range is 0.3 ≤ bg ≤ 0.5.
- [S2, unofficial] Changing the background must be balanced; while it improves speed, extreme values may negatively impact learning rate, training loss, and PSNR states.
- [S2, unofficial] On the truck dataset with 500 training steps, setting bg=0.4 reduces pc time by approximately 21.85% compared to the default white background.

### Mobile-GS Rendering
- [S3, unofficial] A sorting-free rendering method that uses a transmittance proxy enriched with view-dependent modulation.
- [S3, unofficial] Pixel color $C = (1 - T) \sum_{i=1}^N \frac{c_i \alpha_i + T c_i b_g}{\sum_{i=1}^N \alpha_i w_i} + T c_i b_g$. Weight $w_i = \phi_i^2 + \frac{\phi_i}{d_i^2} + \exp \left( \frac{x_{max}}{d_i} \right)$. $T$ is transmittance, $c_i$ is color, $\alpha_i$ is opacity, $b_g$ is background color, $d_i$ is depth, $\phi_i$ is auxiliary feature from MLP, $x_{max}$ is a constant.
- [S3, unofficial] Unlike SortFreeGS which uses Gaussian depth but not scale, Mobile-GS exploits both depth and scale attributes to compute importance weight, reflecting that farther Gaussians have lower contribution and larger Gaussians provide more meaningful evidence.
- [S3, unofficial] Unlike GES which uses a two-stage rendering pipeline (depth image then filtering), Mobile-GS is single-stage and better suited for mobile deployment.

## Background
The optimization of the 3D Gaussian Splatting pipeline has evolved to address the high memory bandwidth requirements of sorting and the computational cost of per-pixel Gaussian evaluation. Early implementations relied on standard radix sorts, which were later refined by hardware-specific approaches like RoofGS [S1, unofficial] and comparison-free sorting [S1, unofficial]. Concurrently, training efficiency improvements, such as background color tuning [S2, unofficial] and specific loss weighting [S1, unofficial], have been integrated into the pipeline. Mobile-oriented variants like Mobile-GS [S3, unofficial] have further explored sorting-free alternatives to enable real-time rendering on resource-constrained devices.

## Sources
[S1] Alias-free 4D Gaussian Splatting, https://arxiv.org/abs/2511.18367v1, fetched 2026-10-26 (unofficial)
[S2] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S3] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29 (unofficial)
