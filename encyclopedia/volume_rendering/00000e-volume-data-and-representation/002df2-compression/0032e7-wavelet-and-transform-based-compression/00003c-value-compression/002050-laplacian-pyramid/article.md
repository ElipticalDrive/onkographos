# Laplacian Pyramid

Also called: Burt and Adelson pyramid, 拉普拉斯金字塔

## Overview
The Laplacian Pyramid is a multiresolution decomposition technique that represents a volume by a set of detail residuals (the Laplacian pyramid) and a single low-resolution approximation (the Gaussian pyramid root) [S1, unofficial][S3]. It solves the problem of inter-pixel correlation by predicting the original data from a coarser version and encoding the difference, which concentrates values around zero and reduces entropy [S1, unofficial]. Unlike orthogonal wavelet transforms, it uses non-orthogonal, redundant filters, resulting in an oversampled representation where the total data size exceeds the original, but allows for perfect reconstruction and progressive rendering [S1, unofficial].

## How it works
**Inputs:** A 3D volume $V$ of dimensions $N_x \times N_y \times N_z$.
**Outputs:** A Gaussian pyramid root $g_L$ and a set of Laplacian detail levels $l_0, l_1, \dots, l_{L-1}$.

**Data Structures:**
The decomposition produces a hierarchy of volumes.
1.  **Gaussian Pyramid Levels ($g_j$):** A sequence of low-pass filtered volumes. $g_0 = V$. $g_j$ has dimensions $\lceil N_x/2^j \rceil \times \lceil N_y/2^j \rceil \times \lceil N_z/2^j \rceil$.
2.  **Laplacian Pyramid Levels ($l_j$):** A sequence of detail volumes. $l_j$ has the same dimensions as $g_j$.

**Parameters:**
*   **Filter Kernel ($w$):** A separable 1D low-pass filter. The sources specify a 5-tap symmetric kernel for 1D, which is extended to a $5 \times 5 \times 5$ 3D kernel via separability [S1, unofficial].
*   **Levels ($L$):** The number of decomposition steps. Typical values are $L=5$ or $6$ [S1, unofficial].

**Steps:**

1.  **Define 1D Filter:**
    Let $w(n)$ be a real, symmetric, normalized 1D filter kernel with support $[-2, 2]$ (5 taps).
    $$ \sum_n w(n) = 1 $$
    $$ w(n) = w(-n) $$
    The equal contribution constraint requires:
    $$ \sum_n w(2n) = \sum_n w(2n+1) $$
    [S1, unofficial]

2.  **REDUCE (Downsampling):**
    To generate the next Gaussian level $g_{j+1}$ from $g_j$:
    First, convolve $g_j$ with the 3D filter $w$. Since the filter is separable, this is done in three passes along the x, y, and z axes [S1, unofficial].
    Then, decimate by taking every second voxel along each axis.
    For a voxel at index $(i, j, k)$ in $g_{j+1}$:
    $$ g_{j+1}(i, j, k) = \sum_{m=-2}^{2} \sum_{n=-2}^{2} \sum_{p=-2}^{2} w(m) w(n) w(p) \cdot g_j(2i+m, 2j+n, 2k+p) $$
    [S1, unofficial]

3.  **EXPAND (Upsampling):**
    To generate the predicted version $\bar{g}_j$ from $g_{j+1}$ (same dimensions as $g_j$):
    Insert zeros between voxels of $g_{j+1}$ and convolve with the same filter $w$.
    $$ \bar{g}_j(i, j, k) = \sum_{m=-2}^{2} \sum_{n=-2}^{2} \sum_{p=-2}^{2} w(m) w(n) w(p) \cdot g_{j+1}\left(\frac{i-m}{2}, \frac{j-n}{2}, \frac{k-p}{2}\right) $$
    Note: The indices $(i-m)/2$, etc., are valid only when the numerator is even; otherwise, the term is zero (due to the inserted zeros in upsampling) [S1, unofficial].

4.  **Compute Laplacian Detail ($l_j$):**
    Subtract the expanded lower level from the current level:
    $$ l_j = g_j - \bar{g}_j $$
    [S1, unofficial]

5.  **Iterate:**
    Repeat steps 2-4 for $j = 0$ to $L-1$.
    The final Gaussian level $g_L$ is the root.
    The set $\{l_0, l_1, \dots, l_{L-1}, g_L\}$ constitutes the Laplacian Pyramid representation.
    [S1, unofficial]

