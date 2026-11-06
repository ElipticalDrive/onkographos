# Joint Optimization with Loss Functions

Also called: Joint Optimization of Surfel Parameters and HDR Environment Map

## Overview
This method defines a composite loss function for the joint optimization of 3D surface splatting (surfel) parameters and a learnable HDR environment map in inverse rendering tasks [S1, unofficial]. It addresses the ambiguity of multi-layer compositing and the lack of geometric priors by combining photometric reconstruction with depth consolidation and normal consistency regularizers [S1, unofficial]. It is distinguished from sibling regularization methods by its specific focus on surfel-level geometric constraints (depth variance and inter-layer spacing) rather than general volume density smoothing or output noise suppression.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered image $\hat{\mathbf{I}}$, ground-truth image $\mathbf{I}^*$, ground-truth mask $\mathbf{M}^*$ (optional), surfel parameters (positions $\mathbf{c}$, normals $\mathbf{n}$, scales, albedo, metallic, roughness), and environment map parameters.
*   **Outputs:** Scalar total loss $\mathcal{L}$ and gradients for the Adam optimizer [S1, unofficial].

**Data Structures:**
*   **Surfel Parameters:** Positions $\mathbf{c}_i$, tangent vectors parameterized as unit quaternions $\mathbf{q}_i$, log-space scales, albedo $\rho_i$, metallic $\rho_{m,i}$, roughness $\alpha_i$ [S1, unofficial].
*   **Layer Data:** For each pixel $p$, a set of $K$ composited surface layers. Each layer $i$ has compositing weight $w_i$, mean depth $z_i$, and depth variance $\sigma_i^2$ [S1, unofficial].
*   **Neighbor Index:** A spatial data structure (e.g., KD-Tree) mapping surfel indices to nearest neighbors $\mathcal{N}_i$ with compatible normal orientations [S1, unofficial].

**Formulas:**
The total loss is a weighted sum:
$$
\mathcal{L} = \mathcal{L}_{\text{photo}} + \lambda_{\text{sil}} \mathcal{L}_{\text{sil}} + \lambda_{\text{cons}} \mathcal{L}_{\text{cons}} + \lambda_{\text{nc}} \mathcal{L}_{\text{nc}}
$$
where $\lambda$ terms are scalar weights [S1, unofficial].

1.  **Photometric Loss ($\mathcal{L}_{\text{photo}}$):** Combines L1 reconstruction and Structural Similarity (SSIM). For HDR targets, $\hat{\mathbf{I}}$ and $\mathbf{I}^*$ are transformed via log-space and sRGB transfer before loss computation to compress dynamic range [S1, unofficial].
    $$
    \mathcal{L}_{\text{photo}} = (1 - \lambda_{\text{ssim}}) \|\hat{\mathbf{I}} - \mathbf{I}^*\|_1 + \lambda_{\text{ssim}} (1 - \text{SSIM}(\hat{\mathbf{I}}, \mathbf{I}^*))
    $$
    where $\hat{\mathbf{I}}$ is the rendered image after tone mapping/gamma correction, $\mathbf{I}^*$ is the ground-truth image, and $\lambda_{\text{ssim}}$ is the weighting factor [S1, unofficial].

2.  **Silhouette Loss ($\mathcal{L}_{\text{sil}}$):** Supervises total accumulated coverage using soft Intersection-over-Union (IoU). Disabled once adaptive density control begins [S1, unofficial].
    $$
    \mathcal{L}_{\text{sil}} = \text{IoU}(1 - \bar{T}_{K+1}, \mathbf{M}^*)
    $$
    where $\bar{T}_{K+1}$ is the residual transmittance (silhouette mask) and $\mathbf{M}^*$ is the ground-truth mask [S1, unofficial].

