# Edge Guided Loss

Also called: Edge-Guided Loss Mask, Edge-aware loss for pose alignment

## Overview
Edge Guided Loss is a training regularization technique that amplifies gradient signals on edge pixels to improve the robustness and stability of joint optimization in radiance field reconstruction [S1][S2, unofficial]. It specifically addresses the "weak and noisy gradient problem" caused by aggressive smoothing or filtering of supervision images, which can otherwise lead to pose drift and poor convergence [S2, unofficial]. Unlike standard pixel-wise losses that treat all regions equally, this method dynamically prioritizes high-frequency structural information, ensuring that the optimizer focuses on geometrically significant areas during pose alignment [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Rendered 2D image $I'$ and ground truth 2D image $I$ (both filtered/smoothed) [S2, unofficial].
*   **Output:** A modified loss value or gradient update that emphasizes edge regions [S2, unofficial].

**Data Structures:**
*   **Edge Mask:** A binary map of the same dimensions as the image ($H \times W$), indicating edge pixels [S2, unofficial].
*   **Learning Rate Multiplier:** A scalar factor applied to the loss gradient for edge pixels [S2, unofficial].

**Steps:**
1.  **Edge Detection:** Compute the edge mask by applying a Sobel filter to the filtered 2D images [S2, unofficial]. The Sobel operator calculates horizontal ($G_x$) and vertical ($G_y$) gradients [S2, unofficial]. Pixels with gradient magnitudes exceeding a threshold are classified as edge pixels [S2, unofficial].
    *   Sobel Kernels:
        $$ S_x = \begin{bmatrix} -1 & -2 & -1 \\ 0 & 0 & 0 \\ 1 & 2 & 1 \end{bmatrix}, \quad S_y = \begin{bmatrix} -1 & 0 & 1 \\ -2 & 0 & 2 \\ -1 & 0 & 1 \end{bmatrix} $$ [S2, unofficial].
    *   Gradient Magnitude at $(x,y)$: $M(x,y) = \sqrt{G_x(x,y)^2 + G_y(x,y)^2}$ [G: Standard Euclidean norm].
2.  **Mask Generation:** Create a binary mask $E(x,y)$ where $E(x,y) = 1$ if $(x,y)$ is an edge pixel, and $0$ otherwise [S2, unofficial].
3.  **Loss Calculation:** Compute the rendering loss between $I'$ and $I$ [S2, unofficial].
4.  **Gradient Amplification:** Multiply the loss contribution of edge pixels by a factor $\alpha$ [S2, unofficial].
    *   Typical value: $\alpha = 1.5$ [S2, unofficial].
5.  **Alternating Schedule:** Apply the edge-guided scaling alternately on every other training iteration [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def edge_guided_loss(rendered_img, ground_truth_img, iteration, edge_scale=1.5, threshold=0.0):
    """
    Computes the rendering loss with edge-guided weighting.
    
    Args:
        rendered_img: Tensor of shape (H, W, C), the rendered image.
        ground_truth_img: Tensor of shape (H, W, C), the ground truth image.
        iteration: Integer, the current training iteration index.
        edge_scale: Float, multiplier for edge pixels (default 1.5) [S2, unofficial].
        threshold: Float, magnitude cutoff for edge detection [G].
        
    Returns:
        Float, the weighted loss value.
    """
    # Step 1: Edge Detection via Sobel filter on filtered images [S2, unofficial]
    gx_r = sobel_convolve(rendered_img, kernel_x=True)
    gy_r = sobel_convolve(rendered_img, kernel_y=True)
    mag_r = sqrt(gx_r**2 + gy_r**2)
    edge_mask = (mag_r > threshold).float()

    # Step 2: Base rendering loss [S2, unofficial]
    base_loss = abs(rendered_img - ground_truth_img).mean()

    # Step 3: Alternating schedule [S2, unofficial]
    if iteration % 2 == 0:
        # Amplify gradient signal on edge pixels [S2, unofficial]
        edge_loss = (edge_mask * abs(rendered_img - ground_truth_img)).mean()
        non_edge_loss = ((1 - edge_mask) * abs(rendered_img - ground_truth_img)).mean()
        return edge_loss * edge_scale + non_edge_loss
    else:
        return base_loss

def sobel_convolve(image, kernel_x):
    """
    Applies a 3x3 Sobel convolution with zero-padding for boundaries [G].
    """
    if kernel_x:
        kernel = [[-1, -2, -1], [0, 0, 0], [1, 2, 1]]
    else:
        kernel = [[-1, 0, 1], [-2, 0, 2], [-1, 0, 1]]
    H, W = image.shape[:2]
    padded = zero_pad(image, 1)
    out = zeros_like(image)
    for i in range(H):
        for j in range(W):
            patch = padded[i:i+3, j:j+3]
            out[i, j] = sum(patch[r, c] * kernel[r, c] for r in range(3) for c in range(3))
    return out
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
*   **Time Complexity:** Adds minimal overhead, as edge detection via Sobel filter is lightweight [S1]. The alternating schedule means the extra computation is applied only half the time [S2, unofficial].
*   **Performance:** Experimentally proven crucial for successful pose refinement in ablation studies [S1]. It enables rapid convergence, reducing training iterations from 200k (typical for MLP-based methods) to 50k [S1].
*   **Strengths:** Mitigates weak and noisy gradients caused by aggressive filtering [S2, unofficial]. Improves robustness of joint pose optimization [S1].
*   **Limitations:** Requires careful tuning of the edge scale factor and threshold. May not be effective if the input images lack clear edges or are heavily noisy [S2, unofficial].
*   **When to use:** When using smoothed supervision or aggressive filtering that destroys high-frequency details, leading to pose drift [S2, unofficial].

## Background
Edge Guided Loss was introduced in the context of joint optimization of camera poses and decomposed low-rank tensorial radiance fields (TensoRF) [S1]. It was proposed to improve the robustness and stability of the optimization process, specifically addressing the challenges of pose refinement when using smoothed 2D supervision [S1]. The technique was developed by researchers associated with the "Joint-TensoRF" framework [S1]. It is part of a suite of techniques including smoothed 2D supervision and randomly scaled kernel parameters [S1].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.1609/aaai.v38i2.27859, fetched 2026-10-26
[S2] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
[S3] SOGS: Second-Order Anchor for Advanced 3D Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.01043, fetched 2026-10-30
