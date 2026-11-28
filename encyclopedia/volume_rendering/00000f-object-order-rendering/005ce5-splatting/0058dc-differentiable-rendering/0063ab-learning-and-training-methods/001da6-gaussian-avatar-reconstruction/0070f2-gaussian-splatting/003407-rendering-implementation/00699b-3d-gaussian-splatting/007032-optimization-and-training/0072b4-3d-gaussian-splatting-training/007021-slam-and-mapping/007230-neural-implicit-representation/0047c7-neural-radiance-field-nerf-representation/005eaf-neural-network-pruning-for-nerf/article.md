# Neural Network Pruning for NeRF

Also called: Global Pruning, Magnitude Pruning, Model Pruning, Neural Network Pruning for 3D Scene Compression

## Overview
This method compresses a Neural Radiance Field (NeRF) by removing weights with the smallest magnitudes from the fully-connected neural network representing the scene, thereby reducing storage size and inference time [S1, unofficial]. Unlike traditional 3D compression which stores and compresses redundant 2D images, this approach compresses the functional representation directly, avoiding the inefficiency of decoding multiple images before rendering [S1, unofficial]. It is distinguished from sibling methods like [Scene NeRF](12484-scene-nerf.md) and [NeRF-OSR](14477-nerf-osr.md) by its reliance on unstructured magnitude pruning of the MLP weights rather than structural changes to the scene representation or octree-based optimizations.

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF model, defined as a fully-connected neural network (MLP) that maps a 5D coordinate $(x, y, z, \theta, \phi)$ to volume density $\sigma$ and view-dependent emitted radiance $c$ [S1, unofficial].
*   **Output:** A pruned NeRF model with reduced weight count and a retrained version of the pruned model [S1, unofficial].

**Data Structures:**
The NeRF model consists of fully-connected layers. The weights $W$ are stored in standard dense tensor formats. Pruning creates a sparse mask $M$ where $M_{ij} = 1$ if the weight is kept, and $0$ if removed. The compressed model stores only the non-zero weights and their indices [S1, unofficial].

**Parameters:**
*   **Pruning Ratio ($p$):** The fraction of weights to remove. Typical values used in experiments are 30%, 50%, 70%, and 90% [S1, unofficial].
*   **Pruning Strategy:** Global magnitude pruning. A single threshold $\tau$ is calculated across all layers of the network. Weights with absolute magnitude $|w| < \tau$ are set to zero [S1, unofficial].
*   **Retraining Duration:** 500 iterations [S1, unofficial].
*   **Retraining Hyperparameters:** Identical to the hyperparameters of the corresponding iterations in the original NeRF paper [S1, unofficial].

