# Randomly Scaled Kernel Parameter

Also called: Random Kernel Scaling

## Overview
A regularization technique that mitigates weak and noisy gradient problems in joint camera pose and radiance field optimization by randomly scaling the size of 3D and 2D Gaussian kernels [S1, unofficial]. It sets itself apart from fixed-size smoothing by allowing combinations of different kernel scales to guide joint optimization, providing robustness across varying scene structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: 3D Gaussian splats (radiance field), 2D training images, current optimization iteration.
- Output: Scaled kernel parameters for the current iteration.

**Data Structures:**
- Kernel scale factor: A single floating-point scalar $s \in [0, 1]$.

**Steps:**
1. Sample a random scale factor $s_{3D}$ uniformly from the interval $[0, 1]$ for the 3D Gaussian kernels representing the radiance field [S1, unofficial].
2. Sample a random scale factor $s_{2D}$ uniformly from the interval $[0, 1]$ for the 2D Gaussian kernels applied to the training images [S1, unofficial].
3. Apply $s_{3D}$ and $s_{2D}$ independently to their respective kernels for the current optimization step [S1, unofficial].
4. Alternate between different randomly sampled kernel scales across training iterations to stabilize the schedule [S1, unofficial].

**Parameters:**
- Uniform distribution range: $[0, 1]$ [S1, unofficial].
- Sampling independence: Scales for 3D and 2D kernels are sampled independently [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_random_kernel_scales():
    """
    Returns independent random scale factors for 3D and 2D kernels.
    """
    # Sample s_3D uniformly from [0, 1]
    s_3d = random.uniform(0, 1)
    
    # Sample s_2D uniformly from [0, 1]
    s_2d = random.uniform(0, 1)
    
    return s_3d, s_2d
```

## Subtopics

### Edge Guided Loss
A loss weighting technique that increases the learning rate on edge pixels to amplify gradient signals for pose alignment, mitigating the weak and noisy gradient problem caused by aggressive filtering [S1, unofficial]. It detects edge areas using the Sobel filter on filtered 2D images and increases the learning rate by 1.5x for pixels in the edge region [S1, unofficial]. Empirically, this edge-guided scale is applied alternately on every other training iteration [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: Negligible overhead for sampling random numbers. Edge Guided Loss adds Sobel filter computation and conditional weighting [S1, unofficial].
- Memory: No additional memory required beyond standard optimization buffers [S1, unofficial].

**Measured Performance:**
- Improves robustness of training schedules and pose optimization by preventing overfitting to high-frequency noise and avoiding weak gradients from overly aggressive fixed kernels [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Adapts to different scene structures where a single kernel size might be overly aggressive or gentle [S1, unofficial].
- Limitations: Requires careful alternation of scales to maintain stability [S1, unofficial].

**When to Use:**
- Use when joint optimization of camera poses and radiance fields suffers from weak or noisy gradients due to fixed-size kernel smoothing [S1, unofficial].

## Background
Originated in the context of improving robustness for joint optimization of camera poses and decomposed low-rank tensorial radiance fields [S1, unofficial]. It addresses the insufficiency of using a single-size kernel for different real-world scene structures [S1, unofficial].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
