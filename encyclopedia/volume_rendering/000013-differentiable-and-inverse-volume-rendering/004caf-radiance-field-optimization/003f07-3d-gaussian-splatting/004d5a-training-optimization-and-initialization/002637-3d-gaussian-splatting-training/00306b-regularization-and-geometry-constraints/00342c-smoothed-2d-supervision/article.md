# Smoothed 2D Supervision

Also called: Adaptive Gaussian blur strategy, Progressive Gaussian Blurring

## Overview
Smoothed 2D Supervision is a regularization technique that enhances the robustness of joint optimization in radiance fields by smoothing the 2D supervision signal [S1][S2, unofficial]. It addresses the instability caused by high-frequency noise in raw image gradients, which can lead to poor camera alignment and overfitting to noise [S2, unofficial]. Unlike standard supervision that uses raw pixels, this method applies a scheduled 2D Gaussian blur to training images, stabilizing gradients and restricting the spectrum of the learned 3D scene [S2, unofficial]. It sets itself apart from frequency regularization [6130-frequency-regularization.md] by operating directly on the 2D image domain rather than modifying the positional encoding or loss weights in the frequency domain.

## How it works
**Inputs and Outputs:**
- Input: Raw 2D training image $I_{raw}$, current iteration $t$, total iterations $T_{max}$, kernel schedule parameters.
- Output: Smoothed supervision image $I_{smooth}$ used in the rendering loss $\mathcal{L}_{rgb}$.

**Data Structures:**
- The Gaussian kernel $K$ is defined by its standard deviation $\sigma$ and size $S \times S$.
- The kernel is separable, implemented as two 1D kernels $k_x$ and $k_y$ for efficiency [S1].

