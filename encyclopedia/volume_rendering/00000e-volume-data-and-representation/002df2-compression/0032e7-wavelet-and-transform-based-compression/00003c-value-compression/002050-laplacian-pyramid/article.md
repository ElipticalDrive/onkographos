# Laplacian Pyramid

Also called: Burt and Adelson pyramid

## Overview
The Laplacian Pyramid is a multiresolution decomposition technique that represents a volume by a coarse base layer and a series of residual difference layers (Laplacian levels) [S1, unofficial][S3]. It solves the problem of high inter-pixel correlation in volumetric data by decorrelating the signal, concentrating values around zero to reduce entropy, and enabling efficient compression via quantization and entropy coding [S1, unofficial]. It is distinguished from wavelet-based methods like Triadic frequency band decomposition by its use of non-orthogonal, redundant filter banks that guarantee perfect reconstruction through simple addition, albeit at the cost of oversampling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume data array $V_0$ of dimensions $N_x \times N_y \times N_z$.
*   **Output:** A set of $L+1$ arrays: the coarsest Gaussian level $g_L$ and $L$ Laplacian levels $l_0, l_1, \dots, l_{L-1}$.

**Data Structures:**
*   **Gaussian Pyramid Levels ($g_i$):** Stored as 3D arrays. Level $i$ has dimensions $\lceil N_x/2^i \rceil \times \lceil N_y/2^i \rceil \times \lceil N_z/2^i \rceil$.
*   **Laplacian Levels ($l_i$):** Stored as 3D arrays. Level $i$ has the same dimensions as the input volume at that stage of decomposition (specifically, $l_i$ has the same dimensions as $g_i$ before downsampling, or equivalently, the dimensions of the volume at resolution $2^i$). Note: In the standard Burt-Adelson scheme, $l_i$ is stored at the resolution of the parent level $g_i$ (full resolution relative to $g_{i+1}$).

**Parameters:**
*   **Filter Kernel:** A 3D Gaussian low-pass filter. The source specifies a $5 \times 5 \times 5$ kernel [S1, unofficial].
*   **Separability:** The 3D kernel is separable into three 1D kernels [S1, unofficial].
*   **Number of Levels ($L$):** Typically 5 or 6 [S1, unofficial].
*   **Quantization:** Uniform quantization is used for the Laplacian levels [S1, unofficial].

**Algorithm Steps:**

1.  **Define 1D Low-Pass Filter:**
    Let $w(n)$ be a 1D symmetric, real-valued filter kernel with normalization $\sum_n w(n) = 1$. The source notes the equal contribution constraint $\sum_n w(2n) = \sum_n w(2n+1)$ [S1, unofficial]. For a $5$-tap kernel, indices typically range from $-2$ to $2$.

2.  **Decomposition (Forward Transform):**
    Initialize $g_0 = V_0$.
    For $i = 0$ to $L-1$:
    a.  **Reduce (Downsample):** Compute the next Gaussian level $g_{i+1}$ by convolving $g_i$ with the 3D Gaussian kernel and downsampling by a factor of 2 in each dimension.
        $$g_{i+1}(x, y, z) = \sum_{m=-2}^{2} \sum_{n=-2}^{2} \sum_{p=-2}^{2} w(m) w(n) w(p) \cdot g_i(2x+m, 2y+n, 2z+p)$$
        *Note: Boundary conditions for indices outside $[0, N-1]$ must be handled (e.g., clamping or reflection) [G].*
    b.  **Expand (Upsample):** Compute the predicted version of $g_i$ from $g_{i+1}$, denoted $\bar{g}_i$. This involves upsampling $g_{i+1}$ by inserting zeros and convolving with the same kernel.
        $$\bar{g}_i(x, y, z) = \sum_{m=-2}^{2} \sum_{n=-2}^{2} \sum_{p=-2}^{2} w(m) w(n) w(p) \cdot g_{i+1}(\lfloor \frac{x-m}{2} \rfloor, \lfloor \frac{y-n}{2} \rfloor, \lfloor \frac{z-p}{2} \rfloor)$$
        *Note: The source defines the expand operation as reversing the decimation [S1, unofficial]. Specifically, for even indices in the expanded grid, it maps back to the coarse grid.*
    c.  **Laplacian Level Calculation:** Compute the residual $l_i$ by subtracting the predicted volume from the current Gaussian level.
        $$l_i = g_i - \bar{g}_i$$
    d.  **Store:** Save $l_i$.

3.  **Final Level:**
    After $L$ iterations, store the coarsest Gaussian level $g_L$.

4.  **Compression (Quantization & Encoding):**
    *   Quantize each $l_i$ using uniform quantization [S1, unofficial].
    *   Encode $g_L$ and the quantized $l_i$ arrays using entropy coding (e.g., variable length coding) [S1, unofficial][S3].

