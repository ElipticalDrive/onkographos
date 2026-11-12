# Total Loss Function

Also called: Combined Loss Function, Composite Training Loss, Full Training Objective, Multi-term Loss Function, Total Loss

## Overview
The Total Loss Function is the composite objective function used to optimize radiance fields and Gaussian splatting models end-to-end. It aggregates multiple loss terms—such as photometric reconstruction, geometric consistency, and regularization—to guide network parameters toward a solution that minimizes rendering error while enforcing structural priors [S1, S2, S3, S4, S5][S7, unofficial][S13]. Unlike single-term objectives, it balances competing goals (e.g., pixel accuracy vs. smoothness) via weighted hyperparameters [S1, S2, S3, S4, S5][S7, unofficial][S13]. It builds on [Regularization losses](1413-regularization-losses.md) by combining them with appearance and geometric terms.

## How it works
The total loss is a weighted sum of individual loss components. The specific components and weights vary by method, but the general structure is:

$$
\mathcal{L}_{\text{total}} = \sum_{k} w_k \mathcal{L}_k
$$

where $\mathcal{L}_k$ are individual loss terms (e.g., photometric, geometric, regularization) and $w_k$ are their respective scalar weights [S1, S2, S3, S4, S5][S7, unofficial][S13].

### Inputs and Outputs
**Inputs:**
*   Rendered image $\hat{\mathbf{I}}$ (or specific AOVs like throughput, depth, normals) [S1, S2, S3, S4, S5][S7, S8, unofficial][S13].
*   Ground truth image $\mathbf{I}^*$ (or GT depth, normals, masks) [S1, S2, S3, S4, S5][S7, S8, unofficial][S13].
*   Scene parameters (e.g., density $\sigma$, color $\mathbf{c}$, opacity $\alpha$, SDF $f$, Gaussian attributes) [S1, S2, S3, S4, S5][S7, unofficial][S13].
*   Optional auxiliary data: sparse depth, masks, background color [S1, S5][S8, S10, unofficial].

**Outputs:**
*   Scalar loss value $\mathcal{L}_{\text{total}}$ [S1, S2, S3, S4, S5][S7, unofficial][S13].

### Data Structures
*   **Loss Weights:** Dictionary or array mapping loss term names to float weights [S1, S2, S3, S4, S5][S7, unofficial][S13].
*   **Rendered AOVs:** Dictionary containing tensors for RGB, throughput, depth, etc. [S8, S10, unofficial].
*   **Masks:** Binary tensor (0 or 1) for object/background regions [S8, S10, unofficial].

### Numbered Steps
1.  **Compute Photometric Loss:** Calculate the difference between rendered and ground truth images. Common metrics include L1, L2, SSIM, or Charbonnier loss [S1, S2, S3, S4, S5, S6][S7, unofficial][S13].
2.  **Compute Geometric/Regularization Losses:** Calculate terms such as Eikonal loss, curvature loss, sparsity loss, opacity loss, or total variation [S1, S3, S4][S7, unofficial].
3.  **Compute Auxiliary Losses:** If applicable, calculate losses for distractors, background, or specific AOVs like throughput [S8, unofficial][S13].
4.  **Apply Weights:** Multiply each loss term by its corresponding weight [S1, S2, S3, S4, S5][S7, unofficial][S13].
5.  **Sum Losses:** Sum all weighted terms to get the total loss [S1, S2, S3, S4, S5][S7, unofficial][S13].
6.  **Backpropagate:** Compute gradients of the total loss with respect to all trainable parameters [S1, S2, S3, S4, S5][S7, unofficial][S13].

### Formulas
The specific formulation depends on the method. Common components include:

**Photometric Loss (L1):**
$$
\mathcal{L}_{\text{rgb}} = \frac{1}{N} \sum_{i=1}^N |C_i - \hat{C}_i|
$$
where $N$ is the number of batch rays, $C_i$ is ground truth color, and $\hat{C}_i$ is rendered color [S7, unofficial].

**Eikonal Loss:**
$$
\mathcal{L}_{\text{eik}} = \frac{1}{MN} \sum_{i,j=1}^{MN} \|\nabla f(p_{i,j}) - 1\|^2
$$
where $M$ is sampled locations per ray, $N$ is batch rays, and $f$ is the SDF [S7, unofficial].

**Sparsity Loss:**
$$
\mathcal{L}_{\text{sparse}} = \frac{1}{|\mathcal{P}|} \sum_{p_i \in \mathcal{P}} |\alpha_i|
$$
where $\mathcal{P}$ is a set of uniformly sampled points and $\alpha_i$ is opacity [S1].

