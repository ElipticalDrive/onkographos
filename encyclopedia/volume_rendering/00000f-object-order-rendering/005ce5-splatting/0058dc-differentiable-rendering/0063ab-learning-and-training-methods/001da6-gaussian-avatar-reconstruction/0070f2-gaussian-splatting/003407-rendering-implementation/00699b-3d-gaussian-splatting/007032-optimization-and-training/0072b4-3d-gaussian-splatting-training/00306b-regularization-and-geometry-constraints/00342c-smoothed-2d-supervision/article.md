# Smoothed 2D Supervision

Also called: Adaptive Gaussian blur strategy, Progressive Gaussian Blurring

## Overview
Smoothed 2D Supervision is a regularization technique that enhances the robustness of joint optimization in differentiable rendering by smoothing the 2D supervision signal [S1][S2, unofficial]. It addresses the instability in camera alignment and the overfitting to high-frequency noise that occurs when optimizing 3D scene representations against raw, high-resolution training images [S2, unofficial]. Unlike standard photometric loss which penalizes all frequency components equally, this method applies a scheduled 2D Gaussian blur to the target images, restricting the spectrum of the learned 3D scene and producing smoother image gradients [S2, unofficial][S3]. It sets itself apart from frequency regularization methods by operating directly on the 2D image domain supervision signal rather than modifying the neural network's frequency encoding or loss weights [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** Raw 2D training images $I_{raw}$, camera parameters, and a blur schedule defined by kernel size $k$ and standard deviation $\sigma$.
*   **Output:** Smoothed target images $I_{smooth}$ used for computing the photometric loss $\mathcal{L}_{photo}$.

**Data Structures:**
*   **Blur Kernel:** A 2D Gaussian kernel $K(x, y)$ with standard deviation $\sigma$ and support size $k \times k$.
*   **Schedule:** A sequence of $(\sigma_t, k_t)$ pairs indexed by training iteration $t$.

**Steps:**
1.  **Initialize Schedule:** Define the initial blur parameters. For example, start with a square kernel of size $9 \times 9$ and $\sigma = 2.4$ [S4].
2.  **Iterative Blurring:** For each training iteration $t$:
    a. Determine the current kernel parameters $(\sigma_t, k_t)$ based on the schedule.
    b. Apply 2D Gaussian convolution to the raw training image $I_{raw}$ to produce the smoothed target $I_{smooth} = I_{raw} * K_t$ [S2, unofficial].
    c. Render the scene using current Gaussian parameters to get $I_{render}$.
    d. Compute loss $\mathcal{L}_{photo} = \| I_{render} - I_{smooth} \|_1$ (or SSIM) [S2, unofficial].
3.  **Schedule Decay:** Gradually reduce $\sigma$ and $k$ over iterations to transition from low-frequency to high-frequency supervision [S2, unofficial][S3].
    *   Example Schedule: Reduce blur every $k_b = 100$ iterations until iteration 19,500 [S4].
    *   The kernel size is lowered before densifying Gaussians to introduce extra signal on the gradient of erroneous Gaussians as the noise level reduces [S4].
4.  **Downsampling Adjustment:** If image downsampling is used, lower the kernel size by the scale of downsampling at the current stage, as initial hyperparameters are based on original image resolutions [S4].

**Formulas:**
The 2D Gaussian kernel $K(x, y)$ is defined as:
$$ K(x, y) = \frac{1}{2\pi\sigma^2} e^{-\frac{x^2 + y^2}{2\sigma^2}} $$
where $x, y$ are coordinates relative to the kernel center, and $\sigma$ is the standard deviation [S2, unofficial].

The smoothed supervision image is:
$$ I_{smooth}(u, v) = \sum_{i, j} I_{raw}(u-i, v-j) K(i, j) $$

