# Smoothed 2D Supervision

Also called: Adaptive Gaussian blur strategy, Progressive Gaussian Blurring

## Overview
Smoothed 2D Supervision is a regularization technique that enhances the robustness of joint optimization (camera poses and radiance fields) by blurring 2D training images with a scheduled set of 2D Gaussian kernels [S1][S2, unofficial][S3]. It stabilizes camera alignment by producing smoothed image gradients and restricts the spectrum of the learned 3D scene to prevent high-frequency noise [S2, unofficial]. It sets itself apart from siblings like [Frequency Regularization](6130-frequency-regularization.md) and [Cross-scale geometric adaptation](10796-cross-scale-geometric-adaptation.md) by operating directly on the 2D image space supervision signal rather than modifying the 3D representation's spectral properties or loss weights internally [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** Raw 2D training image $I_{raw}$, current iteration $t$, kernel schedule parameters.
*   **Output:** Smoothed supervision image $I_{smooth}$ used in the rendering loss $\mathcal{L}_{rgb}$.

**Data Structures:**
*   **Kernel Schedule:** A sequence of Gaussian kernel parameters $(\sigma_t, k_t)$ defined over iterations $t$.
    *   $\sigma_t$: Standard deviation of the Gaussian kernel.
    *   $k_t$: Kernel size (width/height), typically odd integer.

**Numbered Steps:**
1.  **Initialize Schedule:** Define the initial kernel size and standard deviation. For example, start with a square kernel of size $9 \times 9$ and $\sigma = 2.4$ [S4].
2.  **Iterative Blurring:** For each training iteration $t$:
    a. Determine the current kernel parameters $(\sigma_t, k_t)$ based on the schedule.
    b. Apply a 2D Gaussian blur to the raw training image $I_{raw}$ using the kernel defined by $(\sigma_t, k_t)$ to produce $I_{smooth}$ [S2, unofficial][S4].
    c. Compute the rendering loss $\mathcal{L}_{rgb}$ between the rendered image and $I_{smooth}$ [S2, unofficial].
3.  **Schedule Decay:** Reduce the blur intensity over time.
    *   **Linear/Progressive Decay:** Gradually reduce $\sigma$ and $k$ until they reach 0 or a minimum value [S2, unofficial][S3].
    *   **Specific Schedule (Trick-GS):** Reduce the kernel size every $k_b = 100$ iterations until iteration 19,500 [S4].
    *   **Specific Schedule (AR-NeRF):** Exponentially reduce $\sigma$ to 0 at 10,000 iterations [S2, unofficial].
4.  **Downsampling Adjustment:** If downsampling is activated, lower the kernel size by the scale of downsampling at the current stage, as initial hyperparameters are based on original image resolutions [S4].
5.  **Pre-Densification Reduction:** Lower the blurring kernel size before densifying Gaussians to introduce extra signal on the gradient of erroneous Gaussians as the noise level reduces [S4].

**Formulas:**
The smoothed image $I_{smooth}(x, y)$ is computed as:
$$ I_{smooth}(x, y) = \sum_{i=-k/2}^{k/2} \sum_{j=-k/2}^{k/2} I_{raw}(x-i, y-j) \cdot G(i, j; \sigma) $$
where $G(i, j; \sigma)$ is the 2D Gaussian kernel:
$$ G(i, j; \sigma) = \frac{1}{2\pi\sigma^2} e^{-\frac{i^2 + j^2}{2\sigma^2}} $$