5.  **Reconstruction (Decompression):**
    *   Decode $g_L$ and the quantized $l_i$ arrays.
    *   Initialize $r_{L-1} = l_{L-1} + \text{EXPAND}(g_L)$.
    *   For $i = L-2$ down to $0$:
        $$r_i = l_i + \text{EXPAND}(r_{i+1})$$
    *   The reconstructed volume is $V_0' = r_0$.
    *   *Note: If quantization was used, $V_0'$ is an approximation of $V_0$.*

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def laplacian_pyramid_decompose(volume, num_levels=5, kernel_1d=None):
    """
    Decomposes a 3D volume into a Laplacian pyramid.
    
    Args:
        volume: 3D numpy array (Nx, Ny, Nz)
        num_levels: Integer L, number of decomposition steps
        kernel_1d: 1D numpy array representing the low-pass filter w(n)
                   Must be symmetric and sum to 1.
    
    Returns:
        gaussian_levels: List of 3D arrays [g_0, g_1, ..., g_L]
        laplacian_levels: List of 3D arrays [l_0, l_1, ..., l_{L-1}]
    """
    if kernel_1d is None:
        # Default 5-tap Gaussian-like kernel for demonstration
        # Source specifies 5x5x5 separable kernel [S1, unofficial]
        kernel_1d = np.array([0.0625, 0.25, 0.375, 0.25, 0.0625])
    
    # Pad volume to handle boundary conditions during convolution
    # Source does not specify boundary handling, so we use zero-padding or reflection [G]
    # Assuming the expand/reduce operations handle indices via floor division as per source
    # We will implement the reduce and expand functions directly based on source formulas
    
    gaussian_levels = []
    laplacian_levels = []
    
    current_vol = volume.copy()
    gaussian_levels.append(current_vol)
    
    for i in range(num_levels):
        # Step 1: Reduce (Downsample)
        # Convolve with 3D kernel (separable: 3 passes of 1D)
        blurred = convolve_3d_separable(current_vol, kernel_1d)
        
        # Downsample by 2 in each dimension
        # Indices: 2x, 2y, 2z
        next_gaussian = blurred[::2, ::2, ::2]
        gaussian_levels.append(next_gaussian)
        
        # Step 2: Expand (Upsample)
        # Upsample next_gaussian by inserting zeros
        expanded_shape = current_vol.shape
        expanded = np.zeros(expanded_shape, dtype=current_vol.dtype)
        
        # Map coarse pixels to fine grid
        # Source: g_bar_0(i,j) = sum w(m,n) g_1((i-m)/2, (j-n)/2)
        # We iterate over the coarse grid and add to the fine grid
        kh = len(kernel_1d) // 2
        for z in range(next_gaussian.shape[0]):
            for y in range(next_gaussian.shape[1]):
                for x in range(next_gaussian.shape[2]):
                    val = next_gaussian[z, y, x]
                    # Center of the kernel in the fine grid corresponds to index 2*z, 2*y, 2*x
                    # The kernel is centered at 0. So we add kernel weights to neighbors
                    for dz in range(-kh, kh + 1):
                        for dy in range(-kh, kh + 1):
                            for dx in range(-kh, kh + 1):
                                fz = 2 * z + dz
                                fy = 2 * y + dy
                                fx = 2 * x + dx
                                
                                # Check bounds
                                if 0 <= fz < expanded_shape[0] and \
                                   0 <= fy < expanded_shape[1] and \
                                   0 <= fx < expanded_shape[2]:
                                    # Kernel index mapping: w(n) where n is offset
                                    # If kernel is symmetric around 0, w(dz)
                                    expanded[fz, fy, fx] += val * kernel_1d[dz + kh] * kernel_1d[dy + kh] * kernel_1d[dx + kh]
        
        # Step 3: Laplacian Level
        laplacian_level = current_vol - expanded
        laplacian_levels.append(laplacian_level)
        
        # Prepare for next iteration
        current_vol = next_gaussian
        
    return gaussian_levels, laplacian_levels

