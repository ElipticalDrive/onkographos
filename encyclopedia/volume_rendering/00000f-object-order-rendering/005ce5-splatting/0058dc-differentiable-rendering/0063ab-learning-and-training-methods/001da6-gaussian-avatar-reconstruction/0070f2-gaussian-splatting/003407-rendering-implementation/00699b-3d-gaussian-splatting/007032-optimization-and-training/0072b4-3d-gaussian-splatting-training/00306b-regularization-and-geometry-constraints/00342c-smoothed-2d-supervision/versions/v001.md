# Smoothed 2D Supervision


## Overview
A regularization technique that stabilizes the joint optimization of camera poses and 3D radiance fields by blurring 2D training images with scheduled Gaussian kernels [S1, unofficial]. It produces smoothed image gradients to stabilize camera alignment and restricts the spectrum of the learned 3D scene [S1, unofficial]. It differs from [Frequency Regularization](6130-frequency-regularization.md) by applying explicit spatial smoothing to the target supervision images rather than penalizing frequency-domain discrepancies in the loss.

## How it works
**Inputs:** 2D training image $I$, current iteration $t$, total iterations $T_{max}$ (typically 10,000), and kernel schedule parameters.
**Outputs:** Smoothed image $I_{smooth}$ used for photometric loss computation.

**Data Structures:**
*   **Gaussian Kernel:** Defined by standard deviation $\sigma$. The 2D kernel $K(x,y)$ is:
    $$ K(x,y) = \frac{1}{2\pi\sigma^2} \exp\left(-\frac{x^2+y^2}{2\sigma^2}\right) $$
*   **Schedule:** $\sigma$ is exponentially reduced to 0 at $T_{max}$ iterations and remains 0 afterward [S1, unofficial]. Exact $\sigma(t)$ values are in the supplement of [S1, unofficial].

**Steps:**
1.  **Schedule Generation:** Compute $\sigma_t$ for the current iteration $t$. The schedule mirrors that used for 3D radiance fields [S1, unofficial].
2.  **Convolution:** Convolve the training image $I$ with the 2D Gaussian kernel $K_{\sigma_t}$ to produce $I_{smooth}$ [S1, unofficial].
3.  **Loss Computation:** Compute the photometric loss $\mathcal{L}_{rgb}$ between the rendered image and $I_{smooth}$ instead of the original $I$ [S1, unofficial].
4.  **Gradient Backprop:** Gradients flow through the differentiable renderer and the smoothing operation (if differentiable) or through the loss term to update Gaussian parameters $\theta$ [S1, unofficial].

**Parameters:**
*   **Kernel Size ($\sigma$):** Starts at a value that provides significant smoothing and decays exponentially to 0 over 10,000 iterations [S1, unofficial].
*   **Random Scale Factor:** If using the variant, a factor $s \sim U[0, 1]$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Weak Gradients:** Overly large kernels reduce the magnitude of the gradient signal $\frac{d\mathcal{L}}{du}$, making pose alignment more susceptible to noise [S1, unofficial].
*   **Insufficient Single-Size:** A single kernel size may be too aggressive for some scene structures and too gentle for others [S1, unofficial].

## Subtopics

### Randomly Scaled Kernel Parameter
A variant that mitigates weak and noisy gradient problems by randomly scaling the kernel size [S1, unofficial]. It allows combinations of different-sized kernels to guide joint optimization [S1, unofficial].
*   **Definition:** Randomly scale the kernel by a factor uniformly sampled from $[0, 1]$ [S1, unofficial].
*   **Steps:**
    1.  Sample a random scale factor $s \sim U[0, 1]$ [S1, unofficial].
    2.  Sample random scales independently for 3D Gaussian kernels (radiance field) and 2D Gaussian kernels (training images) [S1, unofficial].
    3.  Apply the scaled kernel to the training image [S1, unofficial].
*   **When to choose:** When single-size kernels lead to robustness issues or when alternating between sampled kernel scales improves training stability [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Adds convolution cost per iteration. Depends on kernel size.
*   **Memory Complexity:** Minimal, requires storing the smoothed image or kernel.
*   **Strengths:** Stabilizes camera alignment; restricts 3D scene spectrum; robust to weak gradient problems [S1, unofficial].
*   **Limitations:** Large kernels weaken gradient signals, increasing noise sensitivity [S1, unofficial].
*   **When to use:** Joint optimization of poses and radiance fields; scenes with challenging geometry or lighting [S1, unofficial].
*   **When not:** When high-frequency detail preservation is critical without subsequent sharpening; when computational budget is extremely tight [S1, unofficial].

## Background
Originated in "Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields" [S1, unofficial]. Inspired by spectral analysis of radiance fields [S1, unofficial].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
