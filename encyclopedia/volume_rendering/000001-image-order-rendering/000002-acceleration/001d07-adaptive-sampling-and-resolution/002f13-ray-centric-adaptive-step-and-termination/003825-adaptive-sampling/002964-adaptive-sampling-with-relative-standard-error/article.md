# Adaptive Sampling with Relative Standard Error

Also called: Noise-Threshold Adaptive Sampling

## Overview
An adaptive sampling strategy that tracks per-pixel luminance squares to estimate relative standard error, stopping converged pixels and skipping noisy neighborhoods [S1, unofficial]. It differs from [Adaptive Sampling](1385-adaptive-sampling.md) by using statistical error metrics rather than geometric proximity to the isosurface to determine sampling rates, and from [Adaptive Sampling (Sparse Grid)](1656-adaptive-sampling-sparse-grid.md) by operating in image order per-pixel rather than modifying the underlying voxel data structure.

## How it works
**Inputs and Outputs:**
*   **Input:** Per-pixel luminance samples accumulated in a film buffer; a noise threshold parameter (not given in the sources).
*   **Output:** A decision mask indicating which pixels require further sampling and which are converged.

**Data Structures:**
*   **Film Buffer:** Stores per-pixel accumulated luminance $L$ and accumulated luminance squares $L^2$. The exact bit packing and field sizes are not given in the sources [G].

**Steps:**
1.  **Accumulate Samples:** For each new sample ray contributing to a pixel, update the accumulated luminance $L$ and the accumulated luminance squares $L^2$.
2.  **Estimate Error:** Calculate the relative standard error (RSE) for each pixel. The formula for RSE is not explicitly provided in the sources, but it is derived from the variance of the luminance squares [S1, unofficial]. Let $N$ be the number of samples. The variance $\sigma^2$ is estimated from $L^2$ and $L$. The RSE is typically $\frac{\sigma}{\sqrt{N} \cdot \mu}$ or similar, where $\mu$ is the mean luminance. The sources state the method "reports the relative standard error" [S1, unofficial].
3.  **Final Render Termination:** In final (non-progressive) renders, check every 8 samples. If a pixel's RSE is below the noise threshold, stop sampling that pixel [S1, unofficial].
4.  **Progressive Pass Skipping:** In progressive passes, examine the 3x3 neighborhood of each pixel. If the entire 3x3 neighborhood has an RSE below the noise threshold, skip sampling for those pixels in the current pass [S1, unofficial].

**Parameters:**
*   **Noise Threshold:** The RSE value below which a pixel is considered converged. Not given in the sources.
*   **Check Interval:** 8 samples for final renders [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Tracking luminance squares can lead to numerical instability if the accumulated values become very large or if the variance is small relative to the mean. Not given in the sources.
*   **Boundaries:** The 3x3 neighborhood check at image boundaries requires handling pixels with fewer than 8 neighbors. Not given in the sources.
*   **Degenerate Input:** Pixels with zero luminance (e.g., pure black) may cause division by zero in RSE calculation. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Adaptive Sampling with Relative Standard Error
# Assumes a FilmBuffer class with methods to accumulate and query stats

class FilmBuffer:
    def __init__(self, width, height):
        self.width = width
        self.height = height
        # Accumulated luminance L
        self.luminance = [[0.0 for _ in range(width)] for _ in range(height)]
        # Accumulated luminance squares L^2
        self.luminance_squares = [[0.0 for _ in range(width)] for _ in range(height)]
        # Sample count N
        self.sample_count = [[0 for _ in range(width)] for _ in range(height)]
        # Convergence mask
        self.converged = [[False for _ in range(width)] for _ in range(height)]

    def accumulate_sample(self, x, y, luminance):
        self.luminance[y][x] += luminance
        self.luminance_squares[y][x] += luminance * luminance
        self.sample_count[y][x] += 1

    def get_relative_standard_error(self, x, y):
        n = self.sample_count[y][x]
        if n == 0:
            return 0.0
        mean_luminance = self.luminance[y][x] / n
        mean_luminance_sq = self.luminance_squares[y][x] / n
        # Variance = E[X^2] - (E[X])^2
        variance = mean_luminance_sq - (mean_luminance ** 2)
        if variance < 0:
            variance = 0.0
        std_dev = variance ** 0.5
        # Relative Standard Error
        if mean_luminance == 0:
            return 0.0
        rse = std_dev / (mean_luminance * (n ** 0.5))
        return rse

    def is_converged(self, x, y, threshold):
        if self.converged[y][x]:
            return True
        rse = self.get_relative_standard_error(x, y)
        if rse <= threshold:
            self.converged[y][x] = True
            return True
        return False

    def is_neighborhood_converged(self, x, y, threshold):
        # Check 3x3 neighborhood
        for dy in [-1, 0, 1]:
            for dx in [-1, 0, 1]:
                nx, ny = x + dx, y + dy
                if 0 <= nx < self.width and 0 <= ny < self.height:
                    if not self.is_converged(nx, ny, threshold):
                        return False
        return True

def render_adaptive(film_buffer, noise_threshold, is_progressive, check_interval=8):
    sample_counter = 0
    while not all_pixels_converged(film_buffer, noise_threshold):
        sample_counter += 1
        for y in range(film_buffer.height):
            for x in range(film_buffer.width):
                if film_buffer.converged[y][x]:
                    continue
                
                if is_progressive:
                    # Skip if 3x3 neighborhood is converged
                    if film_buffer.is_neighborhood_converged(x, y, noise_threshold):
                        continue
                
                # Perform sampling
                luminance = sample_ray(x, y)
                film_buffer.accumulate_sample(x, y, luminance)
                
                # Check for convergence in final renders
                if not is_progressive and sample_counter % check_interval == 0:
                    film_buffer.is_converged(x, y, noise_threshold)
```

## Subtopics

### Monte Carlo Averaging for Noise Reduction
A technique used in conjunction with adaptive sampling to reduce Monte Carlo noise by rendering the predicted color image multiple times and averaging the results [S2]. Specifically, 64 secondary rays are used per primary ray, and the image is rendered 32 times and averaged [S2]. A pitfall is that renderings from baselines like TensoIR may still contain residual noise despite averaging, indicating that sample counts and importance sampling schemes significantly affect final quality [S2].

## Operating characteristics
*   **Time and Memory Complexity:** Memory complexity is $O(W \times H)$ for the film buffer storing luminance, luminance squares, and sample counts. Time complexity is proportional to the number of samples taken by non-converged pixels.
*   **Measured Performance:** On a showcase scene at the same mean samples per pixel (spp), the relative mean squared error (relMSE) was 0.0036 for uniform sampling (extrapolated) versus 0.0024 for adaptive sampling [S1, unofficial].
*   **Strengths:** Reduces computational overhead by stopping sampling on converged pixels and skipping noisy neighborhoods in progressive passes [S1, unofficial].
*   **Limitations:** Requires tracking additional state (luminance squares) per pixel. The effectiveness depends on the choice of noise threshold.
*   **When to Use:** When visual quality is prioritized and computational resources are limited, especially in scenes with varying levels of noise.
*   **When Not to Use:** When a strict time limit is imposed and the overhead of checking convergence is too high, or when the scene has uniform noise characteristics where uniform sampling might be simpler.

## Background
This method is described in the context of a photon engine implementation, specifically in a pull request adding volumetric rendering and advanced materials [S1, unofficial]. It is part of a broader set of features including thin lens cameras, indirect clamping, and CLI controls for noise and adaptive sampling [S1, unofficial].

## Sources
[S1] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S2] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