**Steps:**
1.  **Calculate Threshold:** Compute the pruning threshold $\tau$ for the global magnitude pruning. Sort all weights $w$ in the network by absolute magnitude $|w|$. Select the threshold such that the bottom $p$ percent of weights are below $\tau$.
2.  **Apply Mask:** Create a binary mask $M$ where $M_{ij} = 0$ if $|W_{ij}| < \tau$, and $M_{ij} = 1$ otherwise. Update the weights: $W_{pruned} = W \odot M$ (element-wise multiplication).
3.  **Initialize Retraining:** Load the pruned weights $W_{pruned}$ as the initialization for the training process.
4.  **Retrain:** Perform gradient descent optimization for 500 iterations using the original NeRF training loss (MSE between rendered and ground truth colors) and the original hyperparameters [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def prune_and_retrain(nerf_model, pruning_ratio, retrain_iterations=500):
    """
    Compress a NeRF model using global magnitude pruning and retrain.
    
    Args:
        nerf_model: The trained NeRF MLP model.
        pruning_ratio: Float between 0 and 1, fraction of weights to remove.
        retrain_iterations: Number of optimization steps for retraining.
        
    Returns:
        retrained_model: The pruned and retrained NeRF model.
    """
    # 1. Collect all weights
    all_weights = []
    for param in nerf_model.parameters():
        all_weights.extend(param.data.abs().flatten().tolist())
        
    # 2. Determine threshold for global pruning
    # Sort weights to find the value at the pruning_ratio percentile
    all_weights.sort()
    threshold_index = int(len(all_weights) * pruning_ratio)
    threshold = all_weights[threshold_index] if threshold_index < len(all_weights) else 0.0
    
    # 3. Apply pruning mask to each layer
    for param in nerf_model.parameters():
        mask = (param.data.abs() >= threshold).float()
        param.data = param.data * mask
        
    # 4. Retrain the pruned model
    # Note: In a real implementation, this involves setting up the optimizer
    # and the NeRF rendering loss function.
    # The hyperparameters (learning rate, etc.) must match the original NeRF paper
    # for the corresponding iteration count.
    optimizer = torch.optim.Adam(nerf_model.parameters(), lr=5e-4) # [G] Typical NeRF LR
    
    for i in range(retrain_iterations):
        optimizer.zero_grad()
        # Render a batch of rays using the pruned model
        # This involves sampling points along rays and passing them through the MLP
        rendered_colors = render_rays(nerf_model, ray_batch) 
        # Calculate loss against ground truth colors
        loss = mse_loss(rendered_colors, ray_batch.gt_colors)
        loss.backward()
        optimizer.step()
        
    return nerf_model
```

**Edge Cases and Pitfalls:**
*   **Artifacts without Retraining:** Pruning without retraining causes visible artifacts in geometry and coloring starting at 50% pruning ratio, with disturbing quality degradation at 70% and severe artifacts at 90% [S1, unofficial].
*   **Retraining Limitations:** Retraining fixes most artifacts, but quality remains significantly degraded at 90% pruning ratio [S1, unofficial].
*   **Global vs. Structured:** Global pruning is chosen over structured pruning for flexibility in adjusting sparsity across different layers, yielding a better performance-compression trade-off [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The storage size is reduced by a factor of $1/(1-p)$, where $p$ is the pruning ratio. For example, 90% pruning yields a compression factor of $\times 10.00$ [S1, unofficial].
*   **Inference Time:** Pruning speeds up inference compared to other neural network compression approaches, providing faster rendering [S1, unofficial].

**Measured Performance:**
*   **Fern Dataset [S1, unofficial]:**
    *   Original: PSNR 24.73, MSE 0.0066, Compression $\times 1.00$.
    *   30% Pruned: PSNR 25.41, MSE 0.0067, Compression $\times 1.43$.
    *   30% Retrained: PSNR 25.45, MSE 0.0062, Compression $\times 1.43$.
    *   50% Pruned: PSNR 22.48, MSE 0.0116, Compression $\times 2.00$.
    *   50% Retrained: PSNR 24.78, MSE 0.0071, Compression $\times 2.00$.
    *   70% Pruned: PSNR 20.97, MSE 0.0165, Compression $\times 3.33$.
    *   70% Retrained: PSNR 22.51, MSE 0.0134, Compression $\times 3.33$.
    *   90% Pruned: PSNR 16.87, MSE 0.0706, Compression $\times 10.00$.
    *   90% Retrained: PSNR 20.25, MSE 0.0318, Compression $\times 10.00$.
*   **Lego Dataset [S1, unofficial]:**
    *   Original: PSNR 31.70, MSE 0.0028, Compression $\times 1.00$.
    *   30% Pruned: PSNR 29.14, MSE 0.0034, Compression $\times 1.43$.
    *   30% Retrained: PSNR 30.43, MSE 0.0029, Compression $\times 1.43$.
    *   50% Pruned: PSNR 25.32, MSE 0.0077, Compression $\times 2.00$.
    *   50% Retrained: PSNR 26.97, MSE 0.0051, Compression $\times 2.00$.
    *   70% Pruned: PSNR 21.70, MSE 0.0144, Compression $\times 3.33$.
    *   70% Retrained: PSNR 25.26, MSE 0.0085, Compression $\times 3.33$.
    *   90% Pruned: PSNR 15.99, MSE 0.0786, Compression $\times 10.00$.
    *   90% Retrained: PSNR 20.42, MSE 0.0352, Compression $\times 10.00$.

**Strengths and Limitations:**
*   **Strengths:** Intuitive implementation; speeds up inference; avoids redundancy of storing 2D images [S1, unofficial].
*   **Limitations:** High pruning ratios (e.g., 90%) lead to severe quality loss even after retraining; requires retraining to mitigate artifacts [S1, unofficial].

**When to Use:**
Use this method when storage size and inference speed are critical constraints for a NeRF application, and moderate compression ratios (30-70%) are acceptable. Avoid for very high compression ratios (>90%) due to significant quality degradation.

## Background
This method was introduced in a 2021 project by Stanford CS 231A students [S1, unofficial]. It applies neural network compression techniques, specifically magnitude pruning, to the regression task of rendering 3D scenes, extending the success of pruning in classification tasks to neural functional representations [S1, unofficial]. It is applied to NeRF models trained on the Lego and Fern datasets [S1, unofficial].

## Sources
[S1] Neural 3D Scene Compression via Model Compression, https://doi.org/10.48550/arxiv.2105.03120, fetched 2026-10-28 (unofficial)