**Parameters:**
*   Initial $\sigma$: 2.4 [S4].
*   Initial Kernel Size: $9 \times 9$ [S4].
*   Decay Interval: Every 100 iterations [S4].
*   End Iteration for Blur: 19,500 [S4].
*   $\sigma$ reaches 0 at 10,000 iterations in some implementations [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Over-smoothing:** If $\sigma$ is too large for too long, high-frequency details are lost, leading to blurry reconstructions [S2, unofficial].
*   **Under-smoothing:** If $\sigma$ decays too quickly, the optimization becomes unstable, especially during camera pose refinement [S1].
*   **Resolution Mismatch:** When using downscaled images, the kernel size must be scaled down proportionally to maintain the effective blur radius in world space [S4].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_smoothed_supervision(raw_image, iteration, schedule):
    """
    Computes the smoothed target image for a given iteration.
    
    Args:
        raw_image: 2D numpy array of shape (H, W, C)
        iteration: Current training iteration number
        schedule: List of tuples (start_iter, end_iter, sigma, kernel_size)
        
    Returns:
        smoothed_image: 2D numpy array of shape (H, W, C)
    """
    # Find the active schedule parameters for the current iteration
    sigma = 0.0
    kernel_size = 1
    
    for start_iter, end_iter, s, k in schedule:
        if start_iter <= iteration < end_iter:
            sigma = s
            kernel_size = k
            break
    else:
        # Default to no blur if iteration is beyond schedule
        sigma = 0.0
        kernel_size = 1

    if sigma == 0.0:
        return raw_image

    # Create 2D Gaussian kernel
    # Kernel size is odd to have a clear center
    if kernel_size % 2 == 0:
        kernel_size += 1
        
    half_k = kernel_size // 2
    x = np.arange(-half_k, half_k + 1)
    y = np.arange(-half_k, half_k + 1)
    xx, yy = np.meshgrid(x, y)
    
    # 2D Gaussian formula
    kernel = (1 / (2 * np.pi * sigma**2)) * np.exp(-(xx**2 + yy**2) / (2 * sigma**2))
    
    # Normalize kernel to sum to 1
    kernel = kernel / np.sum(kernel)
    
    # Apply convolution to each channel
    smoothed_image = np.zeros_like(raw_image)
    for c in range(raw_image.shape[2]):
        smoothed_image[:, :, c] = convolve2d(raw_image[:, :, c], kernel, mode='same')
        
    return smoothed_image

# Example Schedule Definition
# (start_iter, end_iter, sigma, kernel_size)
# Starts with heavy blur, reduces over time
schedule = [
    (0, 10000, 2.4, 9),   # Heavy blur for first 10k iters
    (10000, 19500, 0.0, 1), # No blur after 10k iters (sigma=0)
    (19500, 200000, 0.0, 1) # No blur for rest
]
```

## Subtopics

### [Randomly Scaled Kernel Parameter](7875-randomly-scaled-kernel-parameter.md)
Randomly Scaled Kernel Parameter is a regularization technique used during joint optimization to mitigate weak and noisy gradient signals. It addresses the insufficiency of using a single-size kernel for diverse real-world scene structures, where a fixed kernel may be overly aggressive in some contexts and overly gentle in others. By randomly scaling the kernel size, it allows combinations of different-sized kernels to guide the optimization, resulting in a more robust training schedule.

### [Multiscale Pre-convolved Representation](10911-multiscale-pre-convolved-representation.md)
A regularization technique for neural illumination fields that stabilizes the decomposition of object materials from environmental illumination by enforcing consistency with pre-computed, roughness-adapted background reflections. It addresses the ambiguity where diffuse object color is incorrectly absorbed into the estimated environment map, a problem inherent to high-dimensional neural illumination representations. Unlike methods that ignore the background or treat it as static, this method explicitly models the relationship between rough refractive surfaces and the convolved background, allowing for realistic specular reflections even when roughness exceeds the kernel scale. The technique builds on Smoothed 2D Supervision by applying discrete Gaussian blur kernels to the background to stabilize convergence.

### [Soft Rasterization](13687-soft-rasterization.md)
Soft rasterization is a differentiable rendering technique that mitigates the discontinuities inherent in standard rasterization by blending facet contributions for pixels near primitive edges. It solves the problem of sparse and vanishing gradients caused by the non-uniformity of bilinear interpolation operands, which typically have similar values except at object boundaries. Unlike standard hard rasterization, it yields slightly distorted images but allows backpropagating more useful gradients, thereby stabilizing optimization and preventing gradient vanishing at edges.

### Platonic GAN (pGAN)
Platonic GAN (pGAN) is a learning-based method for 2D supervision per class of reconstructed objects, used here to pre-train a network for per-frame supervision of rising smoke [S5]. It provides a form of structural prior that can improve depth artifacts in single-view evaluations, though it may lack temporal coherence if not combined with temporal constraints [S5].

## Operating characteristics
*   **Time Complexity:** Adds a convolution step per training iteration. The cost is proportional to $O(H \cdot W \cdot k^2)$ per image, where $H, W$ are image dimensions and $k$ is kernel size. Since $k$ is small (e.g., 9x9), this is negligible compared to rendering [S4].
*   **Memory Complexity:** Requires storing the smoothed images or recomputing them on the fly. No additional large data structures [S2, unofficial].
*   **Performance:** Improves PSNR on DTU 3-view from 19.54 (Baseline) to 19.72 [S3]. Reduces training iterations required for convergence (e.g., 50k vs 200k in some MLP-based methods) [S1].
*   **Strengths:** Stabilizes camera alignment, reduces noise in gradients, and prevents overfitting to high-frequency artifacts [S2, unofficial]. Crucial for successful pose refinement [S1].
*   **Limitations:** Handcrafted schedule may not fully align frequency relationships between positional encoding and pixel supervision compared to adaptive methods [S3]. Requires careful tuning of the decay schedule [S4].
*   **When to Use:** When optimizing camera poses jointly with scene representation, or when training is unstable due to high-frequency noise [S1][S2, unofficial].
*   **When Not to Use:** When high-frequency detail preservation is critical and the schedule cannot be tuned to recover it, or when using adaptive frequency-aware methods that outperform handcrafted blurs [S3].

## Background
Smoothed 2D Supervision was introduced as part of techniques to improve the robustness of joint optimization for camera pose and radiance fields [S1][S2, unofficial]. It was inspired by spectral analysis showing that blurring 2D training images helps restrict the spectrum of the learned 3D scene [S2, unofficial]. The method is often compared to adaptive rendering loss regularization, which dynamically adjusts loss weights based on frequency, whereas Smoothed 2D Supervision uses a fixed, scheduled blur [S3]. It has been applied in various contexts including Gaussian Splatting (Trick-GS) and NeRF-based methods [S3, S4].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.1609/aaai.v38i2.27859, fetched 2026-10-26
[S2] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
[S3] Few-shot NeRF by Adaptive Rendering Loss Regularization, https://core.ac.uk/download/708671746.pdf, fetched 2026-10-30
[S4] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S5] Global Transport for Fluid Reconstruction with Learned Self-Supervision, https://doi.org/10.1109/cvpr46437.2021.00168, fetched 2026-10-30
