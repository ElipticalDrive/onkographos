# Smoothed 2D Supervision

Also called: Progressive Gaussian Blurring

## Overview
Smoothed 2D Supervision stabilizes the joint optimization of camera poses and 3D radiance fields by blurring 2D training images with scheduled 2D Gaussian kernels [S1, unofficial]. It produces smoothed image gradients to prevent noisy gradient updates and restricts the high-frequency spectrum of the learned scene [S1, unofficial]. Unlike [Frequency Regularization](6130-frequency-regularization.md) which operates in the frequency domain, this method applies spatial-domain smoothing directly to the target images before loss computation.

## How it works
**Inputs and outputs:**
*   Input: 2D training image $I_{target}$, current iteration $t$, kernel schedule parameters.
*   Output: Blurred image $I_{smooth}$ used for computing the $\mathcal{L}_{rgb}$ term.

**Data structures:**
*   Kernel parameters: Standard deviation $\sigma(t)$ and kernel size $K(t)$.
*   The kernel is a 2D Gaussian defined by:
    $$ G(x, y) = \frac{1}{2\pi\sigma^2} e^{-\frac{x^2 + y^2}{2\sigma^2}} $$

**Steps:**
1.  **Initialize Schedule:** Start with a square kernel of size $9 \times 9$ and $\sigma = 2.4$ [S2].
2.  **Iterative Blurring:** For each training iteration $t$:
    a. Determine the current kernel size $K(t)$ and standard deviation $\sigma(t)$.
    b. Apply 2D Gaussian convolution to $I_{target}$ to produce $I_{smooth}$ [S1, unofficial].
    c. Compute the reconstruction loss $\mathcal{L}_{rgb}$ between the rendered image and $I_{smooth}$ [S1, unofficial].
3.  **Schedule Decay:** Reduce the blur intensity every $k_b = 100$ iterations [S2].
    a. The kernel size and $\sigma$ are gradually reduced.
    b. The schedule typically reduces $\sigma$ exponentially to 0 at 10,000 iterations and remains 0 afterward [S1, unofficial].
    c. Specific implementation in [S2] reduces the kernel size before densifying Gaussians to introduce extra signal on the gradient of erroneous Gaussians as the noise level reduces [S2].
4.  **Downsampling Adjustment:** If image downsampling is active, lower the kernel size by the scale of downsampling at the current stage, as the initial hyperparameter choice is based on original image resolutions [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_smoothed_target(target_image, iteration, total_iterations, k_b=100):
    # Initial parameters from [S2]
    initial_sigma = 2.4
    initial_kernel_size = 9
    
    # Determine if we are in the decay phase
    # [S1, unofficial] mentions exponential reduction to 0 at 10k iterations
    # [S2] mentions reduction every k_b iterations until 19500
    
    if iteration < 10000:
        # Exponential decay model for sigma
        # Note: Exact formula for exponential decay not provided in sources [S1, unofficial]
        # We assume a standard decay function or step-wise reduction
        sigma = initial_sigma * (1 - iteration / 10000) 
        # [S1, unofficial] states it is exponentially reduced, so:
        sigma = initial_sigma * exp(-iteration / 10000)
        
        # Kernel size is typically related to sigma (e.g., 2*sigma + 1 or fixed steps)
        # [S2] uses a fixed 9x9 initially and reduces it
        kernel_size = initial_kernel_size
        # Reduce kernel size based on iteration steps
        steps = iteration // k_b
        kernel_size = max(1, initial_kernel_size - steps)
    else:
        sigma = 0
        kernel_size = 1 # No blur

    # Apply Gaussian blur
    if sigma > 0:
        blurred_image = gaussian_blur(target_image, sigma=sigma, kernel_size=kernel_size)
    else:
        blurred_image = target_image
        
    return blurred_image
```

**Edge cases and pitfalls:**
*   **Over-smoothing:** If the kernel is too large for too long, high-frequency details are lost, leading to blurry reconstructions [S1, unofficial].
*   **Noisy Gradients:** Without smoothing, the gradient of the loss with respect to camera pose $\frac{d\mathcal{L}}{du}$ is weaker and more easily influenced by noise, especially when the modulation magnitude $H(u, k)$ decreases as $k \to 0$ [S1, unofficial].
*   **Downsampling Mismatch:** If the image is downsampled, the kernel size must be scaled down accordingly; otherwise, the effective blur radius in world space is incorrect [S2].

## Subtopics

### [Randomly Scaled Kernel Parameter](7875-randomly-scaled-kernel-parameter.md)
Mitigates weak and noisy gradient problems by randomly scaling kernel sizes, allowing combinations of different scales to guide joint optimization.

## Operating characteristics
*   **Time Complexity:** Adds a convolution operation per iteration. Gaussian blur is $O(N \cdot K^2)$ where $N$ is pixel count and $K$ is kernel size. Since $K$ is small (max 9x9), the overhead is low.
*   **Memory Complexity:** Requires storing the blurred image temporarily. $O(N)$.
*   **Performance:** Improves robustness in camera pose estimation and prevents divergence in joint optimization [S1, unofficial]. [S2] reports that using this trick (along with others) improves PSNR on Deep Blending dataset (e.g., Trick-GS achieves 29.46 PSNR vs baseline 24.62 PSNR in some configurations, though direct isolation is hard due to combined tricks) [S2].
*   **Strengths:** Stabilizes training, reduces noise in gradients, restricts high-frequency artifacts.
*   **Limitations:** Requires careful scheduling of kernel parameters; improper scheduling can lead to slow convergence or loss of detail.

## Background
Originated in the context of improving robustness for joint optimization of camera poses and decomposed low-rank tensorial radiance fields [S1, unofficial]. Inspired by spectral analysis showing that blurring training images helps restrict the spectrum of the learned 3D scene and stabilizes camera alignment [S1, unofficial]. Also adopted in Trick-GS for efficient Gaussian Splatting [S2].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
[S2] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
