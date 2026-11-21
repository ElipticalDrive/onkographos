# Scalar Value Sensitivity Fields

Also called: SVSF

## Overview
Scalar Value Sensitivity Fields (SVSF) are visualization techniques that map the sensitivity of a volume-rendered image to modifications in the opacity transfer function (TF) domain, conditioned on specific scalar values [S1]. They solve the problem of predicting which spatial regions of a volume will change appearance when a user edits a specific range of the TF, allowing for informed interactive exploration [S1]. Unlike Region Sensitivity Plots, which show TF sensitivity conditioned on image regions, SVSFs show image sensitivity conditioned on TF scalar values [S1].

## How it works
**Inputs:**
*   A volume dataset and its current rendering.
*   An opacity transfer function $T(v)$ mapping scalar values $v \in [0, 255]$ to opacity $\alpha \in [0, 1]$.
*   A grid resolution parameter $r$.
*   A selected scalar value index $i$ (or range) in the TF domain.
*   A Gaussian filter bandwidth for visualization smoothing.

**Outputs:**
*   A 3-tensor $\Sigma \in \mathbb{R}^{256 \times r \times r}$, where each slice $\Sigma(i, \cdot, \cdot)$ is a 2D field representing the sensitivity of image blocks to changes in the TF at scalar value index $i$.

**Data Structures:**
*   **Sensitivity Tensor ($\Sigma$):** A 3-dimensional array. The first dimension corresponds to the 256 possible scalar values of the input volume. The second and third dimensions correspond to the $r \times r$ grid blocks covering the image domain.
*   **Image Grid:** The rendered image is divided into $r \times r$ non-overlapping blocks.

**Algorithm:**
1.  **Grid Definition:** Define the grid resolution $r$. The sources specify typical values of $r=8$ or $r=16$ to balance latency and detail. Setting $r=256$ (pixel-level sensitivity) is noted as prohibitively costly [S1].
2.  **Block Division:** Divide the rendered image domain into $r \times r$ blocks.
3.  **Sensitivity Computation:** For each scalar value index $i \in [0, 255]$ and each block $(u, v)$ in the grid:
    *   Compute the TF sensitivity using the gradient of the rendered image with respect to the opacity TF value at index $i$. The sources reference "Equation 11" for this calculation, which represents the partial derivative of the image pixel values with respect to the opacity parameter $T(i)$ [S1].
    *   Store the result in $\Sigma(i, u, v)$.
4.  **Acceleration:** Perform the backpropagation required to compute these gradients in parallel on the GPU using minibatches of size 64 [S1].
5.  **Visualization (Gaussian Filtering):** To visualize the impact of a user-selected TF region (which typically affects a localized range of scalar values), apply a Gaussian filter to the sequence of fields $\Sigma$. The filter is centered at the selected scalar value $i_{sel}$ with a specified bandwidth. The weight of each field $\Sigma(i, \cdot, \cdot)$ in the superposition is determined by the Gaussian kernel centered at $i_{sel}$ [S1].
6.  **Display:** Color-map the resulting sensitivity field based on the global range of the field values. Encode the specific range of the user's selection via opacity mapping to provide local context [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_scalar_value_sensitivity_fields(volume_rendering, opacity_tf, grid_resolution=16):
    """
    Computes the Scalar Value Sensitivity Field tensor.
    
    Args:
        volume_rendering: The current rendered image (H x W x 3).
        opacity_tf: Array of 256 opacity values T[0...255].
        grid_resolution: Integer r, typically 8 or 16.
        
    Returns:
        sigma: Tensor of shape (256, r, r).
    """
    # Initialize the 3-tensor Sigma
    # Dimensions: 256 scalar values, r x r image blocks
    sigma = np.zeros((256, grid_resolution, grid_resolution))
    
    # Determine block size in pixels
    height, width, _ = volume_rendering.shape
    block_h = height // grid_resolution
    block_w = width // grid_resolution
    
    # Compute sensitivity for each scalar value index
    # Accelerated via GPU backpropagation in minibatches of 64 [S1]
    for i in range(0, 256, 64):
        batch_indices = range(i, min(i + 64, 256))
        
        for idx in batch_indices:
            # Compute gradient of the rendered image with respect to opacity_tf[idx]
            # This corresponds to Equation 11 in the source [S1]
            # d_image / d_T(idx)
            sensitivity_map = compute_gradient_wrt_opacity(volume_rendering, opacity_tf, idx)
            
            # Aggregate sensitivity into grid blocks
            for u in range(grid_resolution):
                for v in range(grid_resolution):
                    y_start = u * block_h
                    y_end = (u + 1) * block_h
                    x_start = v * block_w
                    x_end = (v + 1) * block_w
                    
                    # Average sensitivity over the block
                    block_sensitivity = np.mean(sensitivity_map[y_start:y_end, x_start:x_end])
                    sigma[idx, u, v] = block_sensitivity
                    
    return sigma