3.  **Depth Consolidation Loss ($\mathcal{L}_{\text{cons}}$):** Penalizes spurious depth layers. Computed per pixel $p \in \mathcal{P}$ as the average of inter-layer and intra-layer terms [S1, unofficial].
    $$
    \mathcal{L}_{\text{cons}} = \frac{1}{|\mathcal{P}|} \sum_{p \in \mathcal{P}} (\mathcal{L}_{\text{inter-c}} + \mathcal{L}_{\text{intra-c}})
    $$
    *   **Inter-layer term:** Penalizes weight allocation across distant layers.
        $$
        \mathcal{L}_{\text{inter-c}} = \sum_{i < j} w_i w_j |z_i - z_j|
        $$
        where $w_i, w_j$ are compositing weights and $z_i, z_j$ are mean depths of layers $i, j$ [S1, unofficial].
    *   **Intra-layer term:** Penalizes depth spread within a layer.
        $$
        \mathcal{L}_{\text{intra-c}} = \sum_i w_i^2 \sigma_i^2
        $$
        where $w_i$ is the compositing weight and $\sigma_i^2$ is the depth variance of layer $i$ [S1, unofficial].

4.  **Normal Consistency Loss ($\mathcal{L}_{\text{nc}}$):** Penalizes angular deviation between a surfel's normal and its orientation-aware nearest neighbors in 3D [S1, unofficial].
    $$
    \mathcal{L}_{\text{nc}} = \frac{\sum_i \sum_{j \in \mathcal{N}_i} w_{ij} (1 - \mathbf{n}_i \cdot \mathbf{n}_j)}{\sum_i \sum_{j \in \mathcal{N}_i} w_{ij}}
    $$
    where $\mathcal{N}_i$ are nearest neighbors with compatible normal orientation, $\mathbf{n}_i, \mathbf{n}_j$ are normals, and $w_{ij} = \exp(-\|c_i - c_j\|^2 / 2\sigma^2)$ are distance-dependent weights derived from local intersample spacing [S1, unofficial].

