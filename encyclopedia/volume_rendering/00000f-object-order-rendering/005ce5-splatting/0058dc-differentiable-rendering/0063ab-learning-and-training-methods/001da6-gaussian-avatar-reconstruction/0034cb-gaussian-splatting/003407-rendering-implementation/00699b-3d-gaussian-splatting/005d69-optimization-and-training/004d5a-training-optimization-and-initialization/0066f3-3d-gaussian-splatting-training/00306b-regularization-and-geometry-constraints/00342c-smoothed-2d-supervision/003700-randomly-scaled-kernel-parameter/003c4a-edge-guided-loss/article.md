# Edge Guided Loss

Also called: Edge-Guided Loss Mask, Edge-aware loss for pose alignment

## Overview
Edge Guided Loss is a training regularization technique that amplifies gradient signals on edge pixels to improve the robustness and stability of joint optimization in radiance field reconstruction [S1][S2, unofficial]. It specifically addresses the "weak and noisy gradient problem" caused by aggressive smoothing or filtering of supervision images, which can otherwise lead to pose drift and poor convergence [S2, unofficial]. Unlike standard pixel-wise losses that treat all regions equally, this method dynamically prioritizes high-frequency structural information, ensuring that the optimizer focuses on geometrically significant areas during pose alignment [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Rendered 2D image $I'$ and ground truth 2D image $I$ (both filtered/smoothed).
*   **Output:** A modified loss value or gradient update that emphasizes edge regions.

**Data Structures:**
*   **Edge Mask:** A binary or weighted map of the same dimensions as the image ($H \times W$), indicating edge pixels.
*   **Learning Rate Multiplier:** A scalar factor applied to the loss gradient for edge pixels.

**Steps:**
1.  **Edge Detection:** Compute the edge mask by applying a Sobel filter to the filtered 2D images [S2, unofficial]. The Sobel operator calculates horizontal ($G_x$) and vertical ($G_y$) gradients. Pixels with gradient magnitudes exceeding a threshold (or the top percentile) are classified as edge pixels [S2, unofficial].
    *   Sobel Kernels:
        $$ S_x = \begin{bmatrix} -1 & -2 & -1 \\ 0 & 0 & 0 \\ 1 & 2 & 1 \end{bmatrix}, \quad S_y = \begin{bmatrix} -1 & 0 & 1 \\ -2 & 0 & 2 \\ -1 & 0 & 1 \end{bmatrix} $$
    *   Gradient Magnitude at $(x,y)$: $M(x,y) = \sqrt{G_x(x,y)^2 + G_y(x,y)^2}$ [G: Standard Euclidean norm].
2.  **Mask Generation:** Create a binary mask $E(x,y)$ where $E(x,y) = 1$ if $(x,y)$ is an edge pixel, and $0$ otherwise [S2, unofficial].
3.  **Loss Calculation:** Compute the base rendering loss (e.g., L1 or L2) between $I'$ and $I$ [S2, unofficial].
4.  **Gradient Amplification:** Multiply the loss contribution of edge pixels by a factor $\alpha$ [S2, unofficial].
    *   Typical value: $\alpha = 1.5$ [S2, unofficial].
5.  **Alternating Schedule:** Apply the edge-guided scaling alternately on every other training iteration [S2, unofficial]. This prevents the optimizer from becoming overly biased toward edges in a single step, allowing for balanced refinement [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def edge_guided_loss(rendered_img, ground_truth_img, iteration, edge_scale=1.5):
    """
    Computes the rendering loss with edge-guided weighting.
    
    Args:
        rendered_img: Tensor of shape (H, W, C), the rendered image.
        ground_truth_img: Tensor of shape (H, W, C), the ground truth image.
        iteration: Integer, the current training iteration index.
        edge_scale: Float, multiplier for edge pixels (default 1.5).
        
    Returns:
        Float, the weighted loss value.
    """
    # Step 1: Detect edges using Sobel filter
    # Compute gradients for rendered image
    grad_x_rendered = sobel_filter(rendered_img, kernel='horizontal')
    grad_y_rendered = sobel_filter(rendered_img, kernel='vertical')
    
    # Compute gradients for ground truth
    grad_x_gt = sobel_filter(ground_truth_img, kernel='horizontal')
    grad_y_gt = sobel_filter(ground_truth_img, kernel='vertical')
    
    # Compute gradient magnitudes
    mag_rendered = sqrt(grad_x_rendered**2 + grad_y_rendered**2)
    mag_gt = sqrt(grad_x_gt**2 + grad_y_gt**2)
    
    # Combine magnitudes (average of rendered and GT magnitude for robustness)
    # Note: Source [S2, unofficial] implies detection on filtered images; using GT for mask 
    # is standard practice to avoid noise in rendered edges, but source says 
    # "detected using Sobel filter on the filtered 2D images". 
    # We will use the rendered image's edge structure as it is the signal being optimized.
    # [G: Using rendered image edges is consistent with "edge region... from which 
    # learning signal... comes"].
    edge_mask = (mag_rendered > threshold).float() 
    # Note: Threshold value not explicitly given in sources. 
    # [G: A common heuristic is top 10-20% of pixels or a fixed magnitude cutoff. 
    # Implementation requires a specific threshold choice.]
    
    # Step 2: Compute base loss (L1)
    base_loss = abs(rendered_img - ground_truth_img).mean()
    
    # Step 3: Apply edge-guided scaling alternately
    # Apply edge scale on even iterations (or odd, as long as it alternates)
    if iteration % 2 == 0:
        # Compute edge-specific loss
        edge_loss = (edge_mask * abs(rendered_img - ground_truth_img)).mean()
        # Non-edge loss
        non_edge_loss = ((1 - edge_mask) * abs(rendered_img - ground_truth_img)).mean()
        
        # Combine with amplified edge weight
        # [G: The source says "increase the learning rate by 1.5x on pixels in the edge region".
        # This is equivalent to weighting the loss term for those pixels.]
        weighted_loss = edge_loss * edge_scale + non_edge_loss
        return weighted_loss
    else:
        # Standard loss on alternating iterations
        return base_loss

def sobel_filter(image, kernel):
    """
    Applies Sobel filter to an image.
    """
    if kernel == 'horizontal':
        s_x = [[-1, -2, -1], [0, 0, 0], [1, 2, 1]]
    else:
        s_y = [[-1, 0, 1], [-2, 0, 2], [-1, 0, 1]]
        
    # Convolve image with kernel
    # [G: Standard 2D convolution with padding='same' or boundary handling]
    filtered = convolve_2d(image, s_x if kernel == 'horizontal' else s_y)
    return filtered
```

**Parameters:**
*   **Edge Scale ($\alpha$):** 1.5 [S2, unofficial].
*   **Alternation Frequency:** Every other iteration [S2, unofficial].
*   **Edge Threshold:** Not given in the sources. [G: Implementer must choose a threshold (e.g., top 10% magnitude) or a fixed value based on image intensity range.]

**Edge Cases and Pitfalls:**
*   **Noise Sensitivity:** If the input images are not sufficiently filtered, the Sobel filter may detect noise as edges, leading to unstable gradients [S2, unofficial]. This is why it is used in conjunction with "Smoothed 2D Supervision" and "Randomly Scaled Kernel Parameter" [S1][S2, unofficial].
*   **Over-weighting:** Excessively high edge weights can cause the optimizer to ignore texture details in flat regions, potentially leading to artifacts in smooth areas [S2, unofficial].
*   **Boundary Conditions:** The Sobel filter requires careful handling of image boundaries to avoid edge artifacts in the mask [G: Padding or boundary reflection should be used].

## Subtopics

### Selective Gradient Loss
Selective Gradient Loss (SGL) is a variant that uses gradient maps and dynamic region selection to focus training on difficult-to-render textures and structures [S3]. Unlike Edge Guided Loss which uses a fixed magnitude threshold, SGL computes horizontal ($l_x$) and vertical ($l_y$) gradient discrepancies using the Euclidean metric averaged over image dimensions $H, W$:
$$ l_x = \frac{1}{\sqrt{HW}} \sum_{x=0}^{H-1} \sum_{y=0}^{W-1} |G'_x(x,y) - G_x(x,y)| $$
$$ l_y = \frac{1}{\sqrt{HW}} \sum_{x=0}^{H-1} \sum_{y=0}^{W-1} |G'_y(x,y) - G_y(x,y)| $$
It then uses the rendering error map (absolute difference between gradient maps of rendered and ground truth) as a weight map to dynamically prioritize regions with larger rendering errors [S3]. The final loss is $L_s = w_x * l_x + w_y * l_y$, where $w_x, w_y$ are the weight maps [S3]. This addresses the issue where flat regions dominate naive gradient loss [S3].

## Operating characteristics
*   **Time Complexity:** Adds minimal overhead. Edge detection via Sobel filter is $O(HW)$ and is lightweight [S1]. The alternating schedule means the extra computation is applied only half the time [S2, unofficial].
*   **Memory Complexity:** Requires storing the edge mask and gradient maps, which is $O(HW)$ [S3].
*   **Performance:** Experimentally proven crucial for successful pose refinement in ablation studies [S1]. It enables rapid convergence, reducing training iterations from 200k (typical for MLP-based methods) to 50k [S1].
*   **Strengths:** Mitigates weak and noisy gradients caused by aggressive filtering [S2, unofficial]. Improves robustness of joint pose optimization [S1].
*   **Limitations:** Requires careful tuning of the edge scale factor and threshold. May not be effective if the input images lack clear edges or are heavily noisy [S2, unofficial].
*   **When to use:** When using smoothed supervision or aggressive filtering that destroys high-frequency details, leading to pose drift [S2, unofficial].
*   **When not to use:** When the scene is very smooth with few edges, or when computational resources are extremely constrained and the overhead of edge detection is prohibitive (though the overhead is low).

## Background
Edge Guided Loss was introduced in the context of joint optimization of camera poses and decomposed low-rank tensorial radiance fields (TensoRF) [S1]. It was proposed to improve the robustness and stability of the optimization process, specifically addressing the challenges of pose refinement when using smoothed 2D supervision [S1]. The technique was developed by researchers associated with the "Joint-TensoRF" framework [S1]. It is part of a suite of techniques including smoothed 2D supervision and randomly scaled kernel parameters [S1].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.1609/aaai.v38i2.27859, fetched 2026-10-26
[S2] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
[S3] SOGS: Second-Order Anchor for Advanced 3D Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.01043, fetched 2026-10-30