def visualize_sensitivity(sigma, selected_scalar_value, bandwidth=5.0):
    """
    Visualizes the sensitivity field by applying a Gaussian filter centered at the selected scalar value.
    
    Args:
        sigma: The computed sensitivity tensor (256, r, r).
        selected_scalar_value: The index i in the TF domain the user is hovering over.
        bandwidth: The standard deviation for the Gaussian filter.
        
    Returns:
        visual_field: The final 2D sensitivity map for display.
    """
    # Apply Gaussian filter to the sequence of fields along the first dimension
    # The weight for each field sigma[i] is determined by the Gaussian kernel
    # centered at selected_scalar_value [S1]
    gaussian_weights = np.exp(-0.5 * ((np.arange(256) - selected_scalar_value) / bandwidth) ** 2)
    
    # Superimpose the fields weighted by the Gaussian
    # Sum over the scalar value dimension
    visual_field = np.zeros((sigma.shape[1], sigma.shape[2]))
    for i in range(256):
        visual_field += gaussian_weights[i] * sigma[i]
        
    # Normalize and map to display colors based on global range [S1]
    # Encode user selection range via opacity mapping [S1]
    return visual_field
```

**Edge Cases and Pitfalls:**
*   **Performance:** Computing pixel-level sensitivity ($r=256$) requires $256^2$ backpropagations, which is prohibitively costly [S1]. Use $r=8$ or $16$.
*   **Global vs. Local Context:** The field sensitivity for a specific edit is often small relative to the global sensitivity. Visualization must encode the user's specific selection range via opacity to maintain context [S1].

## Operating characteristics
*   **Time Complexity:** The computation requires backpropagation for each of the 256 scalar values. With minibatch processing on the GPU, the latency is manageable for interactive use, but scales linearly with the number of scalar values and the cost of the rendering backpropagation [S1].
*   **Memory Complexity:** The output is a fixed-size tensor $\mathbb{R}^{256 \times r \times r}$. For $r=16$, this is $256 \times 16 \times 16 = 65,536$ floats, which is negligible.
*   **Strengths:** Provides immediate visual feedback on which parts of the volume will change when a specific TF value is edited. Complements Region Sensitivity Plots by offering the inverse perspective (TF-conditioned vs. Image-conditioned) [S1].
*   **Limitations:** Relies on the accuracy of the gradient computation (backpropagation) through the rendering pipeline. The block-based aggregation ($r < 256$) loses pixel-level precision [S1].

## Background
Scalar Value Sensitivity Fields were introduced in the context of a generative model for volume rendering, specifically within the framework of exploring the opacity transfer function latent space [S1]. The method was developed to help users understand the relationship between the high-dimensional latent space of opacity TFs and the resulting rendered images, facilitating intuitive editing and exploration [S1].

## Sources
[S1] A Generative Model for Volume Rendering, https://doi.org/10.1109/tvcg.2018.2816059, fetched 2026-10-26