def laplacian_pyramid_reconstruct(gaussian_levels, laplacian_levels):
    """
    Reconstructs the volume from the Laplacian pyramid.
    
    Args:
        gaussian_levels: List of 3D arrays [g_0, ..., g_L]
        laplacian_levels: List of 3D arrays [l_0, ..., l_{L-1}]
        
    Returns:
        reconstructed_volume: 3D numpy array
    """
    L = len(gaussian_levels) - 1
    
    # Start from the coarsest level
    # r_{L-1} = l_{L-1} + EXPAND(g_L)
    # Note: gaussian_levels[L] is g_L
    # laplacian_levels[L-1] is l_{L-1}
    
    # We need to reconstruct from L-1 down to 0
    # Let's work backwards
    
    # Initialize r with the last laplacian level
    # But we need to expand g_L first
    g_L = gaussian_levels[L]
    
    # Expand g_L to the size of l_{L-1} (which is size of g_{L-1})
    # We reuse the expand logic from decomposition, but here we just need to upsample
    # The expand function is defined as: sum w(m,n) g_1((i-m)/2...)
    
    # Helper to expand a volume to double size
    def expand_volume(vol, kernel_1d):
        kh = len(kernel_1d) // 2
        new_shape = (vol.shape[0] * 2, vol.shape[1] * 2, vol.shape[2] * 2)
        expanded = np.zeros(new_shape, dtype=vol.dtype)
        for z in range(vol.shape[0]):
            for y in range(vol.shape[1]):
                for x in range(vol.shape[2]):
                    val = vol[z, y, x]
                    for dz in range(-kh, kh + 1):
                        for dy in range(-kh, kh + 1):
                            for dx in range(-kh, kh + 1):
                                fz = 2 * z + dz
                                fy = 2 * y + dy
                                fx = 2 * x + dx
                                if 0 <= fz < new_shape[0] and \
                                   0 <= fy < new_shape[1] and \
                                   0 <= fx < new_shape[2]:
                                    expanded[fz, fy, fx] += val * kernel_1d[dz + kh] * kernel_1d[dy + kh] * kernel_1d[dx + kh]
        return expanded

    # Default kernel for reconstruction
    kernel_1d = np.array([0.0625, 0.25, 0.375, 0.25, 0.0625])
    
    # Reconstruct r_{L-1}
    r = laplacian_levels[L-1] + expand_volume(g_L, kernel_1d)
    
    # Iterate downwards
    for i in range(L-2, -1, -1):
        # r_i = l_i + EXPAND(r_{i+1})
        # r is currently r_{i+1}
        # l_i is laplacian_levels[i]
        r = laplacian_levels[i] + expand_volume(r, kernel_1d)
        
    return r

def convolve_3d_separable(volume, kernel_1d):
    """
    Convolve a 3D volume with a separable 3D kernel.
    """
    # Pass 1: X dimension
    kh = len(kernel_1d) // 2
    temp = np.zeros_like(volume)
    for z in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for x in range(volume.shape[2]):
                val = 0.0
                for k in range(-kh, kh + 1):
                    if 0 <= x + k < volume.shape[2]:
                        val += volume[z, y, x + k] * kernel_1d[k + kh]
                temp[z, y, x] = val
    volume = temp
    
    # Pass 2: Y dimension
    temp = np.zeros_like(volume)
    for z in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for x in range(volume.shape[2]):
                val = 0.0
                for k in range(-kh, kh + 1):
                    if 0 <= y + k < volume.shape[1]:
                        val += volume[z, y + k, x] * kernel_1d[k + kh]
                temp[z, y, x] = val
    volume = temp
    
    # Pass 3: Z dimension
    temp = np.zeros_like(volume)
    for z in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for x in range(volume.shape[2]):
                val = 0.0
                for k in range(-kh, kh + 1):
                    if 0 <= z + k < volume.shape[0]:
                        val += volume[z + k, y, x] * kernel_1d[k + kh]
                temp[z, y, x] = val
    return temp
```

## Operating characteristics
*   **Time Complexity:** Decomposition and reconstruction are $O(N)$ where $N$ is the number of voxels, assuming fixed kernel size. The separable 3D filter reduces the constant factor significantly; the source notes a speedup of roughly 8 in reconstruction due to separability [S1, unofficial].
*   **Memory Complexity:** The method is redundant. The total storage is $\sum_{i=0}^{L-1} V_i + V_L$, where $V_i$ is the volume at level $i$. Since $V_i \approx 8 V_{i+1}$, the total size is approximately $1 + 1/8 + 1/64 + \dots \approx 1.14$ times the original volume size before quantization. This is the "oversampling" drawback mentioned in the source [S1, unofficial].
*   **Performance:** A compression ratio of 10:1 was achieved with uniform quantization on CT volume data, with a root mean square error (RMSE) of 1.3 [S1, unofficial].
*   **Strengths:** Perfect reconstruction (without quantization), simple addition-based reconstruction, effective decorrelation of spatial data [S1, unofficial].
*   **Limitations:** Redundant representation (oversampling) increases memory usage compared to orthogonal wavelets. Computational cost of filtering can be high without separability [S1, unofficial].

## Background
The Laplacian Pyramid was introduced by Burt and Adelson [S1, unofficial][S3]. It is an early multiresolution decomposition technique, predating the rapid development of wavelet transformations in the late 1980s [S1, unofficial]. It is closely related to multiresolution analysis and orthonormal wavelet bases [S1, unofficial]. It is used in volume rendering for adaptive rendering, efficient coding, coarse-to-fine progressive rendering, and error bounds checking [S1, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
