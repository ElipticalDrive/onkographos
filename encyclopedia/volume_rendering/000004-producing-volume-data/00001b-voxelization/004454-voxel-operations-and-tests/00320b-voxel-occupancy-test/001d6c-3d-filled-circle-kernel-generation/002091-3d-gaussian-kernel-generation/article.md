# 3D Gaussian Kernel Generation
Also called: 3D Gaussian Smoothing Kernel, 3D Gaussian Filter Kernel

## Overview
Generates a 3D Gaussian kernel for smoothing operations, with the center element explicitly set to zero [S1, unofficial]. It differs from its parent [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md) by using a continuous probability density function for weighted smoothing rather than a binary occupancy mask, and from [Sphere Carving](2377-sphere-carving.md) by providing isotropic (or elliptical via children) decay rather than hard geometric boundaries.

## How it works
**Inputs and Outputs:**
- Input: `size` (integer, kernel dimension), `sigma` (float, standard deviation). Optional: `scale` (float, ratio of grid size to AABB dimensions) and `c2f_parameter` (float, width multiplier) if used within a specific conversion pipeline [S2, unofficial].
- Output: A 3D numpy/tensor array of shape `(size, size, size)` containing normalized Gaussian weights with the center element equal to 0.

**Data Structures:**
- The kernel is stored as a dense 3D array. No bit-packing is used.

**Steps:**
1. Define the coordinate range: Create a linear space from `-(size // 2)` to `size // 2` (inclusive or exclusive depending on implementation, typically `np.linspace` or `np.arange` centered at 0). Let this be vector $v$ of length $size$.
2. Generate meshgrid: Create a 3D meshgrid of coordinates $(x, y, z)$ from $v$.
3. Compute radial distance: Calculate the Euclidean distance $d$ from the center $(0,0,0)$ for each element:
   $$ d = \sqrt{x^2 + y^2 + z^2} $$
4. Compute Gaussian values: Calculate the unnormalized weight for each element:
   $$ w = \exp\left(-\frac{d^2}{2\sigma^2}\right) $$
   If a `scale` factor is provided (e.g., from AABB ratio), the effective width may be adjusted as `width = scale * c2f_parameter`, potentially modifying $\sigma$ or the grid spacing [S2, unofficial].
5. Zero the center: Set the element at the center index `(size // 2, size // 2, size // 2)` to 0.
6. Normalize: Divide all elements by the sum of all elements (excluding the center if it was zeroed before summation, or including it if zeroed after, but since center is 0, it doesn't affect the sum) to ensure the kernel sums to 1.

**Parameters:**
- `size`: Kernel dimension. Default is 3 [S1, unofficial].
- `sigma`: Standard deviation. Default is 1.0 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_3d_gaussian_kernel(size=3, sigma=1.0):
    """
    Generates a 3D Gaussian kernel with the center element set to 0.
    
    Args:
        size (int): The dimension of the cubic kernel.
        sigma (float): The standard deviation of the Gaussian distribution.
        
    Returns:
        numpy.ndarray: A 3D array of shape (size, size, size) with normalized weights.
    """
    # 1. Create linear space from -(size//2) to size//2
    # Using linspace to ensure symmetry and correct centering
    half = size // 2
    coords = np.linspace(-half, half, size)
    
    # 2. Generate 3D meshgrid of coordinates
    x, y, z = np.meshgrid(coords, coords, coords, indexing='ij')
    
    # 3. Compute radial distance d from center
    d_squared = x**2 + y**2 + z**2
    
    # 4. Compute Gaussian values: exp(-d^2 / (2*sigma^2))
    # Avoid division by zero if sigma is 0, though default is 1.0
    if sigma == 0:
        kernel = np.zeros((size, size, size))
    else:
        kernel = np.exp(-d_squared / (2 * sigma**2))
        
    # 5. Set center element to 0
    center_idx = half
    kernel[center_idx, center_idx, center_idx] = 0.0
    
    # 6. Normalize by sum of all elements
    kernel_sum = np.sum(kernel)
    if kernel_sum > 0:
        kernel = kernel / kernel_sum
    else:
        # Fallback if all weights are 0 (e.g., sigma=0 and size=1)
        kernel = np.zeros((size, size, size))
        
    return kernel
```

## Subtopics

### [Elliptical Gaussian Light Source Generation](5806-elliptical-gaussian-light-source-generation.md)
Generates spatially distributed light sources using either Poisson disk or uniform sampling, then constructs a luminance field by evaluating scaled multivariate Gaussian probability density functions (PDFs) at query points. It differs from its parent [3D Gaussian Kernel Generation](2297-3d-gaussian-kernel-generation.md) by treating the Gaussian as an emission model for light sources rather than a smoothing kernel, and by supporting spatially varying elliptical covariances via eigen-decomposition.

### [Separable Component-Wise Convolution](7446-separable-component-wise-convolution.md)
Separable component-wise convolution is a computationally efficient method for applying 3D Gaussian smoothing to volumes represented as decomposed low-rank tensors. It solves the problem of high computational cost and memory usage associated with naively convolving a 3D kernel on a reconstructed full tensor volume. Unlike traditional separated kernel convolution which only exploits kernel separability, this method distributes the convolution across individual vector and matrix components of the decomposed tensor, leveraging the input signal's inherent separability to maintain space compression and reduce time complexity.

## Operating characteristics
- **Time Complexity:** $O(N^3)$ where $N$ is the kernel size `size`, as every element must be computed.
- **Memory Complexity:** $O(N^3)$ to store the kernel.
- **Strengths:** Simple to implement; provides smooth, isotropic smoothing.
- **Limitations:** The center-zero constraint makes it unsuitable for standard blurring where the center should retain maximum weight; it is better suited for edge detection or specific gradient-based operations. Small kernel sizes (e.g., 3x3x3) may not capture long-range dependencies.
- **When to use:** When a smoothing kernel with a null center is required for specific volume rendering or voxelization steps.
- **When not to use:** For standard image/volume blurring where the center should be the peak.

## Background
Originated in the context of voxel-based neural rendering and volume processing, specifically within implementations like Plenoxels [S1, unofficial] and TensoRF variants [S2, unofficial]. The center-zero variant is a specific modification to the standard Gaussian kernel, likely to prevent self-interference or to act as a difference-of-Gaussians-like operator in certain pipelines.

## Sources
[S1] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S2] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
