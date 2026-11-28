# Composite Training Loss

Also called: Composite Training Loss & Schedule, L1+SSIM+LPIPS Loss, LiteVoxel Training Objective

## Overview
Composite training loss aggregates multiple error metrics to optimize radiance field or inverse rendering parameters by combining photometric, structural, and perceptual terms [S3, S4, S5]. It addresses the ill-posed nature of inverse rendering by balancing reconstruction accuracy with high-frequency detail preservation and geometric/material priors [S1, unofficial][S4].

## How it works
### Inputs and Outputs
- **Inputs**: Rendered image $I_r$, ground truth image $I_{gt}$, and optionally auxiliary data such as estimated normals, semantic maps, shadow masks, or depth maps [S3, S4].
- **Outputs**: A scalar loss value $\mathcal{L}$ used for gradient-based optimization [S2, unofficial].

### Data Structures
Standard image tensors and scalar weights. Some variants require pre-computed auxiliary maps (e.g., normals from Omnidata, semantic maps from MMSegmentation) [S4].

### Steps
1. **Render**: Generate the image $I_r$ from the current model parameters [S3].
2. **Compute Auxiliary Priors (Variant-dependent)**: If required, estimate normals, semantics, or shadow masks using pretrained networks [S4].
3. **Compute Individual Loss Terms**: Calculate each component loss (L1, SSIM, LPIPS, robust loss, etc.) between $I_r$ and $I_{gt}$ (or priors) [S3, S5].
4. **Aggregate**: Sum the weighted loss terms to produce the total loss $\mathcal{L}$ [S2, unofficial].

### Formulas
The specific formula depends on the variant:

**General Composite (MuRF):**
$$ \mathcal{L} = \mathcal{L}_{L1} + \mathcal{L}_{SSIM} + \mathcal{L}_{LPIPS} $$
Where $\mathcal{L}_{L1}$ is the mean absolute error, $\mathcal{L}_{SSIM}$ is the structural similarity index loss, and $\mathcal{L}_{LPIPS}$ is the learned perceptual image patch similarity loss [S3].

**latentSplat:**
$$ \mathcal{L}_{\mathrm{rec}} = \lambda_1\|\hat{\mathbf{T}}-\mathbf{T}\|_1 + \lambda_2 \mathrm{LPIPS}(\hat{\mathbf{T}},\mathbf{T}) $$
$$ \mathcal{L}_{\mathrm{aux}} = \lambda_3\|\hat{\mathbf{T}}_{\mathrm{aux}}-\mathbf{T}\|_2^2 + \lambda_4 \mathrm{LPIPS}(\hat{\mathbf{T}}_{\mathrm{aux}},\mathbf{T}) $$
$$ \mathcal{L}_{\mathrm{gen}}=E_{\hat{\mathbf{T}}} [\log (1- D(\hat{\mathbf{T}}))], \quad \mathcal{L}_{\mathrm{disc}} = E_{\mathbf{X}}[\log (D(\mathbf{X}))] + E_{\hat{\mathbf{T}}}[\log (1-D(\hat{\mathbf{T}}))] $$
Where $\hat{\mathbf{T}}$ is the decoded feature image, $\mathbf{T}$ is the target image, $\hat{\mathbf{T}}_{\mathrm{aux}}$ is the color rendering, $D$ is a pre-trained discriminator from LDM, and $\mathbf{X}$ are real images [S2, unofficial].

**UrbanIR:**
$$ \min _{\theta , \mathbf{L}} \mathcal {L}_{\text {render}} + \lambda _1\mathcal {L}_{\text {visibility}} + \lambda _2\mathcal {L}_{\text {normal}} + \lambda _3\mathcal {L}_{\text {semantics}} + \lambda _4\mathcal {L}_{\text {reg}} $$
Where $\mathcal{L}_{\text{render}}$ is the base rendering loss, $\mathcal{L}_{\text{visibility}}$ enforces visibility consistency, $\mathcal{L}_{\text{normal}}$ aligns with estimated normals, $\mathcal{L}_{\text{semantics}}$ aligns with semantic maps, and $\mathcal{L}_{\text{reg}}$ is a regularization term [S4].

