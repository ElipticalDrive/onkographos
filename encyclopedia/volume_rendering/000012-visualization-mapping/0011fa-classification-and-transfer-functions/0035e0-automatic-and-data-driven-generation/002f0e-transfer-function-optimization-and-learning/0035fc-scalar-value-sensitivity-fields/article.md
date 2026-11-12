# Scalar Value Sensitivity Fields


## Overview
Scalar Value Sensitivity Fields (SVSF) are visualization techniques that map the sensitivity of a volume-rendered image to modifications in the opacity transfer function (TF) domain, conditioned on specific scalar values [S1]. They solve the problem of predicting which spatial regions of a volume will change appearance when a user edits a specific range of the TF, thereby aiding in the interactive exploration and tuning of classification parameters [S1]. Unlike Region Sensitivity Plots, which show TF sensitivity conditioned on image regions, SVSFs show image sensitivity conditioned on TF scalar values [S1].

## How it works
**Inputs:**
*   A volume dataset and its associated rendering pipeline (typically involving a generative model or differentiable renderer capable of backpropagation) [S1].
*   A grid resolution parameter $r$ [S1].
*   A selected scalar value index $i$ from the TF domain (typically 0–255) [S1].
*   A Gaussian filter bandwidth for visualization smoothing [S1].

**Outputs:**
*   A 3-tensor $\Sigma \in \mathbb{R}^{256 \times r \times r}$, where each slice $\Sigma(i, \cdot, \cdot)$ represents the sensitivity field for scalar value index $i$ [S1].
*   A visualized sensitivity map highlighting image regions likely to change upon TF modification at the selected scalar value [S1].

**Data Structures:**
*   **Sensitivity Tensor ($\Sigma$):** A 3-dimensional array of size $256 \times r \times r$ [S1]. The first dimension corresponds to the 256 possible scalar values [S1]. The last two dimensions correspond to the spatial grid of the image, divided into $r \times r$ blocks [S1].
*   **Image Blocks:** The rendered image is partitioned into $r \times r$ non-overlapping blocks [S1].

**Steps:**
1.  **Grid Definition:** Define the grid resolution $r$. Typical values are $r=8$ or $r=16$ to balance computational latency and spatial detail [S1]. Setting $r=256$ (pixel-level sensitivity) is computationally prohibitive [S1].
2.  **Image Partitioning:** Divide the rendered image domain into $r \times r$ blocks [S1].
3.  **Sensitivity Computation:** For each scalar value index $i \in [0, 255]$: Compute the TF sensitivity for each block using backpropagation through the rendering pipeline [S1]. This corresponds to Equation 11 in the source [S1]. The result for index $i$ is a 2D field $\Sigma(i, \cdot, \cdot) \in \mathbb{R}^{r \times r}$ [S1].
4.  **Tensor Assembly:** Assemble all 256 fields into the 3-tensor $\Sigma \in \mathbb{R}^{256 \times r \times r}$ [S1].
5.  **Acceleration:** Perform the backpropagation required for sensitivity calculation in parallel on the GPU using minibatches of size 64 [S1].
6.  **Visualization:** When a user selects a specific scalar value or range, apply a Gaussian filter to the sequence of fields centered at the selected scalar value with a specified bandwidth [S1]. Superimpose the filter weight for each field onto the Sensitivity Plot in red [S1]. Color-map the sensitivity based on the global range of the field [S1]. Encode the range specific to the user's selection via opacity mapping to provide local context against global sensitivity [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def compute_scalar_value_sensitivity_fields(volume_data, tf_values, grid_resolution=16, batch_size=64):
    """
    Computes the Scalar Value Sensitivity Field tensor.
    
    Args:
        volume_data: Input volume tensor.
        tf_values: Transfer function values (size 256).
        grid_resolution: r, the number of blocks per image dimension.
        batch_size: Number of scalar values to process in parallel.
        
    Returns:
        sigma: Sensitivity tensor of shape (256, r, r).
    """
    # Initialize output tensor
    sigma = torch.zeros(256, grid_resolution, grid_resolution)
    num_scalar_values = 256
    
    # Process in batches to save memory/time [S1]
    for start_idx in range(0, num_scalar_values, batch_size):
        end_idx = min(start_idx + batch_size, num_scalar_values)
        current_batch_indices = torch.arange(start_idx, end_idx)
        
        # Compute gradients for the current batch of TF values via backpropagation [S1]
        # The exact gradient computation follows Equation 11 in the source [S1]
        # render_fn is the differentiable rendering pipeline [G]
        image = render_fn(volume_data, tf_values)
        gradients = torch.autograd.grad(
            outputs=image, 
            inputs=tf_values[current_batch_indices], 
            grad_outputs=torch.ones_like(image), 
            create_graph=True
        )[0]
        
        # gradients shape: (batch_size, H, W) assuming grayscale or mean opacity
        img_h, img_w = gradients.shape[1], gradients.shape[2]
        block_h, img_w // grid_resolution, img_w // grid_resolution
        
        # Aggregate gradients into blocks by averaging over each block region [G]
        gradients_4d = gradients.unsqueeze(1)
        block_gradients = F.avg_pool2d(gradients_4d, kernel_size=(block_h, block_w), stride=(block_h, block_w))
        block_gradients = block_gradients.squeeze(1)
        
        # Assign to sigma tensor
        sigma[start_idx:end_idx, :, :] = block_gradients
        
    return sigma

