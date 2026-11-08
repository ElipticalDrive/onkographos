# Frequency Regularization

Also called: FreGS

## Overview
Frequency regularization is a training-time technique that guides the optimization of 3D Gaussian Splatting parameters by penalizing discrepancies in specific frequency bands of the rendered image [S1, unofficial]. It solves the problem where standard splatting methods may converge to low-frequency solutions, resulting in blurry details or missing high-frequency structures [S1, unofficial]. Unlike [Depth Regularization](7616-depth-regularization.md) which enforces geometric consistency via depth maps, frequency regularization operates directly on the image space or feature space to enforce sharpness and detail preservation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered image $I_{render}$, ground truth image $I_{gt}$, and their frequency-domain representations.
*   **Outputs:** A scalar loss value $\mathcal{L}_{freq}$ added to the total loss.

**Data Structures:**
*   **Frequency Domain Representation:** Typically obtained via Discrete Fourier Transform (DFT) or Discrete Wavelet Transform (DWT). For standard FreGS, the magnitude spectrum of the image is used.

**Steps:**
1.  **Render:** Compute the rendered image $I_{render}$ from the current Gaussian parameters $\theta$ using the differentiable rasterizer.
2.  **Transform:** Apply a frequency transform to both the rendered image and the ground truth image.
    *   For standard frequency regularization, compute the magnitude spectrum $M(I)$ of an image $I$.
    *   $M(I) = |\mathcal{F}(I)|$, where $\mathcal{F}$ is the Fourier transform.
3.  **Compute Loss:** Calculate the discrepancy between the frequency components of the rendered and ground truth images.
    *   $\mathcal{L}_{freq} = \| M(I_{render}) - M(I_{gt}) \|_1$ or $\| M(I_{render}) - M(I_{gt}) \|_2$.
    *   Some implementations may weight higher frequencies more heavily to prioritize detail [S1, unofficial].
4.  **Backpropagate:** Compute gradients of $\mathcal{L}_{freq}$ with respect to $\theta$ and update the Gaussians.

**Parameters:**
*   **Weight $\lambda_{freq}$:** The scaling factor for the frequency loss term. Typical values are not explicitly given in the sources but are usually small (e.g., $10^{-3}$ to $10^{-2}$) relative to the photometric loss [G].
*   **Frequency Band:** The method may target specific bands (e.g., high-frequency only). Not given in the sources for standard FreGS [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Frequency transforms are sensitive to image normalization. Ensure images are normalized consistently before transformation [G].
*   **Boundaries:** Discrete transforms assume periodicity, which can cause artifacts at image boundaries. Padding or windowing functions may be required [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def frequency_regularization_loss(rendered_image, ground_truth_image, weight=1e-3):
    """
    Computes the frequency regularization loss between rendered and ground truth images.
    
    Args:
        rendered_image: Tensor of shape (H, W, 3), values in [0, 1].
        ground_truth_image: Tensor of shape (H, W, 3), values in [0, 1].
        weight: Scalar weight for the loss term.
        
    Returns:
        Scalar loss value.
    """
    # Convert to float32 for stability
    rendered = rendered_image.float()
    ground_truth = ground_truth_image.float()
    
    # Compute magnitude spectrum using 2D FFT
    # Note: In practice, use torch.fft or similar library functions
    fft_rendered = torch.fft.fft2(rendered)
    fft_gt = torch.fft.fft2(ground_truth)
    
    magnitude_rendered = torch.abs(fft_rendered)
    magnitude_gt = torch.abs(fft_gt)
    
    # Compute L1 loss on magnitude spectra
    # This encourages the rendered image to match the frequency content of the GT
    loss = torch.mean(torch.abs(magnitude_rendered - magnitude_gt))
    
    return weight * loss
```

## Subtopics

### DWTGS
DWTGS is a variant designed for sparse-view 3D Gaussian Splatting [S2, unofficial]. It rethinks frequency regularization by applying a Discrete Wavelet Transform (DWT) on the rendered image and penalizing the loss in the wavelet domain [S2, unofficial]. This approach encourages sharper edges compared to standard Fourier-based methods, which is particularly beneficial when training data is limited [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The addition of FFT or DWT operations increases per-step computation time. FFT is $O(N \log N)$ where $N$ is the number of pixels. This is generally negligible compared to rasterization for typical resolutions [G].
*   **Memory Complexity:** Requires storing frequency domain representations, adding minimal memory overhead [G].
*   **Performance:** Improves detail preservation and reduces blurriness [S1, unofficial].
*   **Strengths:** Effective at recovering high-frequency details that photometric loss alone might miss [S1, unofficial].
*   **Limitations:** May introduce artifacts if the frequency weighting is not tuned correctly [G].
*   **When to use:** When the scene contains fine details or textures that are prone to blurring in standard 3DGS [S1, unofficial].
*   **When not to use:** For scenes with predominantly low-frequency content where detail preservation is not critical [G].

## Background
Frequency regularization was introduced as part of the FreGS method to address the lack of high-frequency details in standard 3D Gaussian Splatting [S1, unofficial]. It builds on the observation that photometric loss alone may not sufficiently constrain the high-frequency components of the rendered image [S1, unofficial]. The method has been adapted in variants like DWTGS for sparse-view scenarios [S2, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
