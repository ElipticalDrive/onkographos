# Frequency Regularization

Also called: FreGS

## Overview
Frequency regularization is a training-time technique that constrains the spectral content of the rendered output to guide Gaussian densification toward high-frequency regions [S1, unofficial]. It solves the problem where standard 3D Gaussian Splatting (3DGS) initialization and densification may fail to capture fine geometric details or high-frequency textures, leading to blurry reconstructions [S1, unofficial]. Unlike [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md) which operates in the spatial domain via kernel smoothing, or [Depth Regularization](7616-depth-regularization.md) which enforces geometric consistency, frequency regularization operates in the spectral domain to explicitly encourage the representation of high-frequency signal components [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The rendered image $I_{render}$ from the current training step, and the ground truth image $I_{gt}$.
*   **Output:** A scalar regularization loss $\mathcal{L}_{freq}$ added to the total loss.

**Data Structures:**
No specific data structures are required beyond standard image buffers. The method operates on the pixel values of the rendered image.

**Steps:**
1.  **Compute Spectral Representation:** Apply a frequency transform to the rendered image $I_{render}$ and the ground truth $I_{gt}$. While the specific transform is not explicitly detailed in the primary source for the base method, it typically involves a Fourier Transform or a Wavelet Transform to decompose the image into frequency components [S1, unofficial][S2, unofficial].
2.  **Extract High-Frequency Components:** Isolate the high-frequency coefficients from the spectral representation. Let $H(I)$ denote the high-frequency component of an image $I$.
3.  **Compute Loss:** Calculate the difference between the high-frequency components of the rendered and ground truth images. A common formulation is the $L_1$ or $L_2$ norm of the difference in the frequency domain:
    $$ \mathcal{L}_{freq} = \| H(I_{render}) - H(I_{gt}) \|_1 $$
    Alternatively, the loss may penalize the magnitude of high-frequency errors directly to push Gaussians to cover areas with high-frequency detail [S1, unofficial].
4.  **Backpropagation:** The gradients of $\mathcal{L}_{freq}$ flow back through the differentiable rasterizer to update the Gaussian parameters $\theta$ (centers $\mu$, scales $s$, rotations $q$, opacities $\alpha$, and colors $c$) [Parent].

**Parameters:**
*   $\lambda_{freq}$: Weight for the frequency regularization term. Typical values are not given in the sources. [S1, unofficial][S2, unofficial]

**Edge Cases and Pitfalls:**
*   **Precision:** High-frequency components are sensitive to noise and aliasing. [G] Ensure that the frequency transform is implemented with sufficient numerical stability.
*   **Boundaries:** The choice of frequency cutoff (what constitutes "high" frequency) is critical. [G] A cutoff that is too aggressive may remove valid detail; one that is too lenient may fail to regularize effectively.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_frequency_regularization_loss(rendered_image, ground_truth_image, high_freq_cutoff=0.5):
    """
    Computes the frequency regularization loss.
    
    Args:
        rendered_image (Tensor): Rendered image of shape (H, W, C).
        ground_truth_image (Tensor): Ground truth image of shape (H, W, C).
        high_freq_cutoff (float): Normalized frequency threshold for high-frequency components.
        
    Returns:
        Tensor: Scalar loss value.
    """
    # Apply 2D Fourier Transform to both images
    # Note: In practice, FFT is often applied per channel or on luminance
    fft_rendered = torch.fft.fft2(rendered_image)
    fft_gt = torch.fft.fft2(ground_truth_image)
    
    # Create a high-pass filter mask
    H, W = rendered_image.shape[0], rendered_image.shape[1]
    u, v = torch.meshgrid(torch.arange(H), torch.arange(W), indexing='ij')
    # Center the frequency components
    u_centered = u - H // 2
    v_centered = v - W // 2
    # Compute normalized distance from center
    freq_dist = torch.sqrt(u_centered.float()**2 + v_centered.float()**2) / (H // 2)
    
    # Mask for high frequencies (distance > cutoff)
    high_freq_mask = freq_dist > high_freq_cutoff
    
    # Apply mask to FFTs
    fft_rendered_high = fft_rendered * high_freq_mask.float()
    fft_gt_high = fft_gt * high_freq_mask.float()
    
    # Compute L1 loss in frequency domain
    loss = torch.mean(torch.abs(fft_rendered_high - fft_gt_high))
    
    return loss
```

## Subtopics

### DWTGS
DWTGS is a variant that rethinks frequency regularization for sparse-view 3D Gaussian Splatting [S2, unofficial]. It applies a Discrete Wavelet Transform (DWT) on the rendered image and penalizes the loss in the wavelet domain to encourage sharper edges [S2, unofficial]. Unlike the base method which may use Fourier transforms, DWTGS leverages the multi-resolution properties of wavelets to better capture local high-frequency details and edges, which are often under-represented in sparse-view settings [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the frequency transform. FFT-based approaches are $O(N \log N)$ where $N$ is the number of pixels. Wavelet transforms are also efficient, typically $O(N)$.
*   **Memory Complexity:** Requires storing the frequency domain representation of the images, which is $O(N)$.
*   **Measured Performance:** Not given in the sources. [S1, unofficial][S2, unofficial]
*   **Scaling:** Scales linearly with image resolution for FFT/Wavelet operations.
*   **Strengths:** Explicitly targets high-frequency detail, which is often lost in standard 3DGS training. Effective for improving texture sharpness.
*   **Limitations:** May introduce artifacts if the frequency cutoff is not chosen carefully. Computationally more expensive than spatial-domain losses.
*   **When to use:** When the scene contains fine textures or high-frequency details that are not captured by standard densification.
*   **When not to use:** When computational resources are extremely limited, or when the scene is predominantly low-frequency.

## Background
Frequency regularization was introduced as part of the FreGS method to address the limitations of standard 3D Gaussian Splatting in capturing high-frequency regions [S1, unofficial]. It builds on the observation that Gaussian densification based on position and opacity gradients may not sufficiently explore high-frequency areas of the image [S1, unofficial]. The method has been adapted in variants like DWTGS for sparse-view scenarios [S2, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
