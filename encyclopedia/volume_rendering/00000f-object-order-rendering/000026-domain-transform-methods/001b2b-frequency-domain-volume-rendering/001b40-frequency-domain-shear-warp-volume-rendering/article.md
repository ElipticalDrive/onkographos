# Frequency Domain Shear-Warp Volume Rendering

Also called: FDVR, Fourier Volume Rendering, Resampling in the Frequency Domain

## Overview
Frequency Domain Shear-Warp Volume Rendering is a technique that performs the shear transformation of the shear-warp factorization in the frequency domain using the shifting theorem, allowing for arbitrary, view-independent sampling distances along viewing rays via zero-padding interpolation [S1, unofficial]. Unlike standard shear-warp algorithms where sampling distance is coarse and view-dependent, this method decouples sampling density from the viewing direction by resampling in the frequency domain, thereby avoiding the large spatial zero-padding required by spatial-domain shear methods [S1, unofficial].

## How it works
The method builds on [Frequency Domain Volume Rendering](952-frequency-domain-volume-rendering.md) by replacing the spatial shear stage with frequency-domain operations [S1, unofficial].

**Inputs and Outputs**
*   **Input:** Volume dataset $V$ of size $N \times N \times N$ (cubiform, power of two), viewing vector $\vec{v}_{so}$, desired sampling distance $s'$, slice distance $d_k$ [S1, unofficial].
*   **Output:** Rendered image with arbitrary sampling distance $s'$ along viewing rays [S1, unofficial].

**Data Structures**
*   **Volume Data:** 3D array of voxel values [S1, unofficial].
*   **Frequency Domain:** 3D complex array resulting from FFT [S1, unofficial].
*   **Slices:** 2D arrays representing planes perpendicular to the viewing direction [S1, unofficial].

**Numbered Steps**

1.  **Permutation:** Change the storage order of voxels in memory to maximize cache coherency, such that the $(i, j)$ plane is most perpendicular to the viewing direction [S1, unofficial]. The volume starts in the standard object coordinate system, already permuted such that the $(i, j)$ plane is most perpendicular to the viewing direction [S1, unofficial].

2.  **3D Fourier Transform:** Compute the 3D Fourier transform of the permuted volume to reach the frequency domain representation [S1, unofficial].

