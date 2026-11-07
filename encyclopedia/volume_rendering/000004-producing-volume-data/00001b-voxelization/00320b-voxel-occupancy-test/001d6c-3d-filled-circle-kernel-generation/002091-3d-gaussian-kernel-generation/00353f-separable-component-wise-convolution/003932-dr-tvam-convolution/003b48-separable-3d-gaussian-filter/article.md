# Separable 3D Gaussian Filter
Also called: 3D Gaussian Filter via Separable 1D Filters

## Overview
A 3D Gaussian filter implemented as three consecutive 1D filters to leverage separability for interactive performance [S1, unofficial]. It serves as a low-pass filter to reduce high-frequency elements (noise) in volumetric data, often used to compute spatial importance maps for volume illustration [S1, unofficial]. It sets itself apart from non-separable 3D convolutions by reducing computational complexity, allowing for interactive rates on CUDA hardware [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D scalar volume $V$ and a standard deviation $\sigma$.
- Output: A filtered 3D volume $V_{filtered}$ and optionally the difference volume $V_{diff} = V_{filtered} - V$.

**Data Structures:**
- The volume is stored as a 3D array of scalars.
- The 1D Gaussian kernel $G(x)$ is computed for a range of indices determined by $\sigma$.

**Formulas:**
The 1D Gaussian kernel is defined as [S1, unofficial][S2, unofficial]:
$$ G(x) = \frac{1}{\sqrt{2\pi}\sigma} e^{-\frac{x^2}{2\sigma^2}} $$
where $x$ is the distance from the origin and $\sigma$ is the standard deviation.

**Steps:**
1. Compute the 1D Gaussian kernel values for the specified $\sigma$.
2. Apply 1D Gaussian convolution along the first axis (e.g., X) of the input volume $V$ to produce an intermediate volume $V_x$.
3. Apply 1D Gaussian convolution along the second axis (e.g., Y) of $V_x$ to produce an intermediate volume $V_{xy}$.
4. Apply 1D Gaussian convolution along the third axis (e.g., Z) of $V_{xy}$ to produce the final filtered volume $V_{filtered}$.
5. Subtract the original volume $V$ from the filtered volume $V_{filtered}$ to obtain the difference volume $V_{diff}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def separable_3d_gaussian_filter(volume, sigma):
    """
    Applies a 3D Gaussian filter using separable 1D convolutions.
    
    Args:
        volume: 3D numpy array or tensor of shape (D, H, W).
        sigma: Standard deviation of the Gaussian kernel.
        
    Returns:
        filtered_volume: The result of the 3D Gaussian blur.
        diff_volume: The difference between the original and filtered volume.
    """
    # 1. Compute 1D Gaussian Kernel
    # Determine kernel size based on sigma (e.g., 6*sigma + 1)
    kernel_size = int(6 * sigma + 1)
    half_kernel = kernel_size // 2
    x = np.arange(-half_kernel, half_kernel + 1)
    kernel_1d = (1.0 / (np.sqrt(2 * np.pi) * sigma)) * np.exp(-0.5 * (x / sigma)**2)
    kernel_1d = kernel_1d / np.sum(kernel_1d) # Normalize
    
    # 2. Convolve along first axis (Depth)
    # Note: In practice, this involves sliding the 1D kernel along the depth dimension
    # for each (y, z) coordinate.
    volume_x = convolve_1d_axis(volume, kernel_1d, axis=0)
    
    # 3. Convolve along second axis (Height)
    volume_xy = convolve_1d_axis(volume_x, kernel_1d, axis=1)
    
    # 4. Convolve along third axis (Width)
    volume_filtered = convolve_1d_axis(volume_xy, kernel_1d, axis=2)
    
    # 5. Subtract original volume
    diff_volume = volume_filtered - volume
    
    return volume_filtered, diff_volume

def convolve_1d_axis(volume, kernel, axis):
    """
    Performs 1D convolution along a specific axis of a 3D volume.
    This is a placeholder for the actual convolution logic which would
    iterate over the volume and apply the kernel.
    """
    # Implementation detail: Pad volume, then slide kernel along the specified axis
    # and compute dot product with kernel values.
    # [G] Standard 1D convolution implementation required here.
    pass
```

**Parameters:**
- $\sigma$: Standard deviation of the Gaussian distribution. Controls the spread of the kernel. Typical values are not explicitly given for the main method in [S1, unofficial], but [S2, unofficial] uses $\sigma_s=4$ for a similar spatial filter context.
- Kernel Radius ($R$): Related to $\sigma$. Table 1 in [S1, unofficial] tests different radii.

**Edge Cases and Pitfalls:**
- Boundary conditions: How to handle voxels at the volume boundaries during 1D convolution (e.g., zero-padding, replication, or clamping) is not specified in the sources [S1, unofficial].
- Precision: CUDA implementation is used to obtain interactive rates, implying potential floating-point precision considerations on GPU [S1, unofficial].

## Subtopics

### Bilateral Filter
A non-linear filter used to reduce noise while preserving edges, by weighting pixels based on both spatial proximity and value similarity [S2, unofficial]. Unlike the Gaussian filter which only uses spatial distance, the Bilateral filter also uses value similarity, preserving edges better [S2, unofficial]. It is defined as [S2, unofficial]:
$$ BF(p) = \frac{1}{W_p} \sum_{q \in S} G_{\sigma_s}(\|p - q\|) G_{\sigma_r}(\|I_p - I_q\|) I_q $$
where $W_p$ is the normalization factor [S2, unofficial].
- $\sigma_s$: Spatial standard deviation, controls the decrease in influence based on distance.
- $\sigma_r$: Range standard deviation, controls the decrease in influence based on value difference.
- Typical values: $\sigma_s = 4$, $\sigma_r = 0.2$ [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The separable approach reduces the complexity from $O(N \cdot K^3)$ for a direct 3D convolution to $O(N \cdot 3K)$, where $N$ is the number of voxels and $K$ is the kernel size [S1, unofficial].
- **Performance:** Interactive rates are achieved using a CUDA implementation [S1, unofficial]. Table 1 in [S1, unofficial] provides specific timings for a CT body (256 × 256 × 415), CT head (512 × 512 × 297), and salmon (336 × 173 × 511) with different radii. For example, the total time for the CT body with a specific radius is 617.24 ms [S1, unofficial].
- **Strengths:** Interactive performance due to separability; effective noise reduction [S1, unofficial].
- **Limitations:** Blurs edges compared to non-linear filters like the Bilateral filter [S2, unofficial].

## Background
The method is part of a pipeline for interactive volume illustration, where it is used to compute a spatial importance map to modulate color and opacity assigned by transfer functions [S1, unofficial]. It was presented in the context of Computer Graphics, specifically Transfer Functions and Volume Rendering [S1, unofficial].

## Sources
[S1] Interactive Volume Illustration Using Intensity Filtering, https://doi.org/10.2312/compaesth/compaesth10/051-058, fetched 2026-10-26 (unofficial)
[S2] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