**SVG-IR / LiteVoxel:**
$$ \mathcal{L} = \lambda_{SSIM}\mathcal{L}_{SSIM} + \lambda_{Tc}\mathcal{L}_{Tc} + \lambda_{Ti}\mathcal{L}_{Ti} + \lambda_{NDM}\mathcal{L}_{NDM} + \mathcal{L}_{mask} + \mathcal{L}_{tv} $$
Where $\mathcal{L}_{SSIM}$ is a structure-aware penalty, $\mathcal{L}_{Tc}$ encourages final ray transmittance near 0 or 1, $\mathcal{L}_{Ti}$ discourages floating density, $\lambda_{NDM}\mathcal{L}_{NDM}$ is median-based depth-normal consistency, $\mathcal{L}_{mask}$ is foreground/background supervision, and $\mathcal{L}_{tv}$ is total variation on the sparse density field [S1, unofficial].

**INPC:**
$$ L = L_R + L_{D-SSIM} + \lambda_{vgg} \cdot L_{VGG} + \lambda_{decay} \cdot L_{Reg} $$
Where $L_R$ is a robust per-pixel loss, $L_{D-SSIM}$ is differentiable SSIM, $L_{VGG}$ is VGG perceptual loss, and $L_{Reg}$ is normalized weight decay on hash grid parameters [S5].

**LINGO:**
$$ L_{total} = L_1(I_r, I_m) + \lambda_{ssim} L_{ssim}(I_r, I_m) + \lambda_{tv} L_{tv}(V_{tv}) $$
$$ T_g = \max(0.01, \mathrm{Otsu}(I_r)) $$
$$ L_{final} = L_{total} \times \left(T_g + \frac{1}{4T_g}\right) $$
Where $I_r$ and $I_m$ are rendered and ground-truth images, $V_{tv}$ represents the density volume, and $T_g$ is a dynamic loss-scaling coefficient [S6, unofficial].

**Water Gaussians:**
$$ \mathcal{L}_{RGB} = 0.8 \cdot \frac{\|I_r - I_{gt}\|_2^2}{\mathrm{stopgrad}(b_I) + 10^{-3}}, \quad \mathcal{L}_{Struct} = 0.2 \cdot (1 - \mathrm{SSIM}(I_r, I_{gt})) $$
$$ \mathcal{L}_{TV} = 10^{-4} \cdot \left(\text{mean absolute first differences along axes} + \text{within-cell deviation of FCC sites}\right) $$
The RGB and structural terms supervise captured underwater RGB, while the TV penalty regularizes water densities [S7, unofficial].

**Pre-training:**
$$ L = L_{color} + \lambda \cdot L_{lpips} $$
$$ L_{color} = \frac{1}{H \times W} \sum_{i=1}^{H \times W} (I_r(i) - I_{gt}(i))^2 $$
$$ L_{lpips} = \sum_{l} \frac{1}{M_l} \sum_{i=1}^{M_l} \| w_l \odot (\hat{f}_l(I_r(i)) - \hat{f}_l(I_{gt}(i))) \|_2^2 $$
Where $H \times W$ is the pixel count, $\hat{f}_l$ is the normalized VGG feature map at layer $l$, and $w_l$ are channel-wise weights [S8, unofficial].