6.  **Reconstruction:**
    To reconstruct $g_0$ from $\{l_0, \dots, l_{L-1}, g_L\}$:
    Start with $r_L = g_L$.
    For $i = L-1$ down to $0$:
    $$ r_i = l_i + \text{EXPAND}(r_{i+1}) $$
    The result $r_0$ is the reconstructed volume $g_0$.
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def laplacian_pyramid_decompose(volume, levels=5):
    """
    Decomposes a 3D volume into a Laplacian Pyramid.
    volume: 3D numpy array (Nx, Ny, Nz)
    levels: integer L
    Returns: list of detail volumes [l0, l1, ..., l(L-1)], and the root gL
    """
    # Define 1D filter kernel (example: Gaussian-like 5-tap)
    # Must satisfy sum(w) = 1 and symmetry
    w = np.array([0.05, 0.25, 0.4, 0.25, 0.05]) 
    
    current_level = volume
    laplacian_levels = []
    
    for j in range(levels):
        # 1. REDUCE: Convolve with 3D separable filter and downsample
        # Apply 1D filter along X
        reduced_x = convolve_1d(current_level, w, axis=0)
        # Downsample X
        reduced_x = reduced_x[::2, :, :]
        
        # Apply 1D filter along Y
        reduced_y = convolve_1d(reduced_x, w, axis=1)
        # Downsample Y
        reduced_y = reduced_y[:, ::2, :]
        
        # Apply 1D filter along Z
        reduced_z = convolve_1d(reduced_y, w, axis=2)
        # Downsample Z
        next_gaussian = reduced_z[:, :, ::2]
        
        # 2. EXPAND: Upsample next_gaussian and convolve
        # Upsample by inserting zeros
        expanded_shape = current_level.shape
        expanded = np.zeros(expanded_shape, dtype=current_level.dtype)
        expanded[::2, ::2, ::2] = next_gaussian
        
        # Convolve expanded with 3D separable filter
        # Apply 1D filter along X
        expanded_x = convolve_1d(expanded, w, axis=0)
        # Apply 1D filter along Y
        expanded_y = convolve_1d(expanded_x, w, axis=1)
        # Apply 1D filter along Z
        predicted = convolve_1d(expanded_y, w, axis=2)
        
        # 3. Compute Detail
        detail = current_level - predicted
        laplacian_levels.append(detail)
        
        current_level = next_gaussian
        
    return laplacian_levels, current_level

def convolve_1d(volume, kernel, axis):
    """
    Convolves a volume with a 1D kernel along a specific axis.
    Assumes zero-padding at boundaries.
    """
    # Implementation of 1D convolution
    # This is a placeholder for the actual convolution logic
    # In a real implementation, use scipy.signal.convolve or manual loop
    pass

def laplacian_pyramid_reconstruct(laplacian_levels, root):
    """
    Reconstructs the volume from Laplacian Pyramid levels.
    laplacian_levels: list [l0, l1, ..., l(L-1)]
    root: gL
    Returns: reconstructed volume
    """
    current = root
    # Iterate backwards from L-1 to 0
    for i in range(len(laplacian_levels) - 1, -1, -1):
        detail = laplacian_levels[i]
        
        # Expand current (which is r_{i+1})
        # Upsample by inserting zeros
        expanded_shape = detail.shape
        expanded = np.zeros(expanded_shape, dtype=current.dtype)
        expanded[::2, ::2, ::2] = current
        
        # Convolve with 3D separable filter
        expanded_x = convolve_1d(expanded, w, axis=0)
        expanded_y = convolve_1d(expanded_x, w, axis=1)
        predicted = convolve_1d(expanded_y, w, axis=2)
        
        # Add detail
        current = detail + predicted
        
    return current
```

**Edge Cases and Pitfalls:**
*   **Oversampling:** The representation is redundant. The sum of the sizes of all levels $g_j$ and $l_j$ is larger than the original volume size [S1, unofficial].
*   **Boundary Conditions:** The convolution and upsampling operations require handling volume boundaries. Zero-padding is a common choice, but may introduce artifacts at the edges [S1, unofficial].
*   **Precision:** Floating-point errors during the subtract/add reconstruction steps can lead to small reconstruction errors if not handled with sufficient precision [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Decomposition and reconstruction are $O(N \log N)$ or $O(N)$ depending on implementation, where $N$ is the number of voxels, due to the hierarchical structure and separable filters. The reconstruction speedup from separability is roughly a factor of 8 compared to a non-separable 3D convolution [S1, unofficial].
*   **Memory Complexity:** The storage requirement is higher than the original volume due to oversampling. The total size is $\sum_{j=0}^{L} (N/2^j)^3 + (N/2^L)^3$. For $L=5$, this is approximately $1.33$ times the original size before quantization [S1, unofficial].
*   **Compression Ratio:** A compression ratio of 10:1 was achieved with uniform quantization on CT volume data, with a root mean square error (RMSE) of 1.3 [S1, unofficial]. Other sources cite a compression ratio of about 10 times [S4, unofficial].
*   **Strengths:** Removes inter-pixel correlation, concentrating values around zero, which reduces entropy and allows for efficient entropy coding (e.g., variable length coding) [S1, unofficial]. Supports progressive rendering (coarse to fine) [S1, unofficial].
*   **Limitations:** Oversampling increases memory usage. Non-orthogonal nature means it is not a perfect basis for all signal processing tasks compared to wavelets. Computational cost of filtering can be high without separability [S1, unofficial].

## Background
The Laplacian Pyramid was introduced by Burt and Adelson [3] (cited in [S1, unofficial]) as an early multiresolution decomposition technique before the rapid development of wavelet transforms in the late 1980s [S1, unofficial]. It is closely related to wavelet research and multiresolution analysis [S1, unofficial]. In volume rendering, it has been used for compressing medical image data (e.g., CT volumes) and enabling adaptive, coarse-to-fine rendering [S1, S4, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S4] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