**Parameters:**
*   Initial $\sigma$: 2.4 [S4].
*   Initial Kernel Size: $9 \times 9$ [S4].
*   Decay Step: Every 100 iterations [S4].
*   End of Decay: Iteration 19,500 [S4] or 10,000 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Weak Gradients:** Overly large kernels can lead to weak gradients $\frac{d\tilde{L}}{du}$ that are easily influenced by noise, as the magnitude of the modulated frequency response $H(u, k)$ decreases linearly as $k \to 0$ [S2, unofficial].
*   **Frequency Mismatch:** Handcrafted schedules (linear or adaptive Gaussian blur) cannot fully align the frequency relationship between positional encodings (PE) and pixel supervision, potentially leading to suboptimal results compared to adaptive methods [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_kernel_params(iteration, initial_sigma=2.4, initial_size=9, decay_step=100, end_iteration=19500):
    """
    Returns (sigma, kernel_size) for the given iteration.
    Implements the schedule from Trick-GS [S4].
    """
    if iteration >= end_iteration:
        return 0.0, 1  # No blur at end
    
    # Linear reduction of sigma and size
    progress = iteration / end_iteration
    sigma = initial_sigma * (1 - progress)
    
    # Kernel size reduction
    # Assuming linear reduction from initial_size to 1
    kernel_size = max(1, int(initial_size * (1 - progress)))
    if kernel_size % 2 == 0:
        kernel_size += 1
        
    return sigma, kernel_size

def apply_gaussian_blur(image, sigma, kernel_size):
    """
    Applies 2D Gaussian blur to the image.
    """
    if sigma == 0:
        return image
    
    # Create Gaussian kernel
    # Center of kernel
    center = kernel_size // 2
    kernel = np.zeros((kernel_size, kernel_size))
    
    for i in range(kernel_size):
        for j in range(kernel_size):
            x = i - center
            y = j - center
            kernel[i, j] = np.exp(-(x**2 + y**2) / (2 * sigma**2))
            
    # Normalize kernel
    kernel_sum = np.sum(kernel)
    if kernel_sum > 0:
        kernel /= kernel_sum
        
    # Apply convolution
    # Padding to handle boundaries
    padded_image = np.pad(image, ((center, center), (center, center), (0, 0)), mode='reflect')
    blurred_image = np.zeros_like(image)
    
    for i in range(image.shape[0]):
        for j in range(image.shape[1]):
            region = padded_image[i:i+kernel_size, j:j+kernel_size, :]
            blurred_image[i, j, :] = np.sum(region * kernel[:, :, np.newaxis], axis=(0, 1))
            
    return blurred_image

def smoothed_2d_supervision_step(rendered_image, raw_image, iteration, downsampling_scale=1.0):
    """
    Computes the smoothed supervision signal for the current iteration.
    """
    # Get kernel parameters
    sigma, kernel_size = get_kernel_params(iteration)
    
    # Adjust kernel size for downsampling if necessary
    if downsampling_scale > 1.0:
        kernel_size = max(1, kernel_size // downsampling_scale)
        if kernel_size % 2 == 0:
            kernel_size += 1
            
    # Apply blur
    smoothed_image = apply_gaussian_blur(raw_image, sigma, kernel_size)
    
    return smoothed_image
```

## Subtopics

### [Randomly Scaled Kernel Parameter](7875-randomly-scaled-kernel-parameter.md)
Randomly scales the kernel size during training to mitigate weak and noisy gradient signals, allowing combinations of different-sized kernels to guide optimization. It sets itself apart from uniform weighting by dynamically adjusting the spectral influence.

### [Multiscale Pre-convolved Representation](10911-multiscale-pre-convolved-representation.md)
Enforces consistency with pre-computed, roughness-adapted background reflections to stabilize material-illumination decomposition. It sets itself apart by explicitly modeling the relationship between rough refractive surfaces and convolved background.

## Operating characteristics
*   **Time Complexity:** Adds $O(k^2)$ per pixel per iteration for blurring, where $k$ is the kernel size. Since $k$ decreases over time, the cost is highest in early iterations.
*   **Memory Complexity:** Requires storing the smoothed image or recomputing it on the fly. No additional large data structures.
*   **Measured Performance:** Achieves 19.72 PSNR on DTU 3-view, improving over the Baseline (19.54) [S3].
*   **Strengths:** Stabilizes camera alignment, reduces training iterations (e.g., 50k vs 200k in some methods) [S1], and restricts high-frequency noise [S2, unofficial].
*   **Limitations:** Handcrafted schedules may not fully align frequency relationships between PE and supervision [S3]. Cannot adapt to scene-specific frequency content as well as adaptive methods.
*   **When to Use:** Crucial for successful pose refinement in joint optimization scenarios [S1]. Useful when camera alignment is unstable or high-frequency artifacts are present.
*   **When Not to Use:** When adaptive frequency alignment is available and superior performance is required [S3].

## Background
Smoothed 2D Supervision was proposed to improve the robustness of joint optimization of camera poses and radiance fields [S1][S2, unofficial]. It is based on the observation that blurring 2D training images produces smoothed gradients and restricts the spectrum of the learned scene [S2, unofficial]. It is often used in conjunction with other techniques like Randomly Scaled Kernel Parameter and Edge-Guided Loss Mask [S1]. The method has been applied in various contexts, including NeRF and Gaussian Splatting training [S2, unofficial][S3, S4].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.1609/aaai.v38i2.27859, fetched 2026-10-26
[S2] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
[S3] Few-shot NeRF by Adaptive Rendering Loss Regularization, https://core.ac.uk/download/708671746.pdf, fetched 2026-10-30
[S4] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
