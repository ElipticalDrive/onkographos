# Separable 3D Gaussian Filter
Also called: 3D Gaussian Filter via Separable 1D Filters, Gaussian filter
## Overview
A 3D Gaussian filter implemented as three consecutive 1D filters to leverage separability for interactive performance [S1, unofficial]. It serves as a low-pass filter to reduce high-frequency elements in volume data, often used to compute spatial importance maps for volume illustration [S1, unofficial]. It is distinguished from its parent, [Dr. TVAM Convolution](8425-dr-tvam-convolution.md), by its specific application in illustrative volume rendering pipelines rather than tomographic additive manufacturing simulations, and from its sibling, [3D Gaussian Smoothing](21379-3d-gaussian-smoothing.md), by its explicit decomposition into axis-aligned passes rather than a binomial approximation via texture sampling.
## How it works
**Inputs and Outputs:**
- Input: A 3D scalar volume $V$ of dimensions $D_x \times D_y \times D_z$.
- Output: A filtered 3D scalar volume $V_{filtered}$ and a difference volume $V_{diff}$ (optional, used for edge enhancement).

**Data Structures:**
- The volume is stored as a contiguous array or 3D texture.
- The 1D Gaussian kernel $G(x)$ is defined by the standard deviation $\sigma$.

**Formulas:**
The 1D Gaussian function is defined as [S1, unofficial]:
$$ G(x) = \frac{1}{\sqrt{2\pi}\sigma} e^{-\frac{x^2}{2\sigma^2}} $$
where $x$ is the distance from the origin and $\sigma$ is the standard deviation. The 3D Gaussian is the product of three 1D Gaussians, one per direction.

**Parameters:**
- $\sigma$: Standard deviation of the Gaussian distribution.
- $R$: Filter radius. The sampling rate is controlled by specifying the number of standard deviations spanning the filter radius, allowing frequency adjustment without changing the radius [S2, unofficial].
- Typical values: Not given in the sources. Table 1 in [S1, unofficial] lists radii $R$ for specific datasets but does not specify the corresponding $\sigma$ or the exact number of standard deviations used.

**Steps:**
1. **Axis 1 Convolution:** Apply 1D Gaussian convolution along the first axis (e.g., X). For each voxel $(i, j, k)$, compute the weighted sum of neighbors along the X-axis using the kernel $G(x)$.
2. **Axis 2 Convolution:** Apply 1D Gaussian convolution along the second axis (e.g., Y) to the result of step 1.
3. **Axis 3 Convolution:** Apply 1D Gaussian convolution along the third axis (e.g., Z) to the result of step 2.
4. **Subtraction:** Subtract the original volume from the filtered volume to obtain the difference (high-frequency component) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def separable_3d_gaussian_filter(volume, sigma, radius):
    """
    Applies a separable 3D Gaussian filter and returns the filtered volume and the difference.
    
    Args:
        volume: 3D array of shape (Dx, Dy, Dz)
        sigma: Standard deviation of the Gaussian kernel
        radius: Filter radius in voxels
        
    Returns:
        filtered_volume: 3D array after 3D Gaussian blur
        difference_volume: 3D array (filtered_volume - volume)
    """
    # Define 1D Gaussian kernel
    # Kernel size is 2 * radius + 1
    kernel_size = 2 * radius + 1
    kernel = []
    for i in range(-radius, radius + 1):
        x = i
        val = (1.0 / (math.sqrt(2 * math.pi) * sigma)) * math.exp(-(x**2) / (2 * sigma**2))
        kernel.append(val)
    
    # Normalize kernel to sum to 1
    kernel_sum = sum(kernel)
    kernel = [k / kernel_sum for k in kernel]
    
    # Step 1: Convolve along X axis
    # Create a temporary volume for intermediate results
    temp_volume_x = copy(volume)
    for z in range(volume.shape[2]):
        for y in range(volume.shape[1]):
            for x in range(volume.shape[0]):
                sum_val = 0.0
                for k_idx, k_val in enumerate(kernel):
                    neighbor_x = x + (k_idx - radius)
                    # Handle boundary conditions (e.g., clamp or wrap)
                    if 0 <= neighbor_x < volume.shape[0]:
                        sum_val += volume[neighbor_x, y, z] * k_val
                temp_volume_x[x, y, z] = sum_val
                
    # Step 2: Convolve along Y axis
    temp_volume_y = copy(temp_volume_x)
    for z in range(volume.shape[2]):
        for y in range(volume.shape[1]):
            for x in range(volume.shape[0]):
                sum_val = 0.0
                for k_idx, k_val in enumerate(kernel):
                    neighbor_y = y + (k_idx - radius)
                    if 0 <= neighbor_y < volume.shape[1]:
                        sum_val += temp_volume_x[x, neighbor_y, z] * k_val
                temp_volume_y[x, y, z] = sum_val
                
    # Step 3: Convolve along Z axis
    filtered_volume = copy(temp_volume_y)
    for z in range(volume.shape[2]):
        for y in range(volume.shape[1]):
            for x in range(volume.shape[0]):
                sum_val = 0.0
                for k_idx, k_val in enumerate(kernel):
                    neighbor_z = z + (k_idx - radius)
                    if 0 <= neighbor_z < volume.shape[2]:
                        sum_val += temp_volume_y[x, y, neighbor_z] * k_val
                filtered_volume[x, y, z] = sum_val
                
    # Step 4: Subtract original volume
    difference_volume = copy(filtered_volume)
    for z in range(volume.shape[2]):
        for y in range(volume.shape[1]):
            for x in range(volume.shape[0]):
                difference_volume[x, y, z] = filtered_volume[x, y, z] - volume[x, y, z]
                
    return filtered_volume, difference_volume
