# Hole Filling

Also called: Depth Image Hole Filling, Statistical Hole Filling

## Overview
Hole filling is a post-processing step for projected depth images that replaces zero-valued pixels (holes) with estimated non-zero values, distinguishing between missing foreground surface information and valid background voids [S1]. It sets itself apart from general inpainting by using a statistical test on local neighborhood statistics ($\mu, \sigma$) to decide whether to apply a moving average filter, thereby preserving background areas that should remain empty [S1].

## How it works
**Inputs and Outputs:**
- Input: A projected depth image $D$ where holes are represented by zero values [S1].
- Output: A filled depth image $D'$ where holes corresponding to missing foreground surface are replaced with interpolated values [S1].

**Data Structures:**
- The depth image is treated as a 2D array of floating-point values [G].
- A kernel size $M \times M$ is used for local neighborhood calculations [S1].

**Parameters:**
- $M$: Kernel size for the moving average filter and neighborhood statistics calculation. Not given in the sources.
- $\kappa$: Threshold for the statistical test. Empirically found best value is 2 [S1].

**Steps:**
1. For each pixel $p$ in the depth image $D$ that has a value of 0 (a hole):
   a. Identify the $M \times M$ neighborhood of pixels centered at $p$ [S1].
   b. Calculate the mean $\mu$ and standard deviation $\sigma$ of the neighboring pixels [S1].
   c. Evaluate the condition:
      $$ \frac{p - \mu}{\sigma} > \kappa $$
      Since $p=0$ for a hole, this evaluates to $-\mu/\sigma > \kappa$ [S1].
   d. If the condition is satisfied, the hole is determined to be missing foreground surface information [S1].
   e. Fill the hole by applying a moving average filter with kernel size $M \times M$ to the neighborhood [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def hole_filling(depth_image: np.ndarray, kernel_size: int, kappa: float) -> np.ndarray:
    """
    Fills holes (zero values) in a depth image using statistical neighborhood analysis.
    
    Args:
        depth_image: 2D numpy array of depth values. Holes are 0.
        kernel_size: Size M of the M x M kernel.
        kappa: Threshold for the statistical test.
        
    Returns:
        Filled depth image.
    """
    filled_image = depth_image.copy()
    M = kernel_size
    half = M // 2
    hole_mask = (depth_image == 0)
    ys, xs = np.where(hole_mask)

    for y, x in zip(ys, xs):
        # Boundary-safe neighborhood extraction
        y0, y1 = max(0, y - half), min(depth_image.shape[0], y + half + 1)
        x0, x1 = max(0, x - half), min(depth_image.shape[1], x + half + 1)
        neighborhood = depth_image[y0:y1, x0:x1]
        
        mu = np.mean(neighborhood)
        sigma = np.std(neighborhood)
        
        # Guard against division by zero; standard practice skips degenerate windows [G]
        if sigma == 0:
            continue
            
        p = 0.0
        if (p - mu) / sigma > kappa:
            # Moving average filter of size M x M is equivalent to the neighborhood mean
            filled_value = np.mean(neighborhood)
            filled_image[y, x] = filled_value
            
    return filled_image
```

**Edge Cases and Pitfalls:**
- If $\kappa$ is too small, holes in background areas may be filled, blurring the depth image [S1].
- If $\kappa$ is too large, holes with missing foreground information may remain unfilled [S1].
- If the neighborhood has zero standard deviation ($\sigma = 0$), the condition is undefined; the pixel is skipped [G].

## Subtopics

### Diffusion Fill (Hole Inpainting)
Fills hole pixels in an image by diffusing values from their surroundings using iterative box filtering [S2, unofficial]. It initializes an output copy of the image and iteratively updates hole pixels using the ratio of a box-filtered numerator (output) to a box-filtered denominator (weight mask), updating weights and shrinking the hole mask until no holes remain or a maximum iteration count is reached [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources. For each hole, an $M \times M$ neighborhood is processed, yielding $O(H \cdot W \cdot M^2)$ in the worst case [G].
- **Memory Complexity:** Not given in the sources. Requires $O(H \cdot W)$ for the depth image and temporary neighborhood arrays [G].
- **Measured Performance:** On the DONeRF Lego dataset ($N=8, \alpha=1/16$), hole filling improves rendering PSNR from 27.29 dB (without filling) to a peak of 30.25 dB with $\kappa=1$ or $\kappa=2$ [S1].
- **Strengths:** Simple to implement, preserves background areas when $\kappa$ is chosen correctly, improves downstream rendering quality [S1].
- **Limitations:** Sensitive to $\kappa$ selection; poor choice leads to either residual holes or background blurring [S1]. Relies on local statistics, which may fail in textureless or uniform regions [G].

## Background
Originated in the context of improving Neural Radiance Fields (NeRF) using near-surface sampling with point cloud generation [S1]. The method was introduced to address the issue of zero-valued pixels in projected depth images derived from point clouds, which correspond to missing 3D surface information [S1]. It was empirically evaluated on synthetic (Lego, Ship) and real (BlendedMVS, Pavillon) datasets [S1].

## Sources
[S1] Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation, https://link.springer.com/article/10.1007/s11063-024-11654-5, fetched 2026-10-30
[S2] devarshh/volumetric-video-4dgs, https://github.com/devarshh/volumetric-video-4dgs, fetched 2026-10-26 (unofficial)
