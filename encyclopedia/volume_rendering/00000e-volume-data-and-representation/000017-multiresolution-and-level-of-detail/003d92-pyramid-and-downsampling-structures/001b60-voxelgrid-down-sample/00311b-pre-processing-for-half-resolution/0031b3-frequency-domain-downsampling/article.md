# Frequency-Domain Downsampling

Also called: 3D Downsampling Module, FFT-based Downsampling, Fourier Downsampling

## Overview
Frequency-Domain Downsampling is a non-learnable pre-processing method that reduces volumetric data resolution by transforming it to the frequency domain, applying a low-pass filter, and transforming it back to the spatial domain [S1]. It reduces data size while preserving essential low-frequency information, avoiding information loss from direct subsampling [S1]. Unlike learned downsampling kernels, it uses explicit frequency-domain filtering to generate controllable scales (×1/2, ×1/4, ×1/8) without training [S1].

## How it works
**Inputs and Outputs:**
- Input: High-resolution volume $x$ [S1].
- Output: Low-resolution volume $y$ [S1].

**Data Structures:**
- The input $x$ is a 3D array of voxel intensities [S1].
- The filter $D$ is a 3D low-pass operator in the frequency domain [S1].
- The noise term $n$ is a 3D array added to the output [S1].

**Parameters:**
- Downsampling scales: ×1/2, ×1/4, and ×1/8 [S1].
- $D$: Low-pass filter operator [S1].
- $n$: Additive noise term [S1].

**Steps:**
1. Apply a high-resolution Fast Fourier Transform (FFT) to the input volume $x$ to obtain the frequency representation $F_{HR}(x)$ [S1].
2. Multiply the frequency representation by the low-pass filter $D$ to eliminate high-frequency components [S1].
3. Apply an inverse FFT for the low-resolution regime ($F_{LR}^{-1}$) to revert the data to the spatial domain, yielding the low-resolution version [S1].
4. Add the noise term $n$ to produce the output $y$ [S1].

**Formula:**
The relationship between the input $x$ and output $y$ is defined as [S1]:

$$y = F_{LR}^{-1} D F_{HR} x + n$$

Where:
- $x$: High-resolution input volume [S1].
- $y$: Low-resolution output volume [S1].
- $F_{HR}$: FFT operator for the high-resolution regime [S1].
- $F_{LR}^{-1}$: Inverse FFT operator for the low-resolution regime [S1].
- $D$: Low-pass filter in the frequency domain [S1].
- $n$: Noise term [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def frequency_domain_downsample(x, scale_factor, noise_std=0.0):
    """
    Reduces the resolution of a 3D volume using frequency-domain filtering.
    
    Args:
        x: numpy array of shape (H, W, D), high-resolution volume.
        scale_factor: int, downsampling scale (e.g., 2, 4, 8).
        noise_std: float, standard deviation of additive noise.
        
    Returns:
        y: numpy array of shape (H/scale, W/scale, D/scale), low-resolution volume.
    """
    # Step 1: Apply high-resolution FFT
    freq_domain = np.fft.fftn(x)
    
    # Step 2: Apply low-pass filter D
    # The exact filter design is not specified in the source [G].
    # We construct an ideal low-pass filter mask centered in frequency space.
    axes = [np.fft.fftfreq(d) for d in x.shape]
    grid = np.meshgrid(*axes, indexing='ij')
    dist = np.sqrt(sum(g**2 for g in grid))
    cutoff = 1.0 / (2.0 * scale_factor)
    D = (dist <= cutoff).astype(np.complex128)
    filtered_freq = freq_domain * D
    
    # Step 3: Apply inverse low-resolution FFT
    # The source formula implies F_LR^-1 operates on the filtered data.
    # Standard implementation crops the frequency spectrum to the target size [G].
    new_shape = (x.shape[0] // scale_factor, x.shape[1] // scale_factor, x.shape[2] // scale_factor)
    slices = []
    for orig_dim, target_dim in zip(x.shape, new_shape):
        start = (orig_dim - target_dim) // 2
        end = start + target_dim
        slices.append(slice(start, end))
    cropped_freq = filtered_freq[tuple(slices)]
    y_spatial = np.fft.ifftn(cropped_freq).real
    
    # Step 4: Add noise term n
    if noise_std > 0:
        noise = np.random.normal(0, noise_std, y_spatial.shape)
        y = y_spatial + noise
    else:
        y = y_spatial
        
    return y
```

**Edge Cases and Pitfalls:**
- **Non-Standard Scales:** Non-standard sampling scales like 1/3, 1/5, or 1/7 introduce unnecessary complexity and inconsistencies, so standard integer factors are recommended [S1].
- **Noise Purpose:** The exact purpose of the noise term $n$ is not detailed in the source [G].

## Operating characteristics
- **Training:** The method does not require training [S1].
- **Scales:** Typical tested downsampling scales are 1/2, 1/4, and 1/8 [S1].
- **Quality:** Decreasing the downsampling scale does not significantly degenerate reconstruction quality in the tested architecture [S1].
- **Integration:** It is used as a pre-processing step to reduce data size for subsequent modules like an Implicit Neural Representation (INR) and Super-Resolution (SR) [S1].

## Background
The method originates from Sheibani Fard et al. (2026) in "An end-to-end implicit neural representation architecture for medical volume data" [S1]. It is part of an end-to-end architecture that includes an INR and an SR module [S1]. The downsampling module is designed to be non-learnable to reduce the computational load for the INR module [S1]. Fourier Transform techniques are widely employed in medical imaging, and this method adapts them for volumetric downsampling [S1].

## Sources
[S1] An end-to-end implicit neural representation architecture for medical volume data., https://doi.org/10.1371/journal.pone.0314944, fetched 2026-10-26
