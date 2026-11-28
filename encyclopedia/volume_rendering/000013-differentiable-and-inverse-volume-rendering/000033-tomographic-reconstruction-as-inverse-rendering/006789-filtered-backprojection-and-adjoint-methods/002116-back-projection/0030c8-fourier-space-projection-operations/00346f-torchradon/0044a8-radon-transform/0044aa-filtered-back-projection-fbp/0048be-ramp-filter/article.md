# Ramp Filter
## Overview
The Ramp Filter is a frequency-domain filter applied to projection data prior to backprojection in Filtered Back-Projection (FBP) [S1, unofficial]. It corrects for the blurring inherent in simple backprojection by weighting frequencies proportional to their magnitude [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A 1D projection array $p$ of width $N$ [G].
- Output: A filtered 1D projection array $\hat{p}$ [G].
**Data Structures:**
- The projection $p$ is stored as a 1D array of floating-point values [G].
- Frequency bins are computed as a 1D array of floats [G].
**Steps:**
1. Compute the frequency bins $f$ corresponding to the projection width $N$ [S1, unofficial].
2. Compute the Fast Fourier Transform (FFT) of the projection $p$ [S1, unofficial].
3. Multiply the FFT result by the absolute value of the frequency bins (the ramp filter kernel) [S1, unofficial].
4. Compute the Inverse Fast Fourier Transform (IFFT) of the result to obtain the filtered projection $\hat{p}$ [G].
**Formula:**
$$ \hat{p} = \text{IFFT}\left( \text{FFT}(p) \cdot |f| \right) $$
where $f$ is the array of frequency bins [S1, unofficial].
**Parameters:** Not given in the sources.
**Edge Cases and Pitfalls:**
- **Degenerate Input:** If the projection is all zeros, the output is zero [G].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def ramp_filter(projection):
    N = len(projection)
    freq_bins = np.abs(np.arange(N) - N / 2.0)
    fft_projection = np.fft.fft(projection)
    filtered_fft = fft_projection * freq_bins
    filtered_projection = np.fft.ifft(filtered_fft).real
    return filtered_projection
```
## Subtopics

### Fourier and Poisson Filtering
Applies a Fourier filter and Poisson weights to residual projection data [S2, unofficial]. Choose this when residual updates require frequency-domain filtering [S2, unofficial].
- **Definition:** Applies a Fourier filter and Poisson weights to a residual projection data [S2, unofficial].
- **Steps:**
    1. If `fourier_filter` is None, compute simple difference: `forward = projections - forward` [S2, unofficial].
    2. Else, apply Fourier filter: `forward = fourier_filter * numpy.fft.fft2(projections, axes=(0, 2))` [S2, unofficial].
    3. Subtract filtered forward projection: `forward -= (fourier_filter**2) * numpy.fft.fft2(forward, axes=(0, 2))` [S2, unofficial].
    4. Compute inverse FFT and take absolute value: `forward = numpy.abs(numpy.fft.ifft2(forward, axes=(0, 2))).astype('float32')` [S2, unofficial].
    5. If `poisson` is True, apply Poisson weights: `forward *= numpy.exp(-projections / projections.mean())` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ per projection due to FFT [G].
- **Memory Complexity:** $O(N)$ for storing the projection and frequency bins [G].
- **Strengths:** Fast, closed-form solution [G].
- **When to use:** Standard FBP reconstruction [S1, unofficial].
## Background
The Ramp Filter is a fundamental component of Filtered Back-Projection (FBP), a standard algorithm in computed tomography (CT) reconstruction [S1, unofficial]. It was developed as part of the mathematical foundation of tomographic reconstruction, correcting the blurring introduced by simple backprojection [S1, unofficial].
## Sources
[S1] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
[S2] cicwi/flexTOMO, https://github.com/cicwi/flexTOMO, fetched 2026-10-27 (unofficial)
