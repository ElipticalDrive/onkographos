# Fast tri-cubic interpolation

Also called: GPU-accelerated tri-cubic B-spline interpolation, separable tri-cubic resampling via linear texture filtering

## Overview
Fast tri-cubic interpolation is a GPU-based higher-order interpolation method that reconstructs a continuous signal from discrete volume samples using a tri-cubic B-spline filter [S1][S3, unofficial]. It solves the problem of high computational cost associated with evaluating the 64-term convolution sum required for standard tri-cubic interpolation by reducing the number of texture fetches to eight [S1][S3, unofficial]. It sets itself apart from its siblings, such as [Trilinear Interpolation](1531-trilinear-interpolation.md) and [Bilinear Interpolation](858-bilinear-interpolation.md), by providing higher-order continuity and reduced aliasing, while distinguishing itself from [Frequency Domain Volume Rendering with Sinc Resampling](826-frequency-domain-volume-rendering-with-sinc-resampling.md) by operating directly in the spatial domain using separable tensor-product B-splines [S1][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume texture $V$ containing discrete samples $f_{i,j,k}$.
*   **Input:** A query position $\mathbf{p} = (x, y, z)$ in texture coordinates.
*   **Input:** A pre-computed lookup texture $L$ storing weight functions $g_i$ and offset functions $h_i$ [S1].
*   **Input:** A kernel storage texture $K$, a 1D floating-point RGBA texture where each channel stores one tile of the discretized reconstruction kernel [S2, S3, unofficial].
*   **Output:** The interpolated scalar value $V(\mathbf{p})$.

**Data Structures:**
1.  **Lookup Texture ($L$):** A 1D texture storing the pre-computed functions $g_0(x), h_0(x), g_1(x), h_1(x)$ [S1]. These functions allow the reduction of four texture fetches to two per dimension [S1].
2.  **Kernel Storage Texture ($K$):** A 1D floating-point RGBA texture [S2, S3, unofficial]. For a kernel of width 4, the kernel is divided into 4 tiles. Each channel (R, G, B, A) stores one tile [S2, S3, unofficial]. A single texture fetch at position $X$ returns 4 weight values [S2, S3, unofficial].

**Method Steps:**

1.  **Coordinate Transformation:** Convert the query position $\mathbf{p}$ to the local coordinate system of the volume data [S1]. Let $x, y, z$ be the texture coordinates.

2.  **1D Interpolation (X-dimension):**
    The standard tri-cubic convolution sum for one dimension is:
    $$ f(x) = w_0(x)f_{i-1} + w_1(x)f_i + w_2(x)f_{i+1} + w_3(x)f_{i+2} \quad (6) $$
    where $w_k(x)$ are the cubic B-spline weights [S1]. To reduce fetches, this is rewritten using linear texture filtering capabilities [S1]:
    $$ f(x) = g_0(x)f_{[x]-h_0(x)} + g_1(x)f_{[x]+h_1(x)} \quad (7) $$
    where $[x]$ is the integer part of $x$, and:
    $$ g_0(x) = w_0(x) + w_1(x) $$
    $$ h_0(x) = 1 - \frac{w_1(x)}{w_0(x) + w_1(x)} $$
    $$ g_1(x) = w_2(x) + w_3(x) $$
    $$ h_1(x) = 1 + \frac{w_3(x)}{w_2(x) + w_3(x)} \quad (8, 9) $$
    The functions $g_i$ and $h_i$ are pre-computed and stored in the lookup texture $L$ [S1].
    
    *Pseudocode for 1D step:*
    ```python
    # Get weights and offsets from lookup texture L at coordinate x
    # L returns (g0, h0, g1, h1)
    g0, h0, g1, h1 = texture_fetch(L, x)
    
    # Calculate fetch coordinates
    # Note: [x] is integer part. The texture fetches use linear interpolation internally.
    idx = floor(x)
    coord1 = idx - h0
    coord2 = idx + h1
    
    # Fetch two samples using linear interpolation
    val1 = texture_linear(V, coord1)
    val2 = texture_linear(V, coord2)
    
    # Weighted sum
    result_x = g0 * val1 + g1 * val2
    ```

3.  **3D Extension via Separability:**
    Due to the separability of tensor-product B-splines, the 3D filter with 64 summands can be evaluated using just eight tri-linear texture fetches [S1]. This is achieved by applying the 1D interpolation sequentially along the X, Y, and Z axes [S1].

4.  **Kernel-Based Resampling (Alternative/Extended Approach):**
    For implementations using explicit kernel tiles (e.g., in Frequency Domain Volume Rendering contexts), the filtering is divided into four passes [S2, S3, unofficial].
    *   Each pass computes the contribution of 16 neighboring samples (4 neighbors $\times$ 4 channels/tiles) [S2, S3, unofficial].
    *   The kernel tiles are stored in the 1D RGBA texture $K$ [S2, S3, unofficial].
    *   Intermediate results are summed in a separate blending pass after the four intermediate passes because hardware does not support blending with floating-point precision during intermediate steps [S2, S3, unofficial].

5.  **Partial Derivatives (Optional):**
    To compute partial derivatives, the functions $g_i$ and $h_i$ are computed using the appropriate derivatives of $w_i$ [S1].

**Parameters:**
*   **Kernel Width:** 4 samples [S3, unofficial].
*   **Texture Format:** 1D floating-point RGBA texture [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Hardware Precision:** Current hardware does not support blending with floating-point precision [S2, S3, unofficial]. Intermediate sums cannot be blended directly; they must be summed in a separate pass [S2, S3, unofficial].
*   **Fragment Program Length:** ATI GPU limitations require a multi-pass approach (4 passes) due to fragment program length limits [S3, unofficial]. NVIDIA supports longer fragment programs, allowing the multi-pass approach to be folded into a single pass [S3, unofficial].
*   **Boundary Conditions:** The weights $w_k(x)$ are periodic in the sample positions of the input texture [S1]. Boundary handling must respect this periodicity or clamp appropriately if non-periodic boundaries are required [G].

## Subtopics

### [Adaptive Filter Size for Rendering Time Reduction](861-adaptive-filter-size-for-rendering-time-reduction.md)
A technique to reduce rendering time by adaptively selecting smaller interpolation filters for low-energy frequency components, maintaining image quality within a user-defined tolerance. It sets itself apart from standard interpolation schemes by dynamically switching between a high-quality large filter and a low-cost small filter based on the spectral energy of the input data, rather than using a fixed kernel for all samples. This method is specifically designed for frequency-domain volume rendering, building on [Fast tri-cubic interpolation](848-fast-tri-cubic-interpolation.md) by optimizing the convolution cost during slice extraction.

### [Cubic Interpolation (Mitchell-Netravali)](2073-cubic-interpolation-mitchell-netravali.md)
A kernel-based resampling method that reconstructs a scalar value from a volume using a piecewise cubic filter defined by parameters $B$ and $C$. It distinguishes itself from its siblings by providing $C^0$ and $C^1$ continuity to prevent high-frequency leakage artifacts, while allowing tunable trade-offs between blurring and ringing.

### [Prefiltered Reconstruction](11114-prefiltered-reconstruction.md)
A volume rendering technique that improves the approximation order of non-interpolating filters, such as the cubic B-spline, by convolving input samples with a discrete prefilter before applying the continuous reconstruction kernel. It solves the problem of severe oversmoothing inherent in standard approximating filters, which can disconnect fine anatomical structures like blood vessels in CT scans. Unlike standard cubic B-spline interpolation, which acts as a low-pass filter that blurs high-frequency details, this method allows the cubic B-spline to act as an interpolator with $C^2$ continuity and 4th-order approximation accuracy (4-EF). It sets itself apart from its parent [Fast tri-cubic interpolation](848-fast-tri-cubic-interpolation.md) by explicitly handling the coefficient generation step to correct for the non-interpolating nature of the basis function, and from its sibling [Cubic Interpolation (Mitchell-Netravali)](2073-cubic-interpolation-mitchell-netravali.md) by targeting the specific frequency response characteristics of B-splines rather than general cubic kernels.

### [Unprefiltered Cubic B-Spline Reconstruction](11120-unprefiltered-cubic-b-spline-reconstruction.md)
An approximating reconstruction filter used in volume rendering to reconstruct a continuous signal from discrete voxel samples without prior prefiltering. It is classified as a 2-EF (Error Function of order 2) reconstruction, meaning it is quasi-interpolating of order 2. Unlike interpolating methods such as [Cubic Interpolation (Mitchell-Netravali)](2073-cubic-interpolation-mitchell-netravali.md), it does not pass through the original sample values; instead, it smooths the data, making it particularly suitable for reconstructing signals corrupted by noise rather than for high-fidelity detail reconstruction.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, with a constant factor significantly reduced compared to naive tri-cubic interpolation due to the reduction from 64 fetches to 8 fetches per dimension (or 8 total in 3D via separability) [S1].
*   **Memory Complexity:** Requires additional storage for the lookup texture $L$ and kernel texture $K$ [S1][S2, S3, unofficial]. The kernel texture is a 1D RGBA texture, which is memory efficient [S2, S3, unofficial].
*   **Measured Performance:** Performance is highly dependent on GPU architecture. NVIDIA GPUs can fold the multi-pass approach into a single pass due to longer fragment program support, while ATI GPUs require 4 passes [S3, unofficial].
*   **Strengths:** High-quality reconstruction with reduced aliasing compared to linear interpolation; efficient on GPUs due to texture filtering hardware utilization [S1][S3, unofficial].
*   **Limitations:** Requires floating-point texture support; hardware blending precision limitations necessitate multi-pass approaches on some GPUs [S2, S3, unofficial].
*   **When to Use:** When high-quality volume rendering is required and GPU resources allow for floating-point textures and multi-pass rendering [S1][S3, unofficial].
*   **When Not to Use:** On hardware without floating-point texture support or when strict single-pass constraints exist on ATI-like architectures [S3, unofficial].

## Background
Fast tri-cubic interpolation was introduced as a technique to accelerate higher-order interpolation on GPUs by leveraging linear texture filtering [S1]. It was developed in the context of real-time ray-casting and advanced shading of discrete isosurfaces, and level-of-detail volume rendering [S1]. The method exploits the separability of tensor-product B-splines to reduce the computational cost of tri-cubic convolution [S1]. It has been used in frequency domain volume rendering pipelines to perform efficient resampling of data from 3D textures onto slices [S2, S3, unofficial].

## Sources
[S1] Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces, https://cgl.ethz.ch/people/archive/siggc/publications/eg05.pdf, fetched 2026-10-26
[S2] GPU-based Frequency Domain Volume Rendering - TU Wien, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-GPU/Viola-2004-GPU-Paper.pdf, fetched 2026-10-27 (unofficial)
[S3] Hardware-Accelerated Frequency Domain Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/viola-2004-har2/viola-2004-har2-Paper.pdf, fetched 2026-10-27 (unofficial)
[S4] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
