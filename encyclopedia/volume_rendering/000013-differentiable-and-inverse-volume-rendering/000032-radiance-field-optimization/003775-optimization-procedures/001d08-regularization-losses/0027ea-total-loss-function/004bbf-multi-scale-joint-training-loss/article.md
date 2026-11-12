# Multi-scale Joint Training Loss

Also called: Coarse-Fine Joint Loss, Joint Loss, Multi-scale Joint Loss

## Overview
A composite loss function designed to balance low-frequency structural agreement and high-frequency detail fidelity during radiance-field optimization [S2, S3]. It addresses the difficulty of simultaneously optimizing coarse geometric structures and fine-grained textures from scratch by weighting multiple training stages or resolution levels [S1, S3]. It differs from single-scale losses by explicitly aggregating errors across multiple network outputs or resolution stages [S1, S2].

## How it works
The method aggregates losses from multiple output stages (layers) or resolution levels. Two primary formulations exist in the literature: a multi-stage summation for dynamically growing networks [S1] and a coarse-fine weighted average for refinement [S2].

### Inputs and Outputs
- **Inputs**: Rendered images from $D$ stages/layers; ground truth images; network parameters $\theta$.
- **Outputs**: Scalar loss value $\mathcal{L}$ or $L(\theta)$.

### Data Structures
- **Stages**: A set of $D$ network outputs, where $D$ is the current number of stages [S1].
- **Loss Components**: Mean Squared Error ($\mathcal{L}_{MSE}$) and Uncertainty Loss ($\mathcal{L}_{unct}$) [S1]; Coarse Loss ($L_{coarse}$) and Fine Loss ($L_{fine}$) [S2].

### Formulas and Parameters

**Formulation 1: Multi-Stage Summation (Recursive-NeRF)**
Used in dynamically growing networks where all query coordinates are output through all outlets [S1].
$$ \mathcal{L} = \sum_{i=1}^D \beta_1 \mathcal{L}_{MSE}^i + \beta_2 \mathcal{L}_{unct}^i $$
- $D$: Current number of stages [S1].
- $\mathcal{L}_{MSE}^i$: MSE loss of the output image of layer $i$ [S1].
- $\mathcal{L}_{unct}^i$: Uncertainty loss of the output image of layer $i$ [S1].
- $\beta_1, \beta_2$: Weights. Typical values: $\beta_1 = 1.0, \beta_2 = 0.1$ [S1].