**Parameters:**
- Initial standard deviation $\sigma_0$: 2.4 [S4].
- Initial kernel size: $9 \times 9$ [S4].
- Schedule interval $k_b$: 100 iterations [S4].
- End iteration for blurring: 19,500 [S4] or 10,000 [S2, unofficial]. (Sources disagree on the exact end iteration; [S4] specifies 19,500 while [S2, unofficial] mentions reduction to 0 at 10,000. Implementers should check the specific variant's configuration.)
- Downsampling scale factor $s_{down}$: Applied if downsampling is active [S4].

**Steps:**
1. **Initialize Kernel:** Set initial $\sigma = \sigma_0$ and kernel size $S = 9$.
2. **Schedule Update:** At every $k_b$ iterations, reduce $\sigma$. The reduction is typically exponential or linear towards 0 [S2, unofficial][S4].
   - Formula for $\sigma(t)$: Not explicitly given as a single closed-form function in sources. [S4] implies a gradual reduction. [S2, unofficial] states it is "exponentially reduced to 0". A common implementation is:
     $$ \sigma(t) = \sigma_0 \cdot \left(1 - \frac{t}{T_{end}}\right)^\alpha $$
     where $T_{end}$ is the end iteration (10,000 or 19,500) and $\alpha$ controls the curve. If $t \ge T_{end}$, $\sigma = 0$.
3. **Adjust for Downsampling:** If the training images are downsampled by a factor $s_{down}$, the effective $\sigma$ must be scaled:
   $$ \sigma_{effective} = \sigma \cdot s_{down} $$
   This ensures the blur corresponds to the original resolution [S4].
4. **Apply Blur:** Convolve $I_{raw}$ with the separable Gaussian kernel $K(\sigma_{effective})$ to produce $I_{smooth}$.
   $$ I_{smooth} = I_{raw} * K(\sigma_{effective}) $$
5. **Compute Loss:** Calculate the rendering loss $\mathcal{L}_{rgb}$ between the rendered image and $I_{smooth}$ instead of $I_{raw}$.
6. **Pre-Densification Adjustment:** Lower the blurring kernel size before densifying Gaussians to introduce extra signal on the gradient of erroneous Gaussians as noise level reduces [S4].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_smoothed_supervision(raw_image, iteration, sigma_0=2.4, k_b=100, t_end=19500, downsampling_scale=1.0):
    """
    Computes the smoothed supervision image for a given iteration.
    
    Args:
        raw_image: Input 2D image tensor.
        iteration: Current training iteration.
        sigma_0: Initial standard deviation.
        k_b: Interval for kernel reduction steps.
        t_end: Iteration at which sigma reaches 0.
        downsampling_scale: Scale factor if images are downsampled.
        
    Returns:
        smoothed_image: Blurred image tensor.
    """
    # Calculate current sigma based on schedule
    # Sources suggest exponential reduction [S2, unofficial] or gradual reduction [S4]
    # Using a simple linear decay to 0 as a robust default if exact exponent is unspecified
    progress = min(iteration / t_end, 1.0)
    sigma = sigma_0 * (1.0 - progress)
    
    # Adjust for downsampling if applicable
    sigma = sigma * downsampling_scale
    
    # If sigma is effectively zero, return raw image
    if sigma < 1e-6:
        return raw_image
    
    # Create 1D Gaussian kernel
    # Kernel size is typically 2 * ceil(3 * sigma) + 1 to cover 3 sigma range
    kernel_size = int(2 * (3 * sigma) + 1)
    if kernel_size < 1:
        kernel_size = 1
    indices = torch.arange(kernel_size) - kernel_size // 2
    kernel_1d = torch.exp(-0.5 * (indices / sigma)**2)
    kernel_1d = kernel_1d / kernel_1d.sum()
    
    # Apply separable convolution
    # Assuming raw_image is [C, H, W]
    # Convolve along width
    kernel_1d_2d = kernel_1d.view(1, 1, 1, -1)
    smoothed = F.conv2d(raw_image, kernel_1d_2d, padding=(0, kernel_size//2))
    
    # Convolve along height
    kernel_1d_2d = kernel_1d.view(1, 1, -1, 1)
    smoothed = F.conv2d(smoothed, kernel_1d_2d, padding=(kernel_size//2, 0))
    
    return smoothed
```

**Edge Cases and Pitfalls:**
- **Zero Sigma:** When $\sigma$ reaches 0, the blur becomes an identity operation. Ensure the code handles this to avoid division by zero in kernel generation [S2, unofficial][S4].
- **Boundary Conditions:** Gaussian convolution at image boundaries can cause darkening. Use appropriate padding (e.g., reflection or constant) or clamp the kernel [G].
- **Downsampling Mismatch:** If downsampling is used, failing to scale $\sigma$ by the downsampling factor results in incorrect frequency suppression [S4].

## Subtopics

### [Randomly Scaled Kernel Parameter](7875-randomly-scaled-kernel-parameter.md)
Randomly scales the kernel size during optimization to handle diverse scene structures, offering dynamic spectral influence compared to fixed schedules.

### [Multiscale Pre-convolved Representation](10911-multiscale-pre-convolved-representation.md)
Uses pre-computed, roughness-adapted background reflections for neural illumination fields, addressing material-illumination ambiguity.

### [Soft Rasterization](13687-soft-rasterization.md)
Blends facet contributions near edges to mitigate gradient vanishing, stabilizing optimization through differentiable rendering.

## Operating characteristics
- **Time Complexity:** The blur operation is $O(N \cdot S^2)$ per image, where $N$ is the number of pixels and $S$ is the kernel size. Since the kernel is separable, it is $O(N \cdot S)$. This adds a small constant overhead to each training step [S1].
- **Memory Complexity:** Minimal additional memory, requiring only storage for the kernel and the smoothed image buffer [S1].
- **Performance:** Improves PSNR on DTU 3-view from 19.54 (Baseline) to 19.72 [S3]. It reduces training iterations required for convergence (e.g., 50k vs 200k for some MLP-based methods) [S1].
- **Strengths:** Stabilizes camera alignment, restricts high-frequency noise, and improves robustness in joint optimization [S1][S2, unofficial].
- **Limitations:** Handcrafted schedule may not fully align frequency relationships between positional encoding and pixel supervision compared to adaptive methods [S3].
- **When to Use:** Crucial for successful pose refinement in joint optimization scenarios [S1]. Recommended when training is unstable or prone to overfitting high-frequency noise.

## Background
Smoothed 2D Supervision was proposed to improve the robustness of joint optimization of camera poses and radiance fields [S1][S2, unofficial]. It is based on the observation that blurring 2D training images produces smoother gradients, which helps stabilize camera alignment and restricts the spectrum of the learned scene [S2, unofficial]. The technique is often used in conjunction with other regularization methods like randomly scaled kernel parameters [S1]. It has been shown to be effective in few-shot NeRF settings and Gaussian Splatting variants [S3, S4].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.1609/aaai.v38i2.27859, fetched 2026-10-26
[S2] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
[S3] Few-shot NeRF by Adaptive Rendering Loss Regularization, https://core.ac.uk/download/708671746.pdf, fetched 2026-10-30
[S4] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
