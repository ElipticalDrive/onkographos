# Randomly Scaled Kernel Parameter

Also called: Random Kernel Scaling

## Overview
Randomly Scaled Kernel Parameter is a regularization technique used during the joint optimization of camera poses and radiance fields to mitigate weak and noisy gradient signals [S1, unofficial]. It addresses the insufficiency of using a single-size kernel for diverse real-world scene structures, where a fixed kernel may be overly aggressive in some contexts and overly gentle in others [S1, unofficial]. By randomly scaling the kernel size, it allows combinations of different-sized kernels to guide the optimization, resulting in a more robust training schedule [S1, unofficial]. It sets itself apart from uniform weighting by dynamically adjusting the spectral influence of the smoothing kernels during training.

## How it works
**Inputs and Outputs:**
- Input: A base kernel size (standard deviation $\sigma$) for 3D Gaussian splats and 2D Gaussian blurring kernels.
- Output: Scaled kernel parameters applied during the forward pass of the differentiable renderer and the supervision loss computation.

**Data Structures:**
- No specific complex data structures are required beyond standard floating-point variables for kernel parameters.

**Steps:**
1. **Sample Scale Factors:** For each training iteration (or step), sample two independent scale factors, $s_{3D}$ and $s_{2D}$, uniformly from the interval $[0, 1]$.
   $$ s_{3D}, s_{2D} \sim \mathcal{U}(0, 1) $$
2. **Apply to 3D Kernels:** Scale the standard deviation of the 3D Gaussian kernels used in the radiance field representation. If the base standard deviation is $\sigma_{3D}$, the effective standard deviation becomes:
   $$ \sigma'_{3D} = s_{3D} \cdot \sigma_{3D} $$
3. **Apply to 2D Kernels:** Scale the standard deviation of the 2D Gaussian kernels used for smoothing the 2D training images (as part of Smoothed 2D Supervision). If the base standard deviation is $\sigma_{2D}$, the effective standard deviation becomes:
   $$ \sigma'_{2D} = s_{2D} \cdot \sigma_{2D} $$
4. **Render and Compute Loss:** Use the scaled kernels $\sigma'_{3D}$ and $\sigma'_{2D}$ for the differentiable rendering and the computation of the smoothed 2D supervision loss.

**Parameters:**
- Scale range: $[0, 1]$ for both $s_{3D}$ and $s_{2D}$ [S1, unofficial].
- Sampling distribution: Uniform [S1, unofficial].

**Edge Cases and Pitfalls:**
- A scale factor of 0 results in a Dirac delta function (no smoothing/blurring), which may lead to high-frequency noise if the scene is complex [S1, unofficial].
- A scale factor close to 1 results in significant smoothing, potentially blurring fine details.
- The independence of $s_{3D}$ and $s_{2D}$ allows for diverse combinations of 3D splat sizes and 2D blur strengths, which is critical for robustness [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sample_scaled_kernels(base_sigma_3d, base_sigma_2d):
    """
    Samples random scale factors and applies them to 3D and 2D kernel parameters.
    
    Args:
        base_sigma_3d (float): Base standard deviation for 3D Gaussian kernels.
        base_sigma_2d (float): Base standard deviation for 2D Gaussian kernels.
        
    Returns:
        tuple: (scaled_sigma_3d, scaled_sigma_2d)
    """
    # Sample scale factors uniformly from [0, 1]
    s_3d = np.random.uniform(0, 1)
    s_2d = np.random.uniform(0, 1)
    
    # Apply scales to the base standard deviations
    scaled_sigma_3d = s_3d * base_sigma_3d
    scaled_sigma_2d = s_2d * base_sigma_2d
    
    return scaled_sigma_3d, scaled_sigma_2d
```

## Subtopics

### [Edge Guided Loss](9166-edge-guided-loss.md)
Edge Guided Loss is a regularization technique used during the joint optimization of camera poses and radiance fields to mitigate weak and noisy gradient signals. It improves robustness by selectively amplifying the learning signal on edge pixels, which are critical for pose alignment, thereby stabilizing the optimization process when using smoothed 2D supervision. It sets itself apart from uniform weighting by dynamically adjusting the loss contribution based on local image structure, specifically targeting the high-frequency information preserved by edge detection.

## Operating characteristics
- **Strengths:** Mitigates the problem of weak and noisy gradients by preventing the optimizer from relying on a single, potentially suboptimal, kernel scale [S1, unofficial]. It makes the training schedule more robust across different scene structures [S1, unofficial].
- **Limitations:** Introduces variability in the smoothing/blurring effect per iteration, which may require careful tuning of the base kernel sizes [S1, unofficial].
- **When to use:** When using Smoothed 2D Supervision in joint optimization of camera poses and radiance fields, especially in real-world scenes with varying structures [S1, unofficial].
- **When not to use:** If the scene is very simple and a single kernel size is sufficient, or if computational stability is a primary concern and variability is undesirable.

## Background
The technique was introduced in the context of improving robustness for the joint optimization of camera poses and decomposed low-rank tensorial radiance fields [S1, unofficial]. It was developed to address the limitations of fixed-size kernels in Smoothed 2D Supervision, which were found to be insufficient for diverse real-world scene structures [S1, unofficial]. The authors observed that alternating between randomly sampled kernel scales led to more robust training [S1, unofficial].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
