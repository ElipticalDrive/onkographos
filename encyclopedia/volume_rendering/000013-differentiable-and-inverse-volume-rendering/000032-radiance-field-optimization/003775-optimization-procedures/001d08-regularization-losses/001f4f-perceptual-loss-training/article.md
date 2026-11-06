# Perceptual Loss Training

Also called: PIPS Loss, Perceptual Loss for 3DGS

## Overview
Perceptual Loss Training is an optimization strategy that incorporates a perceptual loss term into the training objective to improve the perceptual quality of rendered images, addressing the inherent smoothness and blurriness issues associated with standard pixel-wise reconstruction losses [S1]. It sets itself apart from sibling methods like [Total Variation Regularization](860-total-variation-regularization.md) and [L1+SSIM Loss](2890-l1-ssim-loss.md) by leveraging high-level feature representations from Convolutional Neural Networks (CNNs) rather than relying solely on low-level pixel statistics or structural similarity metrics [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered image $I_{render}$, ground-truth image $I_{gt}$, and a pre-trained feature extractor network (typically VGG or similar) [S1].
*   **Outputs:** A scalar perceptual loss value $\mathcal{L}_{\text{PIPS}}$ used for gradient backpropagation [S1].

**Data Structures:**
*   **Feature Maps:** The CNN extracts feature maps at multiple intermediate layers. Let $\Phi_l(I)$ denote the feature map at layer $l$ for image $I$. The layout is a tensor of shape $(C_l, H_l, W_l)$, where $C_l$ is the number of channels, and $H_l, W_l$ are the spatial dimensions specific to layer $l$ [S1].

**Formulas:**
The total training loss $\mathcal{L}$ is a weighted combination of L1 loss, Structural Similarity (SSIM) loss, and Perceptual Loss [S1]:

$$
\mathcal{L} = (1 - \lambda_1 - \lambda_2)\mathcal{L}_1 + \lambda_1\mathcal{L}_{\text{SSIM}} + \lambda_2\mathcal{L}_{\text{PIPS}}
$$

Where:
*   $\mathcal{L}_1$ is the L1 reconstruction loss: $\mathcal{L}_1 = \| I_{render} - I_{gt} \|_1$ [S1].
*   $\mathcal{L}_{\text{SSIM}}$ is the structural similarity loss [S1].
*   $\mathcal{L}_{\text{PIPS}}$ is the perceptual loss, computed as the Euclidean distance between the feature representations of the rendered and ground-truth images [S1].
*   $\lambda_1$ is the weight for the SSIM loss term. Value not explicitly given in sources [S1].
*   $\lambda_2$ is the weight for the perceptual loss term. Value not explicitly given in sources [S1].

**Numbered Steps:**
1.  Render the image $I_{render}$ from the current model state (e.g., Gaussian Splatting parameters) [S1].
2.  Pass both $I_{render}$ and $I_{gt}$ through the pre-trained CNN feature extractor to obtain feature maps $\Phi(I_{render})$ and $\Phi(I_{gt})$ [S1].
3.  Compute the perceptual loss $\mathcal{L}_{\text{PIPS}}$ as the sum of squared differences between feature maps across selected layers [S1].
4.  Compute $\mathcal{L}_1$ and $\mathcal{L}_{\text{SSIM}}$ [S1].
5.  Combine losses using the formula above to get $\mathcal{L}$ [S1].
6.  Backpropagate gradients through $\mathcal{L}$ to update model parameters [S1].

**Computational Optimization:**
Because CNN-based computation increases training time, the perceptual loss $\mathcal{L}_{\text{PIPS}}$ is computed only every 20 training iterations [S1]. In iterations where it is not computed, its gradient contribution is effectively zero or carried over from the last computation, substantially alleviating computational overhead without noticeably increasing overall training time [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_perceptual_loss_training(render_fn, gt_image, cnn_extractor, lambda1, lambda2, iteration):
    """
    Computes the total loss including perceptual loss, updating every 20 iterations.
    """
    # 1. Render the current view
    rendered_image = render_fn()
    
    # 2. Compute standard losses
    l1_loss = compute_l1_loss(rendered_image, gt_image)
    ssim_loss = compute_ssim_loss(rendered_image, gt_image)
    
    # 3. Compute perceptual loss only every 20 iterations
    if iteration % 20 == 0:
        # Extract features from CNN
        rendered_features = cnn_extractor(rendered_image)
        gt_features = cnn_extractor(gt_image)
        
        # Compute perceptual loss (e.g., L2 distance between feature maps)
        # Note: Specific layer selection and weighting within PIPS is not detailed in [S1]
        pips_loss = compute_pips_loss(rendered_features, gt_features)
    else:
        # Use cached value or zero gradient contribution
        # [G] Implementation detail: typically one would cache the last computed pips_loss
        # and use its value for the loss sum, but gradients would be stale.
        # The source states it alleviates overhead, implying the term is skipped in backprop
        # or the gradient is not computed for this term.
        pips_loss = 0.0 
        
    # 4. Combine losses
    # Ensure weights sum to <= 1 to keep L1 weight positive
    l1_weight = 1.0 - lambda1 - lambda2
    total_loss = (l1_weight * l1_loss) + (lambda1 * ssim_loss) + (lambda2 * pips_loss)
    
    return total_loss
```

**Edge Cases and Pitfalls:**
*   **Computational Overhead:** The primary pitfall is the significant increase in training time due to CNN inference [S1]. Mitigated by computing it every 20 iterations [S1].
*   **Metric Reliability:** Perceptual metrics like LPIPS (closely related to PIPS) may be unreliable signals in certain settings, such as scenes with blurry backgrounds where lower LPIPS scores might correspond to perceptually inferior outputs [S2].
*   **Proxy Geometry Dependency (Variant):** In variants relying on proxy geometry (e.g., Stable View Synthesis), incorrect proxy geometry (especially for distant/unobserved content like trees) can cause the perceptual loss to fail or produce poor results [S2].

## Subtopics

### Stable View Synthesis (SVS) Baseline
A view synthesis method that blends training images together, relying on proxy geometry from an external system [S2]. It is trained with a perceptual loss [S2]. It requires multiple large CNNs and access to all training images to render novel views [S2]. It may fail when the external proxy geometry is incorrect, particularly struggling with distant/unobserved content like trees [S2]. It is better suited to large piecewise planar objects (vehicles) due to reliance on piecewise planar mesh proxy geometry, whereas the proposed model handles natural content better [S2].

## Operating characteristics
*   **Time Complexity:** Increases training time significantly due to CNN-based computation [S1]. Mitigated by computing the perceptual loss every 20 iterations [S1].
*   **Memory Complexity:** Requires memory for the pre-trained CNN and intermediate feature maps [S1].
*   **Strengths:** Improves perceptual quality of rendered images, addressing the smoothness issues of pixel-wise losses [S1].
*   **Limitations:** Computationally expensive; relies on external CNNs; may be less effective or fail if dependent on incorrect proxy geometry (in variant contexts) [S1, S2].
*   **When to use:** When perceptual quality is prioritized over raw PSNR and computational resources allow for periodic CNN inference [S1].
*   **When not to use:** When real-time training speed is critical and the overhead of CNNs is prohibitive, or when the scene contains natural content that does not align well with piecewise planar proxy geometry (in SVS context) [S2].

## Background
Perceptual loss training in the context of Gaussian Splatting and radiance fields was introduced to address the limitations of pixel-wise reconstruction errors, which often result in solutions with high signal-to-noise ratios that are perceptually smooth and less convincing [S1]. The approach builds on GAN-based models where perceptual losses have been proven effective [S1]. The specific formulation using $\mathcal{L}_{\text{PIPS}}$ in the context of Duplex-GS was presented in [S1]. The comparison with Stable View Synthesis (SVS), which also uses perceptual loss, highlights the trade-offs between proxy-geometry-dependent methods and direct optimization methods [S2].

## Sources
[S1] Duplex-GS: Proxy-Guided Weighted Blending for Real-Time Order-Independent Gaussian Splatting, https://doi.org/10.1109/tcsvt.2026.3666188, fetched 2026-10-30
[S2] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