### Parameters
- **latentSplat**: $\lambda_1 = 1$, $\lambda_2 = 1$, $\lambda_3 = 10$, $\lambda_4 = 0.5$ [S2, unofficial].
- **UrbanIR**: $\lambda_1 = 0.001$, $\lambda_2 = 0.01$, $\lambda_3 = 0.04$, $\lambda_4 = 0.1$ [S4].
- **INPC**: $\lambda_{vgg} = 0.075$, $\lambda_{decay} = 0.1$, robust loss parameters $\alpha = 0$, $c = 0.2$ [S5].
- **LINGO**: $\lambda_{ssim} = 0.25$, $\lambda_{tv} = 0.05$ [S6, unofficial].
- **Water Gaussians**: RGB/Structural weights 0.8/0.2, TV penalty $10^{-4}$, Adam learning rates $2.5 \times 10^{-3}$ (transport-density) and $10^{-3}$ (directional head), $\epsilon = 10^{-15}$, directional head decays to $1.5 \times 10^{-4}$ over 15k steps [S7, unofficial].
- **Pre-training**: $\lambda = 0.05$ [S8, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def l1_loss(pred, target):
    return np.mean(np.abs(pred - target))

def mse_loss(pred, target):
    return np.mean((pred - target) ** 2)

def ssim_loss(pred, target):
    # Simplified SSIM computation for demonstration
    mu_p = np.mean(pred)
    mu_t = np.mean(target)
    sigma_p = np.var(pred)
    sigma_t = np.var(target)
    sigma_pt = np.mean((pred - mu_p) * (target - mu_t))
    C1, C2 = 1e-4, 1e-4
    ssim = ((2 * mu_p * mu_t + C1) * (2 * sigma_pt + C2)) / \
           ((mu_p**2 + mu_t**2 + C1) * (sigma_p + sigma_t + C2))
    return 1.0 - ssim

def lpips_loss(pred, target):
    # Placeholder for perceptual network forward pass
    return np.mean((pred - target) ** 2)

def robust_loss(pred, target, alpha=0.0, c=0.2):
    # Generalized Charbonnier/robust loss
    diff = pred - target
    return np.mean(np.sqrt(diff**2 + c**2))

def vgg_loss(pred, target):
    # Placeholder for VGG feature distance
    return np.mean((pred - target) ** 2)

def normalized_weight_decay(params):
    return np.mean(params ** 2)

def otsu_threshold(image):
    # Otsu's method for binary thresholding
    hist, bins = np.histogram(image.flatten(), bins=256, range=(0, 1))
    total = image.size
    sum_f = np.dot(bins[:-1], hist)
    sum_total = np.dot(bins[:-1], hist)
    weight_f1 = np.cumsum(hist)
    weight_f2 = total - weight_f1
    mean_f1 = np.cumsum(hist * bins[:-1]) / weight_f1
    mean_f2 = (sum_total - np.cumsum(hist * bins[:-1])) / weight_f2
    variance = weight_f1 * weight_f2 * (mean_f1 - mean_f2) ** 2
    return np.argmax(variance) / 255.0

def total_variation(volume):
    # Mean absolute first differences along axes
    tv_x = np.mean(np.abs(np.diff(volume, axis=0)))
    tv_y = np.mean(np.abs(np.diff(volume, axis=1)))
    tv_z = np.mean(np.abs(np.diff(volume, axis=2)))
    return (tv_x + tv_y + tv_z) / 3.0

def cosine_similarity(a, b):
    return np.sum(a * b) / (np.linalg.norm(a) * np.linalg.norm(b) + 1e-8)

def compute_composite_loss(rendered_image, ground_truth, variant="standard", aux_data=None, params=None):
    """
    Computes the composite training loss based on the specified variant.
    """
    if params is None:
        params = {}
        
    if variant == "standard":
        return l1_loss(rendered_image, ground_truth) + ssim_loss(rendered_image, ground_truth) + lpips_loss(rendered_image, ground_truth)

    elif variant == "latentSplat":
        lambda1 = params.get("lambda1", 1.0)
        lambda2 = params.get("lambda2", 1.0)
        lambda3 = params.get("lambda3", 10.0)
        lambda4 = params.get("lambda4", 0.5)
        rec = lambda1 * l1_loss(rendered_image, ground_truth) + lambda2 * lpips_loss(rendered_image, ground_truth)
        aux = lambda3 * mse_loss(rendered_image, ground_truth) + lambda4 * lpips_loss(rendered_image, ground_truth)
        return rec + aux

    elif variant == "urbanir":
        lambda1 = params.get("lambda1", 0.001)
        lambda2 = params.get("lambda2", 0.01)
        lambda3 = params.get("lambda3", 0.04)
        lambda4 = params.get("lambda4", 0.1)
        loss_render = l1_loss(rendered_image, ground_truth)
        loss_vis = l1_loss(rendered_image, ground_truth) # Visibility proxy
        loss_norm = 1.0 - cosine_similarity(rendered_image, aux_data.get("normals", rendered_image))
        loss_sem = l1_loss(rendered_image, aux_data.get("semantics", rendered_image))
        loss_reg = normalized_weight_decay(rendered_image)
        return loss_render + lambda1 * loss_vis + lambda2 * loss_norm + lambda3 * loss_sem + lambda4 * loss_reg

    elif variant == "svgir":
        # LiteVoxel components: SSIM, Tc, Ti, NDM, mask, TV
        loss_ssim = ssim_loss(rendered_image, ground_truth)
        loss_tv = total_variation(rendered_image)
        return loss_ssim + loss_tv

    elif variant == "inpc":
        lambda_vgg = params.get("lambda_vgg", 0.075)
        lambda_decay = params.get("lambda_decay", 0.1)
        alpha = params.get("alpha", 0.0)
        c = params.get("c", 0.2)
        loss_r = robust_loss(rendered_image, ground_truth, alpha=alpha, c=c)
        loss_dssim = ssim_loss(rendered_image, ground_truth)
        loss_vgg = vgg_loss(rendered_image, ground_truth)
        loss_reg = normalized_weight_decay(rendered_image)
        return loss_r + loss_dssim + lambda_vgg * loss_vgg + lambda_decay * loss_reg

    elif variant == "lingo":
        lambda_ssim = params.get("lambda_ssim", 0.25)
        lambda_tv = params.get("lambda_tv", 0.05)
        l_total = l1_loss(rendered_image, ground_truth) + lambda_ssim * ssim_loss(rendered_image, ground_truth) + lambda_tv * total_variation(rendered_image)
        Tg = max(0.01, otsu_threshold(rendered_image))
        return l_total * (Tg + 1.0 / (4.0 * Tg))

    elif variant == "water":
        b_I = 1.0 # stopgrad placeholder
        rgb_term = 0.8 * mse_loss(rendered_image, ground_truth) / (b_I + 1e-3)
        struct_term = 0.2 * ssim_loss(rendered_image, ground_truth)
        tv_term = 1e-4 * total_variation(rendered_image)
        return rgb_term + struct_term + tv_term

    elif variant == "pretrain":
        lambda_lpips = params.get("lambda_lpips", 0.05)
        l_color = mse_loss(rendered_image, ground_truth)
        l_lpips = lpips_loss(rendered_image, ground_truth)
        return l_color + lambda_lpips * l_lpips

    else:
        raise ValueError("Unknown variant")
```

### Edge Cases and Pitfalls
- **Staged Training**: latentSplat trains with auxiliary losses only for the first 100k iterations to prevent features from fitting to incorrectly positioned Gaussians, and delays generative loss until 125k iterations to prevent an overly strong discriminator [S2, unofficial].
- **Dynamic Scaling**: LINGO uses Otsu's threshold on the rendered image to compute a dynamic loss-scaling coefficient, strengthening gradients in low-density areas without causing gradient explosion [S6, unofficial].
- **Auxiliary Dependency**: UrbanIR requires pretrained 2D priors (MTMT for shadows, Omnidata for normals, MMSegmentation for semantics) which must be available or estimated before optimization [S4].
- **Initialization & Precision**: Water Gaussians initializes direct/backscatter densities and directional heads with specific values and uses $\epsilon = 10^{-15}$ for Adam stability [S7, unofficial].

## Subtopics

### UrbanIR Training Objective
An optimization objective combining rendering loss, visibility, normal, semantic, and regularization losses, using pretrained 2D priors to address the ill-posed inverse problem [S4]. It estimates shadow masks using MTMT, refines geometry using Omnidata, and uses semantic maps from Kitti360 or MMSegmentation [S4]. It is chosen for large-scale urban scene inverse rendering from single videos [S4].

### [SVG-IR Training Loss](17832-svg-ir-training-loss.md)
A composite objective function used to optimize the Spatially-Varying Gaussian Splatting (SVG-IR) representation for inverse rendering. It solves the problem of jointly optimizing geometric, photometric, and material properties from multi-view images by combining data fidelity terms with regularization constraints for normals, albedo, and roughness. It is distinguished from standard Gaussian Splatting optimization by the inclusion of physically-based inverse rendering losses, specifically radiance consistency and surface normal regularization, and differs from implicit neural representation methods like IGR by using a discrete Gaussian primitive set rather than a continuous signed distance function.

### INPC Optimization Loss
A composite training loss combining a robust per-pixel loss, D-SSIM, VGG perceptual loss, and normalized weight decay on multi-resolution hash grid parameters [S5]. It is used for Implicit Neural Point Clouds, balancing perceptual quality with regularization of the hash grid [S5]. It is chosen for high-quality radiance field rendering with fixed model size [S5].

### latentSplat Training Losses
A staged training objective combining auxiliary, reconstruction, and generative losses with specific weights and adaptive scaling [S2, unofficial]. It uses L1 and LPIPS for reconstruction, MSE and LPIPS for auxiliary color rendering, and VAE-GAN losses adapted from LDM for generative sampling [S2, unofficial]. It is chosen when correct sampling and generation in uncertain regions is required [S2, unofficial].

### LINGO Dynamic Gradient Optimization
A dynamic loss-scaling strategy that computes total loss with L1, SSIM, and TV regularization, then scales it using Otsu's threshold on the rendered image to address vanishing gradients in low-density structural regions [S6, unofficial]. It is chosen for sparse-view X-ray novel view synthesis and CT reconstruction with 3D Gaussian Splatting [S6, unofficial].

### Water Gaussians Composite Loss
A convex combination of RGB and structural terms (weights 0.8 and 0.2) with a spatial total-variation penalty on water densities [S7, unofficial]. The RGB term is mean squared residual after division by $\mathrm{stopgrad}(b_I) + 10^{-3}$, and the structural term is $1 - \mathrm{SSIM}$ after the same normalization [S7, unofficial]. It is chosen for per-scene underwater rendering with homogeneous-grid optical-mass identity [S7, unofficial].

### Pre-training Loss Functions
A weighted sum of image color loss (MSE) and LPIPS loss used to pre-train a point cloud encoder via 3D Gaussian Splatting [S8, unofficial]. It encourages the encoder to capture precise geometric and spatial information for downstream tasks [S8, unofficial]. It is chosen for unsupervised pre-training of 3D scene understanding models [S8, unofficial].

## Operating characteristics
- **Time Complexity**: Depends on the number of loss terms and the complexity of auxiliary networks (e.g., VGG, Omnidata). latentSplat requires staged training over 125k+ iterations [S2, unofficial]. UrbanIR requires 100 epochs with Adam optimizer [S4].
- **Memory Complexity**: INPC uses a fixed-size hash grid (1.1 GB) [S5]. UrbanIR uses Instant-NGP style hash tables [S5]. Water Gaussians uses a compact configuration with $N=8$ (2,048 fixed water Gaussians, 12,288 learned RGB transport parameters) [S7, unofficial].
- **Performance**: INPC achieves high PSNR and LPIPS scores on MipNeRF360 and Tanks and Temples datasets [S5]. latentSplat improves on prior methods with view-synthesis benchmarks [S2, unofficial]. LINGO reaches performance comparable to baselines trained with 30k iterations within only 5k iterations under sparse-view conditions [S6, unofficial].
- **Hardware**: INPC tested on RTX 4090 and A100 [S5].
- **Strengths**: Combines multiple metrics for robust optimization. Handles ill-posed problems via priors (UrbanIR). Dynamic scaling accelerates convergence in challenging regions (LINGO).
- **Limitations**: Requires careful tuning of weights. Auxiliary networks add complexity and dependency. Staged training increases total optimization time (latentSplat).

## Background
Composite training losses are standard in differentiable rendering and inverse rendering to balance photometric accuracy with perceptual quality. The L1+SSIM+LPIPS combination is a common baseline [S3]. latentSplat adapts LDM VAE-GAN decoder losses for radiance field optimization [S2, unofficial]. UrbanIR extends this with semantic and normal priors for urban scenes [S4]. SVG-IR adds radiance consistency and material-specific losses for Gaussian splatting [S1, unofficial]. INPC uses robust losses and hash grid regularization for neural point clouds [S5]. LINGO addresses sparse-view X-ray synthesis with dynamic gradient optimization [S6, unofficial]. Water Gaussians handles underwater optical-mass identity with specialized TV penalties [S7, unofficial]. Pre-training objectives use 3DGS for point cloud encoder initialization [S8, unofficial].

## Sources
[S1] LiteVoxel: Low-memory Intelligent Thresholding for Efficient Voxel Rasterization, https://doi.org/10.48550/arxiv.2511.02510, fetched 2026-10-26 (unofficial)
[S2] latentSplat, https://arxiv.org/abs/2403.16292, fetched 2026-10-27 (unofficial)
[S3] MuRF: Multi-Baseline Radiance Fields, https://doi.org/10.1109/cvpr52733.2024.01894, fetched 2026-10-27
[S4] UrbanIR: Large-Scale Urban Scene Inverse Rendering from a Single Video, https://doi.org/10.1109/3dv66043.2025.00052, fetched 2026-10-30
[S5] INPC: Implicit Neural Point Clouds for Radiance Field Rendering, https://doi.org/10.1109/3dv66043.2025.00021, fetched 2026-10-26
[S6] LINGO: Latent Initialization and Gradient Optimization for Sparse-view X-ray Novel View Synthesis and CT Reconstruction with 3D Gaussian Splatting, https://arxiv.org/abs/2609.22849, fetched 2026-10-27 (unofficial)
[S7] Optical-Mass Water Gaussians for Per-Scene Underwater Rendering, https://doi.org/10.21203/rs.3.rs-10938586/v1, fetched 2026-10-26 (unofficial)
[S8] Point Cloud Unsupervised Pre-training via 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2411.18667, fetched 2026-10-26 (unofficial)
