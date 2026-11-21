# VS-Splat Voxel Selection
Also called: Ours, VS-Splat selection

## Overview
VS-Splat Voxel Selection is a learnable, differentiable mechanism for selecting relevant 3D voxels in feed-forward Gaussian Splatting, replacing static opacity thresholding with a confidence-based selection process [S1, unofficial]. It sets itself apart from siblings like [Opacity-Based Thresholding](6234-opacity-based-thresholding.md) by using a Gumbel-Sigmoid distribution to predict confidence scores, allowing the network to learn which voxels contain object-centric geometry rather than relying on manual hyperparameter tuning [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid (resolution $H \times W \times D$) and a feature extractor (e.g., DINOv1) producing 2D feature maps from input images [S1, unofficial].
*   **Output:** A binary mask of selected voxels and refined opacity values for rendering [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D tensor of resolution $H=W=D=48$ [S1, unofficial].
*   **Confidence Scores:** A scalar value per voxel, predicted by a Selective Feature Refinement (SFR) module [S1, unofficial].
*   **Primitives:** $M=16$ fine-grained 3D Gaussians are generated for each *selected* voxel [S1, unofficial].

**Parameters:**
*   **Gumbel-Sigmoid Temperature ($\tau$):** Linearly annealed from $1.0$ to $0.1$ over the first $50,000$ training iterations, then fixed at $0.1$ for the remainder of training and inference [S1, unofficial].
*   **SFR Module Depth ($R$):** Contains $R=2$ residual convolutional blocks [S1, unofficial].
*   **Channel Compression:** The 3D convolutional compressor reduces channel dimension from $2C=1,536$ to $\tilde{C}=128$ [S1, unofficial].

**Steps:**
1.  **Feature Extraction:** Extract 2D feature maps from input images using a pre-trained backbone (e.g., DINOv1) [S1, unofficial].
2.  **Feature Compression:** Compress features using a 3D convolutional compressor to reduce channel dimensionality [S1, unofficial].
3.  **Confidence Prediction:** Pass compressed features through the SFR module to predict confidence scores for each voxel [S1, unofficial].
4.  **Voxel Selection:** Apply the Gumbel-Sigmoid function to the confidence scores to generate a differentiable binary selection mask [S1, unofficial].
5.  **Opacity Refinement:** Use the selection mask to refine the opacity values of the Gaussians associated with the selected voxels [S1, unofficial].
6.  **Primitive Generation:** For each selected voxel, generate $M=16$ fine-grained 3D Gaussian primitives [S1, unofficial].
7.  **Rendering:** Render the scene using the selected voxels and their associated primitives [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vs_splat_voxel_selection(voxel_grid, input_images, temperature_tau):
    """
    Performs voxel selection using VS-Splat method.
    
    Args:
        voxel_grid: 3D tensor of shape (H, W, D) representing the voxel space.
        input_images: List of 2D input images.
        temperature_tau: Float, current Gumbel-Sigmoid temperature.
        
    Returns:
        selected_mask: Binary mask of selected voxels.
        refined_opacities: Refined opacity values for selected voxels.
    """
    # Step 1: Extract 2D features from input images
    # Using DINOv1 pre-trained on ImageNet-1K
    features_2d = extract_features_dinov1(input_images)
    
    # Step 2: Compress features using 3D convolutional compressor
    # Reduces channel dimension from 1536 to 128
    features_compressed = compress_features_3d(features_2d, in_channels=1536, out_channels=128)
    
    # Step 3: Predict confidence scores using SFR module
    # SFR module contains R=2 residual convolutional blocks
    confidence_scores = predict_confidence_sfr(features_compressed, num_residual_blocks=2)
    
    # Step 4: Apply Gumbel-Sigmoid for differentiable selection
    # temperature_tau is annealed from 1.0 to 0.1 over first 50k iterations
    selected_mask = gumbel_sigmoid(confidence_scores, temperature=temperature_tau)
    
    # Step 5: Refine opacities based on selection
    # This step integrates predicted voxel-selection scores into the rendering process
    refined_opacities = refine_opacities(confidence_scores, selected_mask)
    
    return selected_mask, refined_opacities

def gumbel_sigmoid(scores, temperature):
    """
    Applies Gumbel-Sigmoid function for differentiable binary selection.
    
    Args:
        scores: Tensor of confidence scores.
        temperature: Float, temperature parameter for Gumbel-Sigmoid.
        
    Returns:
        Binary mask (soft binary) of selected voxels.
    """
    # Gumbel noise for stochasticity
    gumbel_noise = sample_gumbel_noise(scores.shape)
    # Gumbel-Sigmoid formula
    sigmoid_input = (scores + gumbel_noise) / temperature
    return torch.sigmoid(sigmoid_input)

def sample_gumbel_noise(shape):
    """
    Samples Gumbel(0, 1) noise.
    
    Args:
        shape: Shape of the output tensor.
        
    Returns:
        Tensor of Gumbel noise.
    """
    # Sample from uniform distribution
    u = torch.rand(shape)
    # Gumbel distribution transformation
    return -torch.log(-torch.log(u + 1e-10) + 1e-10)
```

**Edge Cases and Pitfalls:**
*   **Temperature Scheduling:** Using a fixed high temperature may lead to excessive selection of background voxels, while a fixed low temperature may result in insufficient selection [S1, unofficial].
*   **Deterministic Sigmoid:** Replacing Gumbel-Sigmoid with a deterministic Sigmoid function fails to properly select object-centric voxels, leading to significant rendering quality drops [S1, unofficial].
*   **Opacity Thresholding:** Simple opacity-based thresholding (OBT) is unreliable for selecting object-centric voxels and requires manual tuning of the threshold value [S1, unofficial].

## Subtopics

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
VS-Splat was proposed in the paper "VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views" [S1, unofficial]. It addresses the limitations of static opacity thresholding in 3D Gaussian Splatting by introducing a learnable voxel selection mechanism. The method is particularly effective for end-to-end 3D object reconstruction from sparse views [S1, unofficial].

## Sources
[S1] VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views, https://doi.org/10.48550/arxiv.2609.12343, fetched 2026-10-26 (unofficial)