**Parameters:**
*   $\lambda_{\text{ssim}}$: SSIM weighting factor.
*   $\lambda_{\text{sil}}, \lambda_{\text{cons}}, \lambda_{\text{nc}}$: Scalar weights for respective loss terms.
*   $\sigma$: Parameter for normal consistency weight decay, derived from local intersample spacing [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_joint_loss(rendered_img, gt_img, gt_mask, surfels, layers, 
                       lambda_ssim, lambda_sil, lambda_cons, lambda_nc, 
                       use_mask, training_stage):
    """
    Compute the total joint optimization loss for 3DSS inverse rendering.
    
    Args:
        rendered_img: Tensor of shape (H, W, 3), rendered image after tone mapping.
        gt_img: Tensor of shape (H, W, 3), ground-truth image (HDR).
        gt_mask: Tensor of shape (H, W), ground-truth foreground mask (optional).
        surfels: Object containing positions c, normals n, and other parameters.
        layers: List of layer data per pixel, each containing weights w, depths z, variances sigma2.
        lambda_ssim: Weight for SSIM term.
        lambda_sil: Weight for silhouette term.
        lambda_cons: Weight for depth consolidation term.
        lambda_nc: Weight for normal consistency term.
        use_mask: Boolean, whether gt_mask is available.
        training_stage: Enum or string indicating training phase (e.g., 'early', 'densification').
        
    Returns:
        total_loss: Scalar tensor.
    """
    # 1. Photometric Loss
    # Apply log-space and sRGB transfer to HDR targets if necessary before L1/SSIM
    # Assuming rendered_img and gt_img are already pre-processed or handled internally
    l1_loss = torch.mean(torch.abs(rendered_img - gt_img))
    ssim_loss = 1 - ssim(rendered_img, gt_img)
    l_photo = (1 - lambda_ssim) * l1_loss + lambda_ssim * ssim_loss
    
    total_loss = l_photo
    
    # 2. Silhouette Loss
    if use_mask and training_stage == 'early':
        # Residual transmittance is 1 - accumulated_opacity
        # Assuming accumulated_opacity is computed during forward pass
        accumulated_opacity = ... # Placeholder for forward pass output
        pred_silhouette = 1 - accumulated_opacity
        # Soft IoU implementation
        intersection = torch.sum(pred_silhouette * gt_mask)
        union = torch.sum(pred_silhouette) + torch.sum(gt_mask) - intersection
        l_sil = 1 - (intersection / (union + 1e-6))
        total_loss += lambda_sil * l_sil
        
    # 3. Depth Consolidation Loss
    l_cons_total = 0.0
    pixel_count = 0
    for p in range(num_pixels):
        pixel_layers = layers[p]
        l_inter = 0.0
        l_intra = 0.0
        
        # Inter-layer: sum over all pairs i < j
        K = len(pixel_layers)
        for i in range(K):
            for j in range(i + 1, K):
                w_i = pixel_layers[i].w
                w_j = pixel_layers[j].w
                z_i = pixel_layers[i].z
                z_j = pixel_layers[j].z
                l_inter += w_i * w_j * abs(z_i - z_j)
                
        # Intra-layer: sum over all layers
        for i in range(K):
            w_i = pixel_layers[i].w
            sigma2_i = pixel_layers[i].sigma2
            l_intra += (w_i ** 2) * sigma2_i
            
        l_cons_total += (l_inter + l_intra)
        pixel_count += 1
        
    if pixel_count > 0:
        l_cons = l_cons_total / pixel_count
        total_loss += lambda_cons * l_cons
        
    # 4. Normal Consistency Loss
    # Requires neighbor indices computed via spatial data structure
    neighbor_indices = surfels.get_neighbor_indices() # Rebuilt periodically
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
            dist_sq = torch.sum((c_i - c_j) ** 2)
            # sigma is derived from local intersample spacing [S1, unofficial]
            w_ij = torch.exp(-dist_sq / (2 * sigma_local ** 2))
            
            # Dot product for angle
            dot_prod = torch.dot(n_i, n_j)
            # Clamp for numerical stability
            dot_prod = torch.clamp(dot_prod, -1.0, 1.0)
            
            l_nc_num += w_ij * (1 - dot_prod)
            l_nc_den += w_ij
            
    if l_nc_den > 0:
        l_nc = l_nc_num / l_nc_den
        total_loss += lambda_nc * l_nc
        
    return total_loss
```

**Edge Cases and Pitfalls:**
*   **HDR Gradient Dominance:** Bright pixels can dominate gradients. Mitigation: Apply log-space and sRGB transfer to both rendered and ground-truth images before loss computation [S1, unofficial].
*   **Silhouette Activation:** The silhouette loss should be disabled once adaptive density control begins, as the photometric loss and densification mechanism take over coverage enforcement [S1, unofficial].
*   **Neighbor Index Cost:** Computing nearest neighbors is expensive. The spatial data structure must be rebuilt periodically and cached between updates to amortize cost [S1, unofficial].
*   **Sigma Derivation:** The $\sigma$ in the normal consistency weight $w_{ij}$ is derived from local intersample spacing. Exact calculation details are in section A.2 of the source [S1, unofficial].

## Operating characteristics
*   **Complexity:** The normal consistency loss requires $O(N \cdot k)$ operations where $N$ is the number of surfels and $k$ is the number of neighbors, with periodic rebuilds of the spatial index [S1, unofficial]. The depth consolidation loss is $O(P \cdot K^2)$ per iteration where $P$ is pixels and $K$ is layers per pixel.
*   **Performance:** Evaluated on object-level inverse rendering tasks including geometry reconstruction, novel view synthesis, and scene relighting [S1, unofficial].
*   **Strengths:** Enforces geometric thinness at two scales (inter-layer and intra-layer) and aligns surfel normals in 3D, avoiding oversmoothing associated with screen-space regularizers [S1, unofficial].
*   **Limitations:** Requires careful scheduling of loss terms (e.g., disabling silhouette loss later in training). Computationally heavier than pure photometric losses due to neighbor lookups and layer-wise operations.

## Background
Originated in the "3DSS: 3D Surface Splatting for Inverse Rendering" paper [S1, unofficial]. It builds on scattered-data approximation frameworks [Wendland 2004] and depth consolidation ideas from Barron et al. [2022] and Huang et al. [2024], adapting them to surfel primitives. It uses the Adam optimizer [Kingma and Ba 2014] for joint optimization of surfel parameters and HDR environment maps.

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