**Opacity Loss:**
$$
\mathcal{L}_{\text{opacity}} = -\sum_k (\alpha^k \log(\alpha^k) + (1-\alpha^k) \log(1-\alpha^k))
$$
Encourages opacity to be 0 or 1 [S1].

**Total Loss Examples:**
*   **Neural Volumes [S3]:**
    $$
    \ell(\theta) = \frac{1}{P} \sum_{\mathbf{p}} \left\| \hat{\mathbf{I}}_{\text{rgb}}(\mathbf{p}) - \mathbf{I}_{\text{rgb}}^*(\mathbf{p}) \right\|^2 + \lambda_{\text{KL}} D_{\text{KL}}(\mathbf{z} \parallel \mathcal{N}(0, 1)) + J_{\text{tv}}(\mathbf{V}_\alpha) + J_{\text{B}}(\mathbf{I}_\alpha)
    $$
*   **FWD [S5]:**
    $$
    L = \lambda_{l2} L_{l2} + \lambda_c L_c
    $$
*   **Neuralangelo [S4]:**
    $$
    \mathcal{L} = \mathcal{L}_{\text{RGB}} + w_{\text{eik}} \mathcal{L}_{\text{eik}} + w_{\text{curv}} \mathcal{L}_{\text{curv}}
    $$
*   **City-on-Web [S1]:**
    $$
    \mathcal{L}_{\text{train}} = \mathcal{L}_{\text{cb}} + \mathcal{L}_{\text{global}} + \lambda_1 \mathcal{L}_{\text{s3im}} + \lambda_2 \mathcal{L}_{\text{prop}} + \lambda_3 \mathcal{L}_{\text{dist}} + \lambda_4 \mathcal{L}_{\text{s}} + \lambda_5 \mathcal{L}_{\text{opacity}}
    $$
*   **DeSplat [S13]:**
    $$
    L = L_{\text{GS}} + \lambda_d \alpha_d + \lambda_{\text{bg}} L_{\text{bg}}
    $$

### Parameters
*   **Weights:** $\lambda$ or $w$ values are method-specific. Examples: $\lambda_{\text{KL}}=0.001$ [S3], $\lambda_{l2}=5.0$ [S5], $\alpha=5$ (real) or $1$ (synthetic) [S2], $\beta=0.0008$ [S2], $\lambda_d=0.01$, $\lambda_{\text{bg}}=0.15$ [S13].
*   **Optimizer:** Typically Adam [S3, S5, S13].
*   **Learning Rate:** Often $10^{-4}$ [S3, S5].
*   **Batch Size:** e.g., 16 [S3].
*   **Iterations:** e.g., 500,000 [S3], 30,000 [S13].

### Edge Cases and Pitfalls
*   **Background Blending:** If the background is not black, loss computation may require blending predicted RGB with GT background color [S10, unofficial].
*   **Curriculum Learning:** Transitioning from depth/photometric to RGB supervision can be controlled by an exponential schedule [S11].
*   **Stability:** Volume regularization terms (e.g., from Scaffold-GS) may be kept for stability [S6].
*   **Precision:** Step sizes in normalized voxel space $[-1, 1]^3$ help with saturation at initialization [S3].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_total_loss(rendered_outputs, gt_data, weights, config):
    """
    Compute the total loss for radiance field optimization.
    
    Args:
        rendered_outputs: Dict containing 'rgb', 'throughput', 'depth', etc.
        gt_data: Dict containing 'rgb', 'mask', 'depth', etc.
        weights: Dict of loss weights (e.g., {'l1': 1.0, 'eik': 0.1})
        config: Config object with hyperparameters
        
    Returns:
        total_loss: Scalar tensor
    """
    total_loss = 0.0
    
    # 1. Photometric Loss
    if 'l1' in weights:
        l1_loss = l1_loss_fn(rendered_outputs['rgb'], gt_data['rgb'])
        total_loss += weights['l1'] * l1_loss
        
    if 'ssim' in weights:
        ssim_loss = ssim_loss_fn(rendered_outputs['rgb'], gt_data['rgb'])
        total_loss += weights['ssim'] * ssim_loss
        
    # 2. Geometric/Regularization Losses
    if 'eik' in weights:
        # Compute gradient of SDF field
        sdf_grad = compute_sdf_gradient(rendered_outputs['sdf'])
        eik_loss = eikonal_loss_fn(sdf_grad)
        total_loss += weights['eik'] * eik_loss
        
    if 'opacity' in weights:
        opacity = rendered_outputs['opacity']
        opacity_loss = opacity_loss_fn(opacity)
        total_loss += weights['opacity'] * opacity_loss
        
    if 'sparse' in weights:
        # Sample points in bounding box
        sampled_points = sample_uniform_points(config.bounding_box)
        sampled_opacity = query_opacity(sampled_points)
        sparse_loss = sparsity_loss_fn(sampled_opacity)
        total_loss += weights['sparse'] * sparse_loss
        
    # 3. Auxiliary Losses (e.g., AOVs)
    if 'throughput' in rendered_outputs and 'mask' in gt_data:
        throughput = rendered_outputs['throughput']
        mask = gt_data['mask']
        # Empty Space Loss: encourage throughput=0 where mask=0
        empty_space_loss = mean((1 - mask) * abs(throughput))
        total_loss += weights.get('empty_space', 0) * empty_space_loss
        
        # Opacity Loss: encourage throughput=1 where mask=1
        opacity_aov_loss = mean(mask * abs(throughput - 1.0))
        total_loss += weights.get('opacity_aov', 0) * opacity_aov_loss
        
    # 4. Curriculum Learning (optional)
    if config.use_curriculum:
        t = config.current_iteration
        beta = config.curriculum_beta
        photo_weight = 2 ** (beta * t)
        rgb_weight = 1 - photo_weight
        # Adjust weights dynamically if needed
        pass
        
    return total_loss