3.  **Calculate Pre-Resampling Sampling Distance ($s$):** Compute the sampling distance along the viewing rays before resampling using the viewing vector and slice distance [S1, unofficial].
    *   Normalize the viewing vector $\vec{v}_{so} = (v_{so,i}, v_{so,j}, v_{so,k})$ in the $k$ direction by dividing each component by $v_{so,k}$, yielding $\vec{v}'_{so} = (v'_{so,i}, v'_{so,j}, v'_{so,k})$ with $v'_{so,k} = 1.0$ [S1, unofficial].
    *   Calculate $s$ using the formula:
        $$ s = d_k \sqrt{(v'_{so,i})^2 + (v'_{so,j})^2 + (v'_{so,k})^2} $$
        where $s$ is the sampling distance before resampling, $d_k$ is the distance between volume slices along the $k$ axis, and $v'_{so,i}, v'_{so,j}, v'_{so,k}$ are the components of the normalized viewing vector [S1, unofficial].

4.  **Calculate Post-Resampling Slice Count ($K'$):** Determine the number of slices needed after resampling to achieve the desired sampling distance $s'$ using the ratio of desired to current sampling distance and the current slice count $K$ [S1, unofficial].
    *   The relationship is given by:
        $$ \frac{s'}{s} = \frac{K}{K'} $$
    *   Solve for $K'$:
        $$ K' = \frac{sK}{s'} $$
    *   Round $K'$ to the closest integer $K_0 \in \mathbb{N}$ [S1, unofficial].

5.  **Zero-Padding:** Append zeros in the frequency domain along the $k$ direction to increase the number of slices from $K$ to $K_0$, exploiting the packing theorem (zero-padding interpolation property) [S1, unofficial].
    *   The difference between $K_0$ and $K$ is the amount of zero-padding necessary for the selected sampling distance $s_0$ [S1, unofficial].
    *   Note: For $K_0 > 50$, the deviation from the desired sampling distance $s_0$ is below $\pm 1.0\%$ [S1, unofficial].

6.  **Inverse Fourier Transform in $k$:** Apply an inverse Fourier transform in the $k$ direction while keeping the $i$ and $j$ directions in the frequency domain to obtain sheared slices [S1, unofficial]. This stage effectively performs the shear transformation via the shifting theorem in the frequency domain [S1, unofficial].

7.  **Inverse Fourier Transform in $i$ and $j$:** Apply 2D inverse Fourier transforms to the sheared slices to bring the $i$ and $j$ dimensions into the spatial domain, preparing them for compositing [S1, unofficial].

8.  **Compositing:** Perform numeric integration along the viewing rays on the resampled slices. This stage uses the arbitrary sampling distance $s'$ independent of the viewing direction [S1, unofficial].

9.  **Warping:** Transform the intermediate image of the compositing stage into the final image. The warping in this method remains similar to the standard shear-warp algorithm [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def frequency_domain_shear_warp_render(volume, origin, direction, step):
    """
    Renders a ray using Frequency Domain Shear-Warp Volume Rendering.
    Precomputes frequency-domain slices, then composites along the ray.
    """
    N = volume.shape[0]
    
    # Step 1 & 2: Permutation & 3D FFT
    # Permutation aligns (i,j) plane perpendicular to viewing direction.
    # [G] Standard permutation reorders axes so k aligns with the dominant viewing direction component.
    permuted_volume = np.transpose(volume, (2, 0, 1))
    freq_volume = np.fft.fftn(permuted_volume)
    
    # Step 3: Pre-Resampling Sampling Distance (s)
    v_x, v_y, v_z = direction
    v_prime_x = v_x / v_z
    v_prime_y = v_y / v_z
    v_prime_z = 1.0
    s = step * np.sqrt(v_prime_x**2 + v_prime_y**2 + v_prime_z**2)
    
    # Step 4: Post-Resampling Slice Count (K')
    K = N
    K_0_float = (s * K) / step
    K_0_pad = int(round(K_0_float))
    
    # Step 5: Zero-Padding in frequency domain along k
    padded_freq = np.zeros((K_0_pad, N, N), dtype=complex)
    padded_freq[:N, :, :] = freq_volume
    
    # Step 6: Inverse FFT in k
    sheared_slices_freq = np.fft.ifft(padded_freq, axis=0)
    
    # Step 7: Inverse FFT in i and j
    spatial_slices = np.fft.ifft2(sheared_slices_freq, axes=(1, 2))
    
    # Step 8: Compositing along the ray
    # Ray: P(t) = origin + t * direction
    # We sample at intervals of 'step' along the ray.
    # Map ray coordinates to slice indices.
    # [G] Standard shear-warp warping transforms ray coordinates to the permuted space.
    # Here we simulate sampling along the k-axis of the spatial slices.
    num_samples = int(np.ceil(np.linalg.norm(origin) / step))
    samples = []
    T = 1.0
    C = 0.0
    for t in np.arange(0, num_samples * step, step):
        k_idx = int(round(t))
        if 0 <= k_idx < spatial_slices.shape[0]:
            # Sample center of slice (scalar volume)
            val = np.real(spatial_slices[k_idx, N//2, N//2])
            alpha = val * step
            C = C + alpha * T
            T = T * (1.0 - alpha)
            samples.append(C)
        else:
            samples.append(C)
    return samples[:4]
```

**Parameters**
*   **Desired Sampling Distance ($s'$):** The target sampling distance along the viewing rays, which is independent of the view direction [S1, unofficial].
*   **Slice Distance ($d_k$):** The distance between volume slices along the $k$ axis in the object coordinate system [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Spatial Domain Zero-Padding:** If spatial domain zero-padding is too small in shear-based methods, data passes over the border and re-enters from the other side due to periodicity, an error amplified by consecutive shears. This method avoids this by not requiring large spatial domain zero-padding [S1, unofficial].
*   **Rounding Error:** Zero-padding can only be applied in discrete steps, so $K_0$ is rounded to the closest integer. For $K_0 > 50$, the deviation from the desired sampling distance $s_0$ is below $\pm 1.0\%$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** For a cubiform dataset with edge length $N$ (power of two), the computational complexity is $6 \cdot N^3 \log_2 N$, consisting of one 3D FFT ($3 \cdot N^3 \log_2 N$), $N^2$ 1D IFFTs ($N^3 \log_2 N$), and $N^2$ 2D IFFTs ($N^3 \log_2 N$) [S1, unofficial].
*   **Memory Complexity:** The method does not require large spatial domain zero-padding, resulting in significantly lower memory consumption compared to methods like Li et al.'s, which requires memory >5x higher [S1, unofficial].
*   **Performance:** Experimental results show that the presented method outperforms established algorithms in the quality of the produced images [S1, unofficial]. If the data is sampled above the Nyquist rate, the method is capable of perfect reconstruction of the original function [S1, unofficial].
*   **Strengths:** Allows arbitrary, view-independent sampling distances; avoids periodicity errors from spatial shear; high reconstruction quality [S1, unofficial].
*   **Limitations:** Requires FFT computation; rounding of slice count may introduce minor sampling distance deviations (negligible for $K_0 > 50$) [S1, unofficial].

## Background
*   **Origin:** The method was introduced in a paper by researchers from the University of Technology, Austria, and Simon Fraser University, Vancouver, Canada [S1, unofficial].
*   **History:** It is conceptually based on the shear-warp factorization, which consists of permutation, shearing, compositing, and warping stages [S1, unofficial]. The method proposes performing the shear transformation in the frequency domain using the shifting theorem [S1, unofficial].
*   **Usage:** Used for high-quality volume rendering, particularly where view-independent sampling and high reconstruction quality are required [S1, unofficial].

## Sources
[S1] High-Quality Volume Rendering with Resampling in the Frequency Domain, https://doi.org/10.2312/vissym/eurovis05/085-092, fetched 2026-10-26 (unofficial)
