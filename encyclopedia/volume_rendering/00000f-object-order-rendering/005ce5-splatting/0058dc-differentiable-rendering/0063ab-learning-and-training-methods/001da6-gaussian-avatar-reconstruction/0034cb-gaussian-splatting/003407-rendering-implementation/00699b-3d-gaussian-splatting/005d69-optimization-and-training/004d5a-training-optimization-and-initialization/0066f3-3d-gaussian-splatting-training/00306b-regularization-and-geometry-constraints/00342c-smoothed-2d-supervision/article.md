# Smoothed 2D Supervision

Also called: Adaptive Gaussian blur strategy, Progressive Gaussian Blurring

## Overview
Smoothed 2D Supervision is a regularization technique that enhances the robustness of joint optimization by smoothing the 2D supervision signal through the application of scheduled 2D Gaussian kernels to training images [S1][S2, unofficial]. It solves the problem of unstable camera alignment and noisy gradient signals during the learning of 3D scene representations by restricting the spectrum of the learned scene and producing smoother image gradients [S2, unofficial]. Unlike [Frequency Regularization](6130-frequency-regularization.md) which operates in the frequency domain, or [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md) which modifies geometric covariance, this method operates directly on the 2D pixel supervision space [S2, unofficial]. It is distinct from [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md) as it does not use depth priors but rather blurs the color/intensity targets [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A batch of 2D training images $I_{raw}$ with resolution $W \times H$, and a current Gaussian splatting state (positions, covariances, opacities, colors).
*   **Output:** A smoothed supervision image $I_{smooth}$ used to compute the photometric loss $\mathcal{L}_{photometric}$ against the rendered image $I_{render}$.

**Data Structures:**
*   **Kernel Schedule:** A sequence of Gaussian kernel parameters $(\sigma_i, k_i)$ defined for iteration intervals.
*   **Gaussian Kernel:** A 2D separable Gaussian kernel $K(x, y)$ with standard deviation $\sigma$ and support size $k \times k$.

**Parameters:**
*   $\sigma$: Standard deviation of the Gaussian blur.
*   $k$: Kernel support size (width/height).
*   $k_b$: Iteration interval for reducing blur (e.g., 100 iterations) [S4].
*   $I_{max}$: Maximum iteration for full blur reduction (e.g., 19500) [S4].
*   $s_{down}$: Downsampling scale factor if resolution reduction is active [S4].

**Steps:**
1.  **Initialize Kernel:** Start with a square Gaussian kernel with support size $k_{start} \times k_{start}$ and standard deviation $\sigma_{start}$. Typical values: $k_{start}=9$, $\sigma_{start}=2.4$ [S4].
2.  **Schedule Update:** At every $k_b$ iterations, reduce the blur intensity. This can be done by reducing $\sigma$ or $k$. The schedule continues until iteration $I_{max}$, after which $\sigma$ or $k$ becomes 0 (no blur) [S2, unofficial][S4].
3.  **Apply Downsampling Adjustment:** If the training images are downsampled by a factor $s_{down}$, the kernel size must be scaled down by the same factor at the current stage to maintain equivalent spatial frequency suppression relative to the original resolution [S4].
4.  **Convolve:** Apply the 2D Gaussian blur to the raw training image $I_{raw}$ to produce $I_{smooth}$:
    $$ I_{smooth}(u, v) = (I_{raw} * K_{\sigma, k})(u, v) $$
    where $*$ denotes 2D convolution.
5.  **Compute Loss:** Render the scene to get $I_{render}$ and compute the photometric loss (e.g., L1 or LPIPS) between $I_{smooth}$ and $I_{render}$ [S2, unofficial][S3].
6.  **Gradient Propagation:** The gradients from the loss flow back through the differentiable renderer to update Gaussian parameters. The smoothing ensures these gradients are less noisy, stabilizing camera pose refinement [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_blur_kernel(iteration, k_b=100, k_start=9, sigma_start=2.4, i_max=19500, s_down=1.0):
    """
    Returns the kernel size k and sigma for the current iteration.
    """
    # Calculate progress towards i_max
    progress = min(iteration / i_max, 1.0)
    
    # Linearly reduce sigma from sigma_start to 0
    sigma = sigma_start * (1.0 - progress)
    
    # Reduce kernel size proportionally or based on sigma
    # Note: Source [S4] specifies starting at 9x9 and reducing.
    # We map sigma to a kernel size. A common approximation is k = 2 * ceil(3 * sigma) + 1
    # However, [S4] implies a discrete schedule. We interpolate k.
    k_float = k_start * (1.0 - progress)
    k = max(1, int(round(k_float)))
    
    # Adjust for downsampling if applicable
    k = max(1, int(round(k / s_down)))
    sigma = sigma / s_down
    
    return k, sigma

def apply_smoothed_2d_supervision(I_raw, iteration, s_down=1.0):
    """
    Applies the smoothed 2D supervision to the raw image.
    """
    k, sigma = get_blur_kernel(iteration, s_down=s_down)
    
    if sigma <= 0 or k <= 1:
        return I_raw
    
    # Create 2D Gaussian kernel
    # Using separable convolution for efficiency
    x = np.arange(-k//2, k//2 + 1)
    kernel_1d = np.exp(-0.5 * (x / sigma)**2)
    kernel_1d = kernel_1d / np.sum(kernel_1d)
    
    # Apply 1D convolution along rows
    I_temp = np.zeros_like(I_raw)
    for i in range(I_raw.shape[0]):
        I_temp[i, :] = np.convolve(I_raw[i, :], kernel_1d, mode='same')
        
    # Apply 1D convolution along columns
    I_smooth = np.zeros_like(I_temp)
    for j in range(I_temp.shape[1]):
        I_smooth[:, j] = np.convolve(I_temp[:, j], kernel_1d, mode='same')
        
    return I_smooth
```

**Edge Cases and Pitfalls:**
*   **Boundary Effects:** Gaussian convolution at image boundaries can cause artifacts. Padding (e.g., reflection or replication) should be used [G].
*   **Kernel Size vs. Resolution:** If the kernel size is too large relative to the image resolution, it may oversmooth and remove essential high-frequency details needed for sharp reconstruction [S2, unofficial].
*   **Downsampling Mismatch:** If downsampling is used, failing to scale the kernel size by $s_{down}$ leads to incorrect frequency suppression [S4].

## Subtopics

### [Randomly Scaled Kernel Parameter](7875-randomly-scaled-kernel-parameter.md)
Randomly scales the kernel size during optimization to mitigate weak and noisy gradient signals, allowing combinations of different-sized kernels to guide training.

### [Multiscale Pre-convolved Representation](10911-multiscale-pre-convolved-representation.md)
Enforces consistency with pre-computed, roughness-adapted background reflections to stabilize neural illumination fields.

### [Soft Rasterization](13687-soft-rasterization.md)
Blends facet contributions for pixels near primitive edges to mitigate discontinuities and stabilize optimization.

## Operating characteristics
*   **Time Complexity:** The convolution operation is $O(W \cdot H \cdot k^2)$ per image per iteration. Using separable convolution reduces this to $O(W \cdot H \cdot k)$ [S1].
*   **Memory Complexity:** Requires storing the blurred image $I_{smooth}$, adding $O(W \cdot H)$ memory overhead per view [S2, unofficial].
*   **Performance:** On DTU 3-view, it improves PSNR from 19.54 (Baseline) to 19.72 [S3]. It is experimentally proven crucial for successful pose refinement [S1].
*   **Strengths:** Stabilizes camera alignment, restricts the spectrum of the learned 3D scene, and reduces training iterations required (e.g., 50k vs 200k in some MLP-based methods) [S1][S2, unofficial].
*   **Limitations:** It is a handcrafted strategy and cannot fully align frequency relationships between positional encodings and pixel supervision as well as adaptive methods [S3].

## Background
Smoothed 2D Supervision was introduced as a technique to improve the robustness of joint optimization of camera poses and radiance fields [S1][S2, unofficial]. It draws inspiration from the analysis of spectral modulation in radiance fields, where blurring the 2D training images helps stabilize the optimization process [S2, unofficial]. It is often used in conjunction with other regularization techniques like randomly scaled kernel parameters and edge-guided loss masks [S1].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.1609/aaai.v38i2.27859, fetched 2026-10-26
[S2] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
[S3] Few-shot NeRF by Adaptive Rendering Loss Regularization, https://core.ac.uk/download/708671746.pdf, fetched 2026-10-30
[S4] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