def l1_loss_fn(rendered, gt):
    return mean(abs(rendered - gt))

def ssim_loss_fn(rendered, gt):
    # Use standard SSIM implementation
    return 1 - ssim(rendered, gt)

def eikonal_loss_fn(sdf_grad):
    # ||grad(f) - 1||^2
    return mean((sdf_grad - 1.0)**2)

def opacity_loss_fn(opacity):
    # -sum(alpha*log(alpha) + (1-alpha)*log(1-alpha))
    # Add small epsilon to avoid log(0)
    eps = 1e-7
    alpha = clamp(opacity, eps, 1-eps)
    return -mean(alpha * log(alpha) + (1-alpha) * log(1-alpha))

def sparsity_loss_fn(opacity):
    return mean(abs(opacity))
```

## Subtopics

### [Loss Function Configuration](6782-loss-function-configuration.md)
Defines the mathematical structure and hyperparameter weights of the total objective. Focuses on composition and weighting coefficients.

### [Total Variation Loss](7047-total-variation-loss.md)
Regularization constraint enforcing spatial smoothness in latent representations.

### [Sparse Depth Consistency Loss](7294-sparse-depth-consistency-loss.md)
Enforces geometric consistency between rendered depth and sparse SfM depth estimates.

### [Discriminator Loss with Gradient Penalty](7695-discriminator-loss-with-gradient-penalty.md)
GAN discriminator loss enforcing Lipschitz continuity via R1 penalty.

### [Loss Weight Configuration](7883-loss-weight-configuration.md)
Determines scalar coefficients to balance heterogeneous error terms.

### [Photometric and Image Reconstruction Losses](9789-photometric-and-image-reconstruction-losses.md)
Minimizes pixel-wise difference between rendered and GT images (L1, SSIM, perceptual).

### [Specific Method Implementations](9794-specific-method-implementations.md)
Concrete loss formulations for specific architectures (NeRF, GS, voxels).

### [Numerical Gradient Eikonal Loss](9954-numerical-gradient-eikonal-loss.md)
Enforces eikonal equation via numerical finite differences for hash-encoded SDFs.

### [Overall Training Loss](10621-overall-training-loss.md)
Composite objective aggregating photometric, geometric, and regularization terms.

### [Multi-scale Joint Training Loss](13064-multi-scale-joint-training-loss.md)
Balances low-frequency structure and high-frequency detail across multiple resolution stages.

### Empty Space Loss
An AOV loss encouraging rendering throughput to equal 0.0 in regions where a binary object mask is inactive [S8, unofficial]. Formula: $L = w \cdot \frac{1}{N} \sum_{i=1}^{N} (1 - m_i) \cdot |T_i - 0|$, where $w$ is weight (default 1.0), $N$ is samples, $m_i$ is mask (0/1), $T_i$ is throughput [S8, unofficial].

### Opacity Loss
An AOV loss encouraging rendering throughput to equal 1.0 in regions where a binary object mask is active [S8, unofficial]. Formula: $L = w \cdot \frac{1}{N} \sum_{i=1}^{N} m_i \cdot |T_i - 1|$, where $w$ is weight (default 1.0), $N$ is samples, $m_i$ is mask (0/1), $T_i$ is throughput [S8, unofficial].

### Auxiliary Losses
Per-point RGB loss supervises color emission of each query point directly [S9]. Background entropy loss enforces rendered background probability to concentrate on foreground/background [S9]. Adding both yields PSNR 35.54 vs 35.37 (none) or 35.53 (one loss) [S9].

### Background Blending for Loss Computation
Blends predicted RGB with GT background color for accurate loss computation when background is not black [S10, unofficial].

### Curriculum Learning Loss
Transitions from depth/photometric supervision to RGB supervision during training [S11]. Formula: $\mathcal{L}_{\text{final}} = 2^{\beta \cdot t} (\mathcal{L}_{\text{depth}} + \mathcal{L}_{\text{photo}}) + (1-2^{\beta \cdot t}) \mathcal{L}_{\text{rgb}}$, where $\beta = -1e5$ and $t$ is iteration [S11].

### Softmax Weighted Volume Rendering
Accumulates colors along a ray weighted by densities after applying softmax to speed up convergence [S12]. Total loss is sum of contrastive and color losses [S12].

### DeSplat Loss Function
Combines standard GS loss, distractor regularization, and background regularization [S13]. Formula: $L = L_{\text{GS}} + \lambda_d \alpha_d + \lambda_{\text{bg}} L_{\text{bg}}$. Weights: $\lambda_d=0.01$, $\lambda_{\text{bg}}=0.15$ (or 0 for ablation) [S13].

### Energy Function for Optimization
$E(c) = H(c) + C(c)$, where $H(c)$ is coherence metric and $C(c)$ is context metric [S14]. Optimization ends if $E(c) < 1$ [S14].

## Operating characteristics
*   **Complexity:** Depends on the number of loss terms and their computational cost (e.g., SSIM is more expensive than L1). Memory usage is dominated by storing intermediate gradients for backpropagation [44-radiance-field-optimization.md].
*   **Performance:** Training can take significant time (e.g., 10 days on V100 for 500k iterations [S3], 3 days on 4x2080Ti [S5]).
*   **Strengths:** Allows flexible combination of appearance, geometry, and regularization priors. Can handle diverse inputs (RGB, depth, masks).
*   **Limitations:** Requires careful tuning of multiple hyperparameters (weights, learning rates). Can be sensitive to initialization and sampling strategies.
*   **When to use:** When optimizing radiance fields or Gaussian splatting models end-to-end.
*   **When not to use:** When only single-term objectives are sufficient or when computational resources are extremely limited.

## Background
The concept of combining multiple loss terms for optimizing neural radiance fields and related representations has evolved from early NeRF works [S3] to more complex formulations involving Gaussian Splatting [S2, S13] and implicit surfaces [S4][S7, unofficial]. Key developments include the introduction of Eikonal loss for SDFs [S4][S7, unofficial], sparsity and opacity regularization [S1], and specialized losses for distractors and background [S13]. The method is widely used in modern differentiable volume rendering pipelines.

## Sources
[S1] City-on-Web: Real-Time Neural Rendering of Large-Scale Scenes on the Web, https://doi.org/10.1007/978-3-031-72970-6_22, fetched 2026-10-26
[S2] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S3] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
[S4] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S5] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
[S6] SOGS: Second-Order Anchor for Advanced 3D Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.01043, fetched 2026-10-30
[S7] Point-NeuS: Point-Guided Neural Implicit Surface Reconstruction by Volume Rendering, https://doi.org/10.48550/arXiv.2310.07997, fetched 2026-10-30 (unofficial)
[S8] JackZhouSz/tlir, https://github.com/JackZhouSz/tlir, fetched 2026-10-27 (unofficial)
[S9] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S10] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S11] DBARF: Deep Bundle-Adjusting Generalizable Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00011, fetched 2026-10-29
[S12] ContraNeRF: Generalizable Neural Radiance Fields for Synthetic-to-real Novel View Synthesis via Contrastive Learning, https://doi.org/10.1109/cvpr52729.2023.01584, fetched 2026-10-29
[S13] DeSplat: Decomposed Gaussian Splatting for Distractor-Free Rendering, https://doi.org/10.1109/CVPR52734.2025.00076, fetched 2026-11-03
[S14] Visibility-Aware Direct Volume Rendering, https://doi.org/10.1007/s11390-011-9428-3, fetched 2026-10-26
