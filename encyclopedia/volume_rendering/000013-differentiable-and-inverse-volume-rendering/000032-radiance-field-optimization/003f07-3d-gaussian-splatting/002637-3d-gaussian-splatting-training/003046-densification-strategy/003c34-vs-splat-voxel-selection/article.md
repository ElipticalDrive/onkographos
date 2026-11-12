# VS-Splat Voxel Selection
Also called: Ours, VS-Splat selection

## Overview
VS-Splat Voxel Selection is a learnable mechanism for selecting relevant 3D Gaussian primitives from a voxel grid by predicting confidence scores via Gumbel-Sigmoid and using them to refine opacity values [S1, unofficial]. It solves the problem of redundant background primitives and unreliable thresholding in sparse-view reconstruction [S1, unofficial]. It sets itself apart from its sibling [Adaptive Density Control](9188-adaptive-density-control.md) by replacing static opacity thresholding with a differentiable, learned selection process that significantly reduces voxel count while improving rendering quality [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid containing $M$ fine-grained 3D Gaussian primitives per voxel, along with their initial opacity values $\alpha$.
*   **Output:** A refined set of selected voxels/primitives with updated opacity values, effectively masking out non-object regions.

**Data Structures:**
*   **Confidence Scores:** A scalar value per voxel/primitive predicted by the Selective Feature Refinement (SFR) module.
*   **Gumbel-Sigmoid Distribution:** Used to sample stochastic binary masks from the confidence scores.

**Parameters:**
*   **Gumbel-Sigmoid Temperature ($\tau$):** Controls the sharpness of the sigmoid approximation. Linearly annealed from 1.0 to 0.1 over the first 50,000 iterations, then fixed at 0.1 for the remaining training and inference [S1, unofficial].
*   **Voxel Grid Resolution:** $H = W = D = 48$ (used in reported experiments) [S1, unofficial].
*   **Primitives per Voxel:** $M = 16$ fine-grained primitives generated for each selected voxel [S1, unofficial].

**Steps:**
1.  **Feature Extraction:** Extract 2D feature maps from input images using a pre-trained encoder (e.g., DINOv1) [S1, unofficial].
2.  **Compression:** Compress features using a 3D convolutional compressor $C$ reducing channels from $2C = 1,536$ to $\tilde{C} = 128$ [S1, unofficial].
3.  **Confidence Prediction:** Pass compressed features through the Selective Feature Refinement (SFR) module, which contains $R=2$ residual convolutional blocks, to predict confidence scores for each voxel [S1, unofficial].
4.  **Stochastic Selection:** Apply the Gumbel-Sigmoid function to the predicted confidence scores with temperature $\tau$ to generate a differentiable binary mask [S1, unofficial].
    *   Formula: $s = \text{GumbelSigmoid}(\text{score}, \tau)$
    *   This allows gradients to flow through the selection decision during backpropagation [S1, unofficial].
5.  **Opacity Refinement:** Multiply the initial opacity $\alpha$ of each primitive by the selection mask $s$ to refine the final opacity values [S1, unofficial].
6.  **Rendering:** Render the scene using the refined opacities.

**Edge Cases and Pitfalls:**
*   **Excessive Background Primitives:** Using a static opacity threshold (e.g., 0.0) generates too many fine-grained primitives in non-object regions, degrading quality [S1, unofficial].
*   **Deterministic Sigmoid Failure:** Replacing Gumbel-Sigmoid with a standard deterministic Sigmoid fails to properly select object-centric voxels, leading to significant quality drops [S1, unofficial].
*   **STE Limitations:** Using a Straight-Through Estimator (STE) for selection results in an extremely limited number of selected voxels (e.g., 121 on CO3D), severely under-reconstructing the scene [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vs_splat_voxel_selection(confidence_scores, initial_opacities, tau):
    """
    Selects voxels and refines opacities using Gumbel-Sigmoid.
    
    Args:
        confidence_scores: Tensor of shape (N,) containing predicted confidence scores.
        initial_opacities: Tensor of shape (N,) containing initial opacity values.
        tau: Float, Gumbel-Sigmoid temperature.
        
    Returns:
        refined_opacities: Tensor of shape (N,) with refined opacity values.
    """
    # Sample from Gumbel-Sigmoid distribution to get stochastic mask
    # This operation is differentiable w.r.t confidence_scores
    selection_mask = gumbel_sigmoid_sample(confidence_scores, tau)
    
    # Refine opacities by multiplying with the selection mask
    refined_opacities = initial_opacities * selection_mask
    
    return refined_opacities

def gumbel_sigmoid_sample(logits, tau):
    """
    Samples from the Gumbel-Sigmoid distribution.
    
    Args:
        logits: Tensor of shape (N,) representing the confidence scores.
        tau: Float, temperature parameter.
        
    Returns:
        Tensor of shape (N,) with values in [0, 1].
    """
    # Generate Gumbel noise
    gumbel_noise = -torch.log(-torch.log(torch.rand_like(logits) + 1e-9) + 1e-9)
    
    # Compute Gumbel-Sigmoid
    # Note: In practice, this might involve a softmax-like operation over binary choices
    # or a direct sigmoid with noise. The exact implementation details of the 
    # Gumbel-Sigmoid function are not fully specified in the source, 
    # but it serves as a differentiable approximation of a binary selection.
    # A common formulation is: sigmoid((logits + gumbel_noise) / tau)
    # However, for binary selection, it often involves a straight-through 
    # or specific stochastic rounding. Based on the source, it enables 
    # gradient flow to the scores.
    
    # Assuming a standard Gumbel-Sigmoid formulation for binary selection:
    # s = sigmoid((logits + gumbel_noise) / tau)
    # For binary output, one might use a straight-through estimator or 
    # round the result, but the source emphasizes the differentiable nature.
    # We will use the continuous relaxation as the primary mechanism.
    
    s = torch.sigmoid((logits + gumbel_noise) / tau)
    return s
```

## Subtopics

### Deterministic Sigmoid selection
A variant that replaces the Gumbel-Sigmoid with the deterministic Sigmoid function [S1, unofficial]. It fails to properly select object-centric voxels, leading to significant rendering quality drops (PSNR 18.01, SSIM 0.835, LPIPS 0.288 on CO3D) and uses only 2,223 voxels [S1, unofficial].

## Operating characteristics
*   **Voxel Count:** VS-Splat uses approximately 15% fewer voxels than the Opacity-Based Thresholding (OBT) baseline [S1, unofficial].
*   **Performance (CO3D Dataset):**
    *   PSNR: 21.31
    *   SSIM: 0.863
    *   LPIPS: 0.213
    *   Voxels: 19,587 [S1, unofficial]
*   **Comparison with OBT:** Outperforms OBT (PSNR 21.21, SSIM 0.859, LPIPS 0.223, 22,867 voxels) across all metrics while using fewer voxels [S1, unofficial].
*   **Comparison with STE:** Significantly outperforms STE (PSNR 14.71, SSIM 0.814, LPIPS 0.317, 121 voxels) which selects too few voxels [S1, unofficial].
*   **Comparison with Deterministic Sigmoid:** Significantly outperforms Deterministic Sigmoid (PSNR 18.01, SSIM 0.835, LPIPS 0.288, 2,223 voxels) [S1, unofficial].
*   **Strengths:** Better identification of object-centric voxels; reduces redundant background primitives; differentiable selection allows end-to-end training.
*   **Limitations:** Requires tuning of the temperature parameter $\tau$; relies on a pre-trained feature extractor (DINOv1) which adds complexity.

## Background
VS-Splat was proposed in the paper "VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views" [S1, unofficial]. It addresses limitations in existing voxel selection methods like Opacity-Based Thresholding (OBT) and Straight-Through Estimator (STE) approaches, which either select too many background voxels or too few object voxels, respectively. The method integrates a Selective Feature Refinement (SFR) module to predict confidence scores and uses Gumbel-Sigmoid for differentiable selection.

## Sources
[S1] VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views, https://doi.org/10.48550/arxiv.2609.12343, fetched 2026-10-26 (unofficial)