**Formulation 2: Coarse-Fine Weighted Average (Denoise-GS)**
Used in a second training round to refine denoised images, balancing low-frequency agreement and high-frequency fidelity [S2].
$$ L(\theta) = \alpha L_{coarse}(\theta) + (1-\alpha) L_{fine}(\theta) $$
- $\alpha \in [0, 1]$: Weight for coarse loss [S2].
- $L_{coarse}(\theta)$: Low-frequency consistency term comparing input and smoothed output, preserving tone and large-scale structure [S2].
- $L_{fine}(\theta)$: Detail consistency term comprising pixel-level robust distance, structural similarity, gradient consistency, and chroma consistency [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_multi_scale_joint_loss(rendered_images, ground_truth, uncertainty_maps, config):
    """
    Computes the multi-scale joint training loss.
    
    Args:
        rendered_images: List of rendered images for each stage D.
        ground_truth: Ground truth image.
        uncertainty_maps: List of uncertainty maps for each stage D.
        config: Dictionary containing 'method' ('multi_stage' or 'coarse_fine'),
                'beta1', 'beta2', 'alpha', and loss component functions.
                
    Returns:
        Scalar loss value.
    """
    method = config.get('method', 'multi_stage')
    
    if method == 'multi_stage':
        # Formulation 1: Summation over D stages
        total_loss = 0.0
        beta1 = config.get('beta1', 1.0)
        beta2 = config.get('beta2', 0.1)
        
        for i, img in enumerate(rendered_images):
            # Calculate MSE loss for stage i
            mse_loss = mean_squared_error(img, ground_truth)
            
            # Calculate uncertainty loss for stage i
            # Note: Specific definition of uncertainty loss not provided in sources [G]
            # Assuming standard uncertainty regularization based on uncertainty_maps[i]
            unct_loss = compute_uncertainty_loss(uncertainty_maps[i], ground_truth)
            
            total_loss += beta1 * mse_loss + beta2 * unct_loss
            
        return total_loss

    elif method == 'coarse_fine':
        # Formulation 2: Coarse-Fine weighted average
        alpha = config.get('alpha', 0.5) # Not given in sources, typical value placeholder [G]
        
        # Compute coarse loss (low-frequency consistency)
        # Smoothing is applied to input/output before comparison [S2]
        coarse_loss = compute_coarse_loss(rendered_images[0], ground_truth)
        
        # Compute fine loss (detail consistency)
        # Comprises robust distance, structural similarity, gradient, and chroma terms [S2]
        fine_loss = compute_fine_loss(rendered_images[-1], ground_truth)
        
        return alpha * coarse_loss + (1 - alpha) * fine_loss

    else:
        raise ValueError("Unknown method type")

def compute_coarse_loss(rendered, ground_truth):
    """
    Computes low-frequency consistency.
    Focuses on preserving tone and large scale structure [S2].
    Implementation detail: Apply Gaussian blur to both rendered and ground truth, then compute MSE [G].
    """
    blurred_rendered = gaussian_blur(rendered, sigma=2.0) # [G]
    blurred_gt = gaussian_blur(ground_truth, sigma=2.0)   # [G]
    return mean_squared_error(blurred_rendered, blurred_gt)

def compute_fine_loss(rendered, ground_truth):
    """
    Computes detail consistency.
    Consists of pixel level robust distance, structural similarity, gradient consistency, and chroma consistency [S2].
    """
    robust_dist = robust_pixel_distance(rendered, ground_truth) # [G]
    ssim_term = 1.0 - structural_similarity(rendered, ground_truth) # [G]
    gradient_term = gradient_consistency(rendered, ground_truth) # [G]
    chroma_term = chroma_consistency(rendered, ground_truth) # [G]
    
    # Weights for sub-components not given in sources [G]
    return robust_dist + ssim_term + gradient_term + chroma_term
```

### Edge Cases and Pitfalls
- **Precision**: Uncertainty loss definition is not explicitly detailed in the sources [S1], requiring implementation assumptions [G].
- **Boundaries**: The coarse-to-fine weight $\lambda$ in DP-NeRF decays exponentially, requiring careful initialization of $\lambda_s$ and $\lambda_e$ to avoid vanishing gradients [S3].
- **Degenerate Input**: Simultaneous optimization from scratch is difficult due to complex geometry; coarse-to-fine weighting is required to stabilize training [S3].

## Subtopics

### Coarse-to-Fine Optimization
A training strategy that uses an exponentially decaying weight $\lambda$ to balance coarse and fine RGB reconstruction losses, addressing the difficulty of simultaneous optimization from scratch [S3]. It introduces exponential weight decay between the reconstruction loss from the Rigid Body Kinematics (RBK) and the Adaptive Weight Prior (AWP) to stabilize training and gradually increase the effect of AWP [S3]. The weight $\lambda$ decays from $\lambda_s$ to $\lambda_e$ during training [S3]:
$$ \alpha = -\log(\lambda_s / \lambda_e) / (e^f - e^c) $$
$$ \lambda = \lambda_s (\exp(\alpha(e^c - e^i))) $$
where $e^c$ and $e^f$ are the start and end epochs, and $e^i$ is the current epoch [S3].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of stages $D$ in the multi-stage formulation [S1].
- **Memory Complexity**: Requires storing intermediate rendered images and uncertainty maps for all $D$ stages [S1].
- **Strengths**: Balances low-frequency structural consistency with high-frequency detail fidelity [S2]. Stabilizes training for complex 3D scenes by gradually increasing the influence of fine-grained components [S3].
- **Limitations**: Requires careful tuning of weights ($\beta_1, \beta_2, \alpha$) and decay parameters ($\lambda_s, \lambda_e$) [S1, S3]. The definition of uncertainty loss is not fully specified in the primary source [S1].

## Background
Originated in the context of NeRF optimization to handle multi-resolution or multi-stage training. The multi-stage summation approach was introduced in Recursive-NeRF [S1]. The coarse-fine joint loss was proposed in Denoise-GS for sparse-view 3D Gaussian Splatting denoising [S2]. The exponential decay variant was introduced in DP-NeRF for deblurred neural radiance fields [S3].

## Sources
[S1] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
[S2] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
[S3] DP-NeRF: Deblurred Neural Radiance Field with Physical Scene Priors, https://arxiv.org/abs/2211.12046v4, fetched 2026-10-26
