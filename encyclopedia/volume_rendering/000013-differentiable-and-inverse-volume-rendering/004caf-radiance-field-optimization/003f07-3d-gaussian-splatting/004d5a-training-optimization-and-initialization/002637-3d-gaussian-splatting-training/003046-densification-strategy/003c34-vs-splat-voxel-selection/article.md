# VS-Splat Voxel Selection
Also called: Ours, VS-Splat selection

## Overview
VS-Splat Voxel Selection is a learnable, differentiable mechanism for selecting relevant 3D Gaussian primitives from a dense voxel grid during the training of feed-forward Gaussian Splatting models [S1, unofficial]. It solves the problem of inefficient primitive distribution by replacing static opacity thresholding with a neural network that predicts confidence scores for each voxel, allowing the model to focus computational resources on object-centric regions [S1, unofficial]. It sets itself apart from its siblings by using a Gumbel-Sigmoid relaxation to enable gradient flow through the discrete selection process, unlike [Score-based Densification](4705-score-based-densification.md) which relies on 2D positional gradients, or [Straight-Through Estimator (STE) for Confidence Scores](9636-straight-through-estimator-ste-for-confidence-scores.md) which uses hard thresholding [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid where each voxel contains a set of fine-grained 3D Gaussians (typically $M=16$ per voxel) and a learned confidence score $c_v \in [0, 1]$ for each voxel $v$ [S1, unofficial].
*   **Output:** A subset of voxels selected for rendering, effectively pruning background noise and reducing the number of active primitives [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D array with resolution $H \times W \times D$ (e.g., $48 \times 48 \times 48$) [S1, unofficial].
*   **Confidence Score:** A scalar value per voxel, predicted by a Selective Feature Refinement (SFR) module [S1, unofficial].
*   **Gaussian Primitives:** Each selected voxel spawns $M$ Gaussians. The selection determines which voxels contribute to the final image [S1, unofficial].

**Method Steps:**
1.  **Feature Extraction:** Extract 2D feature maps from input images using a pre-trained encoder (e.g., DINOv1) [S1, unofficial].
2.  **Compression:** Compress features using a 3D convolutional compressor $C$ to reduce channel dimensions (e.g., from 1536 to 128) [S1, unofficial].
3.  **Confidence Prediction:** Pass compressed features through the SFR module (containing $R=2$ residual convolutional blocks) to predict a confidence score $c_v$ for each voxel $v$ [S1, unofficial].
4.  **Gumbel-Sigmoid Sampling:** To make the selection differentiable, sample a binary mask $m_v$ for each voxel using the Gumbel-Sigmoid function:
    $$ m_v = \text{GumbelSigmoid}(c_v, \tau) $$
    where $\tau$ is the temperature parameter [S1, unofficial].
5.  **Opacity Refinement:** Use the sampled mask $m_v$ to refine the opacity $\alpha_v$ of the Gaussians in voxel $v$. Voxels with $m_v \approx 0$ are effectively pruned [S1, unofficial].
6.  **Rendering:** Render the scene using only the Gaussians from selected voxels ($m_v \approx 1$) [S1, unofficial].

**Parameters:**
*   **Gumbel-Sigmoid Temperature ($\tau$):** Linearly annealed from 1.0 to 0.1 over the first 50,000 training iterations, then fixed at 0.1 for the remainder of training and inference [S1, unofficial].
*   **Voxel Resolution:** $H = W = D = 48$ [S1, unofficial].
*   **Primitives per Voxel:** $M = 16$ [S1, unofficial].
*   **SFR Blocks:** $R = 2$ residual convolutional blocks [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vsplat_voxel_selection(confidence_scores, temperature):
    """
    Selects voxels using Gumbel-Sigmoid relaxation.
    
    Args:
        confidence_scores: Tensor of shape (H, W, D) with values in [0, 1].
        temperature: Float, the Gumbel-Sigmoid temperature tau.
        
    Returns:
        selection_mask: Binary tensor of shape (H, W, D) indicating selected voxels.
    """
    # Sample Gumbel noise
    # Note: In practice, this is done via the Gumbel-Sigmoid trick
    # m = sigmoid((log(u) - log(1-u) + log(c) - log(1-c)) / tau)
    # where u is uniform noise [0,1] and c is confidence_score
    
    # Simplified representation of the Gumbel-Sigmoid operation
    # This allows gradients to flow through the selection decision
    selection_mask = gumbel_sigmoid(confidence_scores, temperature)
    
    return selection_mask

def gumbel_sigmoid(logits, temperature):
    """
    Approximates binary sampling using Gumbel-Sigmoid.
    
    Args:
        logits: Tensor of confidence scores.
        temperature: Float, temperature parameter tau.
        
    Returns:
        Tensor of sampled values in [0, 1].
    """
    # Generate uniform noise
    u = torch.rand_like(logits)
    
    # Gumbel noise
    g = -torch.log(-torch.log(u + 1e-10) + 1e-10)
    
    # Gumbel-Sigmoid
    y = torch.sigmoid((g + torch.log(logits + 1e-10) - torch.log(1 - logits + 1e-10)) / temperature)
    
    return y
```

**Edge Cases and Pitfalls:**
*   **Temperature Scheduling:** Failing to anneal $\tau$ can lead to poor convergence. A high temperature early on encourages exploration, while a low temperature later sharpens the selection [S1, unofficial].
*   **Over-pruning:** If the confidence scores are not well-calibrated, too many voxels may be pruned, leading to loss of detail [S1, unofficial].
*   **Under-pruning:** If the selection is too loose, the model may retain background noise, degrading rendering quality [S1, unofficial].

## Subtopics

### Deterministic Sigmoid selection
A variant that replaces the Gumbel-Sigmoid with the standard deterministic Sigmoid function for voxel selection [S1, unofficial]. It fails to properly select object-centric voxels, leading to significant rendering quality drops (PSNR 18.01, SSIM 0.835, LPIPS 0.288 on CO3D) and uses only 2,223 voxels, which is significantly fewer than VS-Splat [S1, unofficial]. The stochastic nature of Gumbel-Sigmoid is crucial for encouraging the network to select appropriate object-voxels [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Uses approximately 15% fewer voxels than Opacity-Based Thresholding (OBT) [S1, unofficial]. On the CO3D dataset, VS-Splat uses 19,587 voxels compared to OBT's 22,867 [S1, unofficial].
*   **Time:** Inference speed is comparable to LaRa, with the added benefit of better rendering quality [S1, unofficial].

**Measured Performance:**
*   **CO3D Dataset:** PSNR 21.31, SSIM 0.863, LPIPS 0.213 [S1, unofficial].
*   **Comparison:** Outperforms OBT (PSNR 21.21, SSIM 0.859, LPIPS 0.223) and STE (PSNR 14.71, SSIM 0.814, LPIPS 0.317) across all metrics [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Learnable selection adapts to the scene, better identifies object-centric voxels, and reduces the number of primitives without sacrificing quality [S1, unofficial].
*   **Limitations:** Requires careful tuning of the temperature schedule and SFR module architecture [S1, unofficial].

**When to Use:**
*   Use when you need efficient, end-to-end 3D reconstruction from sparse views and want to automatically prune background noise [S1, unofficial].
*   Avoid when computational resources for training the SFR module are limited or when a simpler, non-learnable thresholding method suffices [S1, unofficial].

## Background
VS-Splat was introduced in the paper "VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views" [S1, unofficial]. It builds on the 3D Gaussian Splatting framework [10] and aims to improve efficiency and quality in sparse-view reconstruction by selectively activating voxels based on learned confidence scores [S1, unofficial].

## Sources
[S1] VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views, https://doi.org/10.48550/arxiv.2609.12343, fetched 2026-10-26 (unofficial)