def visualize_sensitivity_field(sigma, selected_scalar_value, bandwidth=1.0):
    """
    Visualizes the sensitivity field for a selected scalar value.
    
    Args:
        sigma: Sensitivity tensor (256, r, r).
        selected_scalar_value: The index i in the TF domain.
        bandwidth: Gaussian filter bandwidth.
        
    Returns:
        visual_map: 2D array of sensitivity values for the selected value.
    """
    # Extract the field for the selected scalar value
    field = sigma[selected_scalar_value, :, :]
    
    # Apply Gaussian filter to smooth the field [S1]
    field_4d = field.unsqueeze(0).unsqueeze(0)
    kernel_size = int(2 * bandwidth + 1)
    gaussian_kernel = create_gaussian_kernel(kernel_size, bandwidth)
    
    # Apply convolution
    smoothed_field = F.conv2d(field_4d, gaussian_kernel, padding=kernel_size//2)
    
    # Squeeze back to 2D
    visual_map = smoothed_field.squeeze(0).squeeze(0)
    return visual_map

def create_gaussian_kernel(kernel_size, sigma_param):
    """Creates a 2D Gaussian kernel."""
    coords = torch.arange(kernel_size).float() - kernel_size // 2
    g = torch.exp(-(coords ** 2) / (2 * sigma_param ** 2))
    g /= g.sum()
    g_2d = torch.outer(g, g)
    return g_2d.unsqueeze(0).unsqueeze(0)
```

**Edge Cases and Pitfalls:**
*   **Computational Cost:** Computing sensitivity at pixel-level resolution ($r=256$) is prohibitively expensive because it requires $256^2$ backpropagation passes. Use block aggregation ($r=8$ or $16$) [S1].
*   **Global vs. Local Context:** High global sensitivity might mask local changes. The visualization encodes user-specific selection ranges via opacity mapping to distinguish local impact from global sensitivity [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The dominant cost is backpropagation through the rendering pipeline for each scalar value (or batch thereof) [S1].
*   **Memory Complexity:** $O(256 \cdot r^2)$ to store the sensitivity tensor $\Sigma$ [S1].
*   **Performance:** GPU acceleration with minibatches of size 64 is used to mitigate latency [S1].
*   **Strengths:** Provides immediate visual feedback on where TF edits will have the most impact, aiding in precise tuning of opacity TFs [S1]. Complements Region Sensitivity Plots by offering the inverse perspective (TF-to-Image vs. Image-to-TF) [S1].
*   **Limitations:** Requires a differentiable rendering pipeline or generative model. High-resolution sensitivity is computationally prohibitive. The method is specific to opacity TFs and may not directly apply to color or emission TFs without extension [S1].

## Background
Scalar Value Sensitivity Fields were introduced in the context of generative models for volume rendering, specifically in the paper "A Generative Model for Volume Rendering" [S1]. The method leverages the latent space encoding of opacity TFs learned by a generative adversarial network (GAN) to compute sensitivities [S1]. It was developed to help users explore the opacity TF latent space and understand the relationship between TF modifications and rendered image changes [S1]. The technique is part of a broader interface that combines Region Sensitivity Plots and Scalar Value Sensitivity Fields for interactive TF design [S1].

## Sources
[S1] A Generative Model for Volume Rendering, https://doi.org/10.1109/tvcg.2018.2816059, fetched 2026-10-26
