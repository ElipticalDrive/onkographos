# VS-Splat Voxel Selection
Also called: Ours, VS-Splat selection

## Overview
VS-Splat Voxel Selection is a learnable, differentiable mechanism for selecting relevant 3D voxels in feed-forward Gaussian Splatting, replacing static opacity thresholding with a confidence-based selection process [S1, unofficial]. It sets itself apart from siblings like [Opacity-Based Thresholding](6234-opacity-based-thresholding.md) by using a Gumbel-Sigmoid distribution to predict confidence scores, allowing the network to learn which voxels contain object-centric geometry rather than relying on manual hyperparameter tuning [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid and a set of input 2D images [S1, unofficial].
*   **Output:** A binary selection mask indicating active voxels and refined opacity values for rendering [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D tensor with resolution $H=W=D=48$ [S1, unofficial].
*   **Confidence Scores:** A scalar value per voxel, predicted by the Selective Feature Refinement (SFR) module [S1, unofficial].
*   **Primitives:** $M=16$ fine-grained 3D Gaussian primitives are generated for each selected voxel [S1, unofficial].

**Numbered Steps:**
1.  Extract 2D feature maps from input images using a pre-trained backbone (DINOv1) [S1, unofficial].
2.  Compress features using a 3D convolutional compressor to reduce channel dimensionality [S1, unofficial].
3.  Pass compressed features through the SFR module to predict confidence scores for each voxel [S1, unofficial].
4.  Apply the Gumbel-Sigmoid function to the confidence scores to generate a differentiable binary selection mask [S1, unofficial].
5.  Refine opacity values using the selection mask and confidence scores [S1, unofficial].
6.  Generate $M=16$ fine-grained 3D Gaussian primitives for each selected voxel [S1, unofficial].
7.  Render the scene using the selected voxels and their associated primitives [S1, unofficial].

**Formulas:**
The Gumbel-Sigmoid function generates the differentiable selection mask $y$ from a confidence score $s$ and temperature $\tau$:
$$y = \frac{1}{1 + \exp\left(-\frac{s + g}{\tau}\right)}$$
where $g = -\ln(-\ln(u))$ is Gumbel noise sampled from $u \sim \text{Uniform}(0,1)$ [G]. The refined opacity is computed as the product of the confidence score and the selection mask [S1, unofficial].

**Parameters:**
*   **Gumbel-Sigmoid Temperature ($\tau$):** Linearly annealed from $1.0$ to $0.1$ over the first $50,000$ training iterations, then fixed at $0.1$ for the remainder of training and inference [S1, unofficial].
*   **SFR Module Depth ($R$):** Contains $R=2$ residual convolutional blocks [S1, unofficial].
*   **Channel Compression:** The 3D convolutional compressor reduces the channel dimension from $2C=1,536$ to $\tilde{C}=128$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Temperature Scheduling:** Using a fixed high temperature may lead to excessive selection of background voxels, while a fixed low temperature may result in insufficient selection [S1, unofficial].
*   **Deterministic Sigmoid:** Replacing Gumbel-Sigmoid with a deterministic Sigmoid function fails to properly select object-centric voxels, leading to significant rendering quality drops [S1, unofficial].
*   **Opacity Thresholding:** Simple opacity-based thresholding (OBT) is unreliable for selecting object-centric voxels and requires manual tuning of the threshold value [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vs_splat_voxel_selection(voxel_grid, input_images, temperature_tau):
    # Step 1: Extract 2D features using DINOv1 backbone
    features_2d = extract_2d_features(input_images)
    # Step 2: Compress features via 3D convolution (1536 -> 128 channels)
    features_compressed = compress_3d(features_2d, in_channels=1536, out_channels=128)
    # Step 3: Predict confidence scores via SFR module (R=2 residual blocks)
    confidence_scores = sfr_forward(features_compressed, num_residual_blocks=2)
    # Step 4: Differentiable selection via Gumbel-Sigmoid
    selected_mask = gumbel_sigmoid(confidence_scores, temperature=temperature_tau)
    # Step 5: Refine opacities
    refined_opacities = refine_opacities(confidence_scores, selected_mask)
    return selected_mask, refined_opacities

def extract_2d_features(images):
    # Standard ViT forward pass for DINOv1 pre-trained on ImageNet-1K
    return [backbone_forward(img) for img in images]

def compress_3d(features, in_channels, out_channels):
    # 3D convolutional compressor reducing channel dimensionality
    return conv3d(features, in_channels, out_channels, kernel_size=1)

def sfr_forward(features, num_residual_blocks):
    # SFR module with residual convolutional blocks
    out = features
    for _ in range(num_residual_blocks):
        out = out + conv_block(out)
    return out

def gumbel_sigmoid(scores, temperature):
    # Sample uniform noise
    u = uniform_like(scores)
    # Gumbel noise
    g = -log(-log(u + 1e-10) + 1e-10)
    # Gumbel-Sigmoid formula
    return sigmoid((scores + g) / temperature)

def refine_opacities(scores, mask):
    # Integrate predicted voxel-selection scores into the rendering process
    return scores * mask
```

## Subtopics

### [Straight-Through Estimator (STE) for Confidence Scores](9636-straight-through-estimator-ste-for-confidence-scores.md)
A variant that applies the Straight-Through Estimator to confidence scores to enable gradient flow. It extends gradients beyond the hard selection boundary to prevent optimization stalling. On the CO3D dataset, it selects an extremely limited number of voxels (121), yielding PSNR 14.71, SSIM 0.814, and LPIPS 0.317.

### Deterministic Sigmoid selection
A variant that replaces the Gumbel-Sigmoid with the deterministic Sigmoid function for voxel selection [S1, unofficial]. It fails to properly select object-centric voxels, leading to significant rendering quality drops [S1, unofficial]. On the CO3D dataset, it yields PSNR 18.01, SSIM 0.835, LPIPS 0.288, and uses 2,223 voxels [S1, unofficial].

## Operating characteristics
*   **Performance:** On the CO3D dataset, VS-Splat yields PSNR 21.31, SSIM 0.863, LPIPS 0.213, and uses 19,587 voxels [S1, unofficial].
*   **Comparison:** VS-Splat uses approximately 15% fewer voxels than Opacity-Based Thresholding (OBT) while outperforming it across all metrics [S1, unofficial]. OBT yields PSNR 21.21, SSIM 0.859, LPIPS 0.223, and uses 22,867 voxels [S1, unofficial].
*   **Comparison:** Simply applying Straight-Through Estimator (STE) to confidence scores results in selecting an extremely limited number of voxels (121 voxels on CO3D), yielding PSNR 14.71, SSIM 0.814, LPIPS 0.317 [S1, unofficial].
*   **Scalability:** The method uses a voxel grid resolution of $H=W=D=48$ and generates $M=16$ primitives per selected voxel [S1, unofficial].
*   **Strengths:** Learnable selection allows for better identification of object-centric voxels compared to static thresholds [S1, unofficial].
*   **Limitations:** Requires careful tuning of the Gumbel-Sigmoid temperature schedule [S1, unofficial].

## Background
VS-Splat was proposed in the paper "VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views" [S1, unofficial]. It addresses the limitations of static opacity thresholding in 3D Gaussian Splatting by introducing a learnable voxel selection mechanism [S1, unofficial]. The method is particularly effective for end-to-end 3D object reconstruction from sparse views [S1, unofficial].

## Sources
[S1] VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views, https://doi.org/10.48550/arxiv.2609.12343, fetched 2026-10-26 (unofficial)