```

**Edge Cases and Pitfalls:**
- Boundary conditions: The pseudocode above uses clamping. Other methods may use reflection or zero-padding. The source does not specify the boundary handling method [S1, unofficial].
- Precision: Floating-point precision should be sufficient for the exponential calculation.
- Radius and Sigma relationship: The radius must be chosen large enough to capture the significant part of the Gaussian distribution. A common rule of thumb is $R \ge 3\sigma$, but the source notes that the sampling rate can be altered by specifying the number of standard deviations spanning the radius [S2, unofficial].

## Subtopics

### [Bilateral Filter](8983-bilateral-filter.md)
A non-linear filter used to reduce noise while preserving edges by weighting pixels based on both spatial proximity and value similarity. Unlike the [Separable 3D Gaussian Filter](8956-separable-3d-gaussian-filter.md) which only uses spatial distance, the Bilateral filter also uses value similarity, preserving edges better.

### [3D Gaussian Smoothing](21379-3d-gaussian-smoothing.md)
3D Gaussian Smoothing is a voxelization post-processing technique that convolves a spectral cube with a 3D Gaussian distribution to reduce noise and provide smoother transitions than box filters. It is distinguished from its sibling, the [Bilateral Filter](8983-bilateral-filter.md), by its reliance on spatial proximity for weighting rather than intensity differences, and from its parent, the [Separable 3D Gaussian Filter](8956-separable-3d-gaussian-filter.md), by its specific implementation using a binomial approximation via bilinear texture sampling to minimize texture fetches.

## Operating characteristics
**Time and Memory Complexity:**
- Time Complexity: $O(N \cdot R)$, where $N$ is the number of voxels and $R$ is the filter radius. Due to separability, the complexity is $O(3 \cdot N \cdot R)$ instead of $O(N \cdot R^3)$ for a direct 3D convolution.
- Memory Complexity: $O(N)$ for the input and output volumes, plus $O(R)$ for the kernel.

**Measured Performance:**
- Table 1 in [S1, unofficial] provides times for a CUDA implementation on CT body (256 × 256 × 415), CT head (512 × 512 × 297), and salmon (336 × 173 × 511) datasets.
- For CT body (256 × 256 × 415):
  - 1D filter time: Not given in the sources (Table 1 column exists but values are missing in the text).
  - Subtract time: Not given in the sources.
  - Total time: Not given in the sources.
- The source states that the separability feature allows obtaining interactive rates with a CUDA implementation [S1, unofficial].

**Scaling:**
- The method scales linearly with the number of voxels and the filter radius.
- The CUDA implementation allows for parallel processing of voxels, improving performance on large datasets [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Interactive performance due to separability and GPU implementation [S1, unofficial]. Provides a low-pass filter effect [S1, unofficial].
- Limitations: Still requires calculating distances to neighboring data points to determine data frequency and set filter parameters [S2, unofficial]. The choice of $\sigma$ and $R$ affects the result, and the source notes that further investigation is required to determine the functions used to set these parameters [S2, unofficial].

**When to Use:**
- When interactive performance is required for volume illustration [S1, unofficial].
- When a low-pass filter effect is desired to reduce high-frequency noise [S1, unofficial].

**When Not to Use:**
- When edge preservation is critical (consider [Bilateral Filter](8983-bilateral-filter.md)) [S1, unofficial].
- When a simpler approximation is sufficient and texture sampling is available (consider [3D Gaussian Smoothing](21379-3d-gaussian-smoothing.md)) [S1, unofficial].

## Background
The method is described in the context of "Interactive Volume Illustration Using Intensity Filtering" [S1, unofficial]. It is used to compute a spatial importance map for volume rendering, which is then used to modulate color and opacity assigned by a transfer function [S1, unofficial]. The separability of the Gaussian filter is key to achieving interactive rates [S1, unofficial]. The source also references a broader context of sampling scattered data onto rectangular grids for volume visualization [S2, unofficial], where Gaussian filters are compared to distance weighting filters [S2, unofficial].

## Sources
[S1] Interactive Volume Illustration Using Intensity Filtering, https://doi.org/10.2312/compaesth/compaesth10/051-058, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
