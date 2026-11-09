# Joint Optimization with Loss Functions

Also called: Joint Optimization of Surfel Parameters and HDR Environment Map

## Overview
This method defines a composite loss function for the joint optimization of 3D surface splatting (surfel) parameters and a learnable HDR environment map in inverse rendering tasks [S1, unofficial]. It addresses the ambiguity of multi-layer compositing and the lack of geometric priors by combining photometric reconstruction with depth consolidation and normal consistency regularizers [S1, unofficial]. It is distinguished from sibling regularization methods by its specific focus on surfel-level geometric constraints (depth variance and inter-layer spacing) rather than general volume density smoothing or output noise suppression [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered image $\hat{\mathbf{I}}$, ground-truth image $\mathbf{I}^*$, ground-truth mask $\mathbf{M}^*$ (optional), surfel parameters (positions $\mathbf{c}$, tangent vectors parameterized as unit quaternions $\mathbf{q}$, log-space scales, albedo $\rho$, metallic $\rho_m$, roughness $\alpha$), and environment map parameters [S1, unofficial].
*   **Outputs:** Scalar total loss $\mathcal{L}$ and gradients for the Adam optimizer [S1, unofficial].

**Data Structures:**
*   **Surfel Parameters:** Positions $\mathbf{c}_i$, tangent vectors parameterized as unit quaternions $\mathbf{q}_i$, log-space scales, albedo $\rho_i$, metallic $\rho_{m,i}$, roughness $\alpha_i$ [S1, unofficial].
*   **Layer Data:** For each pixel $p$, a set of $K$ composited surface layers [S1, unofficial]. Each layer $i$ has compositing weight $w_i$, mean depth $z_i$, and depth variance $\sigma_i^2$ [S1, unofficial].
*   **Neighbor Index:** A spatial data structure mapping surfel indices to nearest neighbors $\mathcal{N}_i$ with compatible normal orientations [S1, unofficial].

**Formulas:**
The total loss is a weighted sum:
$$
\mathcal{L} = \mathcal{L}_{\text{photo}} + \lambda_{\text{sil}} \mathcal{L}_{\text{sil}} + \lambda_{\text{cons}} \mathcal{L}_{\text{cons}} + \lambda_{\text{nc}} \mathcal{L}_{\text{nc}}
$$
where $\lambda$ terms are scalar weights [S1, unofficial].

1.  **Photometric Loss ($\mathcal{L}_{\text{photo}}$):** Combines L1 reconstruction and Structural Similarity (SSIM) [S1, unofficial]. For HDR targets, $\hat{\mathbf{I}}$ and $\mathbf{I}^*$ are transformed via log-space and sRGB transfer before loss computation to compress dynamic range [S1, unofficial].
    $$
    \mathcal{L}_{\text{photo}} = (1 - \lambda_{\text{ssim}}) \|\hat{\mathbf{I}} - \mathbf{I}^*\|_1 + \lambda_{\text{ssim}} (1 - \text{SSIM}(\hat{\mathbf{I}}, \mathbf{I}^*))
    $$
    where $\hat{\mathbf{I}}$ is the rendered image after tone mapping/gamma correction, $\mathbf{I}^*$ is the ground-truth image, and $\lambda_{\text{ssim}}$ is the weighting factor [S1, unofficial].

2.  **Silhouette Loss ($\mathcal{L}_{\text{sil}}$):** Supervises total accumulated coverage using soft Intersection-over-Union (IoU) [S1, unofficial]. Disabled once adaptive density control begins [S1, unofficial].
    $$
    \mathcal{L}_{\text{sil}} = \text{IoU}(1 - \bar{T}_{K+1}, \mathbf{M}^*)
    $$
    where $\bar{T}_{K+1}$ is the residual transmittance (silhouette mask) and $\mathbf{M}^*$ is the ground-truth mask [S1, unofficial].

3.  **Depth Consolidation Loss ($\mathcal{L}_{\text{cons}}$):** Penalizes spurious depth layers [S1, unofficial]. Computed per pixel $p \in \mathcal{P}$ as the average of inter-layer and intra-layer terms [S1, unofficial].
    $$
    \mathcal{L}_{\text{cons}} = \frac{1}{|\mathcal{P}|} \sum_{p \in \mathcal{P}} (\mathcal{L}_{\text{inter-c}} + \mathcal{L}_{\text{intra-c}})
    $$
    *   **Inter-layer term:** Penalizes weight allocation across distant layers [S1, unofficial].
        $$
        \mathcal{L}_{\text{inter-c}} = \sum_{i < j} w_i w_j |z_i - z_j|
        $$
        where $w_i, w_j$ are compositing weights and $z_i, z_j$ are mean depths of layers $i, j$ [S1, unofficial].
    *   **Intra-layer term:** Penalizes depth spread within a layer [S1, unofficial].
        $$
        \mathcal{L}_{\text{intra-c}} = \sum_i w_i^2 \sigma_i^2
        $$
        where $w_i$ is the compositing weight and $\sigma_i^2$ is the depth variance of layer $i$ [S1, unofficial].

4.  **Normal Consistency Loss ($\mathcal{L}_{\text{nc}}$):** Penalizes angular deviation between a surfel's normal and its orientation-aware nearest neighbors in 3D [S1, unofficial].
    $$
    \mathcal{L}_{\text{nc}} = \frac{\sum_i \sum_{j \in \mathcal{N}_i} w_{ij} (1 - \mathbf{n}_i \cdot \mathbf{n}_j)}{\sum_i \sum_{j \in \mathcal{N}_i} w_{ij}}
    $$
    where $\mathcal{N}_i$ are nearest neighbors with compatible normal orientation, $\mathbf{n}_i, \mathbf{n}_j$ are normals, and $w_{ij} = \exp(-\|\mathbf{c}_i - \mathbf{c}_j\|^2 / 2\sigma^2)$ are distance-dependent weights derived from local intersample spacing [S1, unofficial].

**Parameters:**
*   $\lambda_{\text{ssim}}$: SSIM weighting factor.
*   $\lambda_{\text{sil}}, \lambda_{\text{cons}}, \lambda_{\text{nc}}$: Scalar weights for respective loss terms.
*   $\sigma$: Parameter for normal consistency weight decay, derived from local intersample spacing [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_joint_loss(rendered_img, gt_img, gt_mask, surfels, layers,
                       lambda_ssim, lambda_sil, lambda_cons, lambda_nc,
                       use_mask, training_stage, sigma_local):
    """
    Compute the total joint optimization loss for 3DSS inverse rendering.
    """
    # Dimensions derived from inputs
    num_pixels = len(rendered_img)
    num_surfels = len(surfels.positions)

    # 1. Photometric Loss
    # Apply log-space and sRGB transfer to HDR targets if necessary before L1/SSIM
    l1_loss = sum(abs(rendered_img[i] - gt_img[i]) for i in range(num_pixels)) / num_pixels
    ssim_loss = 1 - standard_ssim(rendered_img, gt_img)  # Standard structural similarity metric [G]
    l_photo = (1 - lambda_ssim) * l1_loss + lambda_ssim * ssim_loss

    total_loss = l_photo

    # 2. Silhouette Loss
    if use_mask and training_stage == 'early':
        sil_loss_sum = 0.0
        for p in range(num_pixels):
            # Residual transmittance is 1 - accumulated_opacity
            accumulated_opacity = sum(layer.w for layer in layers[p])
            pred_sil = 1.0 - accumulated_opacity
            intersection = sum(pred_sil * gt_mask[p])
            union = sum(pred_sil) + sum(gt_mask[p]) - intersection
            sil_loss_sum += 1.0 - (intersection / (union + 1e-6))
        l_sil = sil_loss_sum / num_pixels
        total_loss += lambda_sil * l_sil

    # 3. Depth Consolidation Loss
    l_cons_total = 0.0
    for p in range(num_pixels):
        pixel_layers = layers[p]
        K = len(pixel_layers)
        l_inter = 0.0
        l_intra = 0.0
        # Inter-layer: sum over all pairs i < j
        for i in range(K):
            for j in range(i + 1, K):
                l_inter += pixel_layers[i].w * pixel_layers[j].w * abs(pixel_layers[i].z - pixel_layers[j].z)
        # Intra-layer: sum over all layers
        for i in range(K):
            l_intra += (pixel_layers[i].w ** 2) * pixel_layers[i].sigma2
        l_cons_total += (l_inter + l_intra)
    l_cons = l_cons_total / num_pixels
    total_loss += lambda_cons * l_cons

    # 4. Normal Consistency Loss
    # Requires neighbor indices computed via spatial data structure
    neighbor_indices = build_spatial_index(surfels)  # Spatial data structure lookup [G]
    l_nc_num = 0.0
    l_nc_den = 0.0
    for i in range(num_surfels):
        neighbors = neighbor_indices[i]
        n_i = surfels.normals[i]
        c_i = surfels.positions[i]
        for j in neighbors:
            n_j = surfels.normals[j]
            c_j = surfels.positions[j]
            # Distance-dependent weight
            dist_sq = sum((c_i[k] - c_j[k]) ** 2 for k in range(3))
            w_ij = math.exp(-dist_sq / (2 * sigma_local ** 2))
            # Dot product for angle
            dot_prod = sum(n_i[k] * n_j[k] for k in range(3))
            # Clamp for numerical stability
            dot_prod = max(-1.0, min(1.0, dot_prod))
            l_nc_num += w_ij * (1 - dot_prod)
            l_nc_den += w_ij
    if l_nc_den > 0:
        l_nc = l_nc_num / l_nc_den
        total_loss += lambda_nc * l_nc

    return total_loss

def standard_ssim(a, b):
    """Placeholder for standard SSIM implementation [G]."""
    return 0.95  # Dummy value for structural completeness

def build_spatial_index(surfels):
    """Placeholder for spatial data structure (e.g., KD-Tree) lookup [G]."""
    return {i: [j for j in range(len(surfels.positions)) if i != j] for i in range(len(surfels.positions))}
```

**Edge Cases and Pitfalls:**
*   **HDR Gradient Dominance:** Bright pixels can dominate gradients [S1, unofficial]. Mitigation: Apply log-space and sRGB transfer to both rendered and ground-truth images before loss computation [S1, unofficial].
*   **Silhouette Activation:** The silhouette loss should be disabled once adaptive density control begins, as the photometric loss and densification mechanism take over coverage enforcement [S1, unofficial].
*   **Neighbor Index Cost:** Computing nearest neighbors is expensive [S1, unofficial]. The spatial data structure must be rebuilt periodically and cached between updates to amortize cost [S1, unofficial].
*   **Sigma Derivation:** The $\sigma$ in the normal consistency weight $w_{ij}$ is derived from local intersample spacing [S1, unofficial]. Exact calculation details are in section A.2 of the source [S1, unofficial].

## Subtopics

### HDR Environment Map Optimization
Models distant lighting as a high-dynamic-range spherical environment map in equirectangular coordinates [S2]. Radiance values are optimized directly with solid-angle compensation for non-uniform texel area [S2]. The pipeline uses normal-based hemisphere sampling to halve the number of rays per pixel compared to full-sphere sampling [S2]. After hemisphere culling and empty-space removal, there are 512 candidate directions [S2]. The map resolution is $16 \times 32$ equirectangular, which limits recoverable illumination bandwidth to the object's BRDF lobes [S2].

## Operating characteristics
*   **Complexity:** The normal consistency loss requires $O(N \cdot k)$ operations where $N$ is the number of surfels and $k$ is the number of neighbors, with periodic rebuilds of the spatial index [S1, unofficial]. The depth consolidation loss is $O(P \cdot K^2)$ per iteration where $P$ is pixels and $K$ is layers per pixel [G].
*   **Performance:** Evaluated on object-level inverse rendering tasks including geometry reconstruction, novel view synthesis, and scene relighting [S1, unofficial].
*   **Strengths:** Enforces geometric thinness at two scales (inter-layer and intra-layer) and aligns surfel normals in 3D, avoiding oversmoothing associated with screen-space regularizers [S1, unofficial].
*   **Limitations:** Requires careful scheduling of loss terms (e.g., disabling silhouette loss later in training) [S1, unofficial]. Computationally heavier than pure photometric losses due to neighbor lookups and layer-wise operations [G].

## Background
Originated in the "3DSS: 3D Surface Splatting for Inverse Rendering" paper [S1, unofficial]. It builds on scattered-data approximation frameworks [Wendland 2004] and depth consolidation ideas from Barron et al. [2022] and Huang et al. [2024], adapting them to surfel primitives [S1, unofficial]. It uses the Adam optimizer [Kingma and Ba 2014] for joint optimization of surfel parameters and HDR environment maps [S1, unofficial].

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S2] VisibilitySplat: Ray-Traced Occlusion for Relighting with 3D Gaussian Splatting, https://doi.org/10.24132/csrn.2026-e17, fetched 2026-10-26
