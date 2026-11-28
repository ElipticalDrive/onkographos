# Wavelet Noise
Also called: Cook and DeRose Noise
## Overview
Wavelet noise is an explicit noise function introduced by Cook and DeRose in 2005 that generates values by representing the noise field as a quadratic B-spline surface, achieving near-perfect band-limiting to solve the aliasing and detail loss inherent in weakly band-limited methods like [Perlin Noise 2D](4211-perlin-noise-2d.md) [S1, S3, S4]. It differs from siblings such as [Fractal Brownian Motion (fBm) with Perlin Noise](4253-fractal-brownian-motion-fbm-with-perlin-noise.md) by providing strict spectral control via a weighted sum and supporting solid noise with isotropic filtering [S2, unofficial][S4].
## How it works
**Inputs:** A tile size (period $N$), spatial coordinates $\mathbf{x}$, and dimensionality $d$ [S1, S3, S4].
**Outputs:** A scalar noise value $N(\mathbf{x})$ [S1, S3, S4].

**Data Structures:**
*   **Noise Coefficients Tile ($N$):** A 3D array (for 3D noise) or 2D array (for 2D noise) of size $N \times N \times N$ (or $N \times N$) [S1, S3, S4]. Each element stores a floating-point value representing the coefficient at that grid point [S1, S3, S4]. The space is tiled periodically with this volume [S1, S3, S4].

**Preprocessing Steps:**
1.  **Generate Random Image ($R$):** Create an image $R$ of size $M \times M \times M$ (where $M = 2N$) filled with independent random values uniformly distributed in $[0, 1)$ [S1][S2, unofficial][S3, S4].
2.  **Downsample ($R \downarrow$):** Downsample $R$ by a factor of 2 using an analysis filter derived from the uniform quadratic B-spline basis [S1, S3, S4]. The result is an image $R \downarrow$ of size $N \times N \times N$ [S1, S3, S4].
3.  **Upsample ($R \downarrow \uparrow$):** Upsample $R \downarrow$ by a factor of 2 using a refinement filter derived from the uniform quadratic B-spline basis [S1, S3, S4]. The result is an image $R \downarrow \uparrow$ of size $M \times M \times M$ [S1, S3, S4].
4.  **Compute Coefficients ($N$):** Subtract the upsampled image from the original random image: $N = R - R \downarrow \uparrow$ [S1, S3, S4]. The resulting tile $N$ contains the noise coefficients representing the band-limited part [S1][S2, unofficial][S3, S4].

**Runtime Evaluation:**
1.  **Tiling:** Map the query coordinate $\mathbf{x}$ to the local tile coordinates by taking $\mathbf{x} \mod N$ [S1, S3, S4].
2.  **B-Spline Evaluation:** Evaluate the quadratic B-spline basis function at the local coordinates using the precomputed coefficients $N$ [S1][S2, unofficial][S3, S4]. Let $\mathbf{x}_{local}$ be the fractional position within the tile. The value is computed as a convolution of the coefficients with the quadratic B-spline kernel [S1, S3, S4].

**Formulas:**
The noise function is defined as:
$$ N(\mathbf{x}) = \sum_{\mathbf{k}} n_{\mathbf{k}} B(\mathbf{x} - \mathbf{k}) $$
where:
*   $n_{\mathbf{k}}$ are the precomputed coefficients from the tile $N$ [S1, S3, S4].
*   $B(\mathbf{x})$ is the uniform quadratic B-spline basis function [S1, S3, S4].
*   The sum is over the grid indices $\mathbf{k}$ in the vicinity of $\mathbf{x}$ [S1, S3, S4].

The filters used in downsampling and upsampling correspond to the analysis and refinement coefficients of the uniform quadratic B-spline basis function [S1, S3, S4]. Specifically, the downsampling filter $h$ and upsampling filter $g$ are determined by the wavelet analysis of the quadratic B-spline [S1, S3, S4].

**Parameters:**
*   **Tile Size ($N$):** The period of the noise [S1, S3, S4].
*   **Random Seed:** Determines the initial random image $R$ [S1][S2, unofficial][S3, S4].

**Edge Cases and Pitfalls:**
*   **Sampling 2D from 3D:** Sampling a 3D band-limited noise function on a 2D surface does not result in a band-limited texture, potentially causing aliasing artifacts on the surface despite the 3D function being band-limited [S1][S2, unofficial].
*   **Tiling Artifacts:** The method assumes periodic tiling, leading to repeating patterns or seams when true random accessibility is required [S2, unofficial].
*   **Precision:** The subtraction $R - R \downarrow \uparrow$ may suffer from precision loss if filters introduce significant rounding errors [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_wavelet_noise(tile_size: int, random_seed: int) -> list:
    """Preprocesses to create the noise coefficient tile."""
    M = 2 * tile_size
    R = generate_random_image(M, M, M, random_seed)
    
    # Standard quadratic B-spline analysis coefficients [G]
    analysis_filter = [0.5, 0.0, -0.5]
    R_down = downsample(R, analysis_filter)
    
    # Standard quadratic B-spline refinement coefficients [G]
    refinement_filter = [0.25, 0.75, 0.75, 0.25]
    R_down_up = upsample(R_down, refinement_filter)
    
    N = subtract_images(R, R_down_up)
    return N

def evaluate_wavelet_noise(x: float, y: float, z: float, coefficients: list, tile_size: int) -> float:
    """Evaluates the wavelet noise at point (x, y, z)."""
    x_local = x % tile_size
    y_local = y % tile_size
    z_local = z % tile_size
    return evaluate_quadratic_b_spline(x_local, y_local, z_local, coefficients)

def generate_random_image(width: int, height: int, depth: int, seed: int) -> list:
    """Generates a 3D array of random floats in [0, 1)."""
    import random
    random.seed(seed)
    return [[[random.random() for _ in range(depth)] for _ in range(height)] for _ in range(width)]

def convolve_1d(signal: list, kernel: list) -> list:
    """Applies a 1D convolution to a signal."""
    out = []
    k_len = len(kernel)
    pad = k_len // 2
    padded = [signal[0]] * pad + signal + [signal[-1]] * pad
    for i in range(len(signal)):
        val = 0.0
        for j in range(k_len):
            val += padded[i + j] * kernel[j]
        out.append(val)
    return out

def downsample(image: list, filter_1d: list) -> list:
    """Downsamples a 3D image by a factor of 2 using separable 1D convolution."""
    # Apply filter along axis 0
    temp = [[[convolve_1d(row, filter_1d) for row in slice_] for slice_ in image] for _ in range(len(image[0][0]))]
    # Transpose to axis 1
    temp = [[[temp[k][j][i] for k in range(len(temp))] for j in range(len(temp[0]))] for i in range(len(temp[0][0]))]
    # Apply filter along axis 1
    temp = [[[convolve_1d(row, filter_1d) for row in slice_] for slice_ in temp] for _ in range(len(temp[0][0]))]
    # Transpose to axis 2
    temp = [[[temp[k][j][i] for k in range(len(temp))] for j in range(len(temp[0]))] for i in range(len(temp[0][0]))]
    # Apply filter along axis 2
    temp = [[[convolve_1d(row, filter_1d) for row in slice_] for slice_ in temp] for _ in range(len(temp[0][0]))]
    # Stride by 2
    return [[[temp[i][j][k] for k in range(0, len(temp[0][0]), 2)] for j in range(0, len(temp[0]), 2)] for i in range(0, len(temp), 2)]

def upsample(image: list, filter_1d: list) -> list:
    """Upsamples a 3D image by a factor of 2 using zero-insertion and separable 1D convolution."""
    M = 2 * len(image)
    temp = [[[0.0 for _ in range(M)] for _ in range(M)] for _ in range(M)]
    for i in range(len(image)):
        for j in range(len(image[0])):
            for k in range(len(image[0][0])):
                temp[2*i][2*j][2*k] = image[i][j][k]
    # Apply filter along axis 0
    temp = [[[convolve_1d(row, filter_1d) for row in slice_] for slice_ in temp] for _ in range(len(temp[0][0]))]
    # Transpose to axis 1
    temp = [[[temp[k][j][i] for k in range(len(temp))] for j in range(len(temp[0]))] for i in range(len(temp[0][0]))]
    # Apply filter along axis 1
    temp = [[[convolve_1d(row, filter_1d) for row in slice_] for slice_ in temp] for _ in range(len(temp[0][0]))]
    # Transpose to axis 2
    temp = [[[temp[k][j][i] for k in range(len(temp))] for j in range(len(temp[0]))] for i in range(len(temp[0][0]))]
    # Apply filter along axis 2
    temp = [[[convolve_1d(row, filter_1d) for row in slice_] for slice_ in temp] for _ in range(len(temp[0][0]))]
    return temp

def subtract_images(A: list, B: list) -> list:
    """Element-wise subtraction of two 3D arrays."""
    return [[[A[i][j][k] - B[i][j][k] for k in range(len(A[0][0]))] for j in range(len(A[0]))] for i in range(len(A))]

def evaluate_quadratic_b_spline(x: float, y: float, z: float, coefficients: list) -> float:
    """Evaluates the quadratic B-spline surface at (x, y, z) using the coefficients."""
    def B(t: float) -> float:
        if -1 <= t < 0: return 0.5 * (t + 1)**2
        if 0 <= t < 1: return 0.5 * (-2*t**2 + 3*t + 1)
        if 1 <= t < 2: return 0.5 * (t - 2)**2
        return 0.0
    ix, fx = divmod(x, 1.0)
    iy, fy = divmod(y, 1.0)
    iz, fz = divmod(z, 1.0)
    val = 0.0
    for dx in range(-1, 2):
        for dy in range(-1, 2):
            for dz in range(-1, 2):
                cx = int(ix) + dx
                cy = int(iy) + dy
                cz = int(iz) + dz
                cx = cx % len(coefficients)
                cy = cy % len(coefficients[0])
                cz = cz % len(coefficients[0][0])
                val += coefficients[cx][cy][cz] * B(fx - dx) * B(fy - dy) * B(fz - dz)
    return val
```
## Operating characteristics
*   **Time Complexity:** Generally slower than Perlin noise due to the complexity of the B-spline evaluation and the preprocessing cost [S2, unofficial][S4].
*   **Memory Complexity:** Requires storing the coefficient tile, which is $O(N^d)$ where $N$ is the tile size and $d$ is the dimensionality [S2, unofficial][S4]. However, since the tile is small and tiled, the effective memory usage per query is low, but the initial storage is higher than $O(1)$ methods like Perlin noise [S2, unofficial][S4].
*   **Strengths:** Almost perfect band-limiting prevents aliasing and detail loss [S1][S2, unofficial][S3, S4]. Supports solid noise and isotropic filtering [S4]. Provides strict spectral control via a weighted sum [S2, unofficial][S4].
*   **Limitations:** Slower than Perlin noise [S4]. Sampling a 3D function on a 2D surface can still cause aliasing [S1][S2, unofficial]. Lacks true random accessibility due to fixed-size tiling [S2, unofficial].
*   **When to Use:** When high-quality, band-limited noise is required, particularly for rendering where aliasing artifacts are unacceptable [S2, unofficial].
*   **When Not to Use:** When performance is critical and some aliasing is acceptable, or when $O(1)$ memory usage is strictly required [S2, unofficial][S4].
## Background
Wavelet noise was introduced by Cook and DeRose in 2005 [S1][S2, unofficial][S3, S4]. They observed that Perlin noise is only weakly band-limited, leading to aliasing and detail loss when summing octaves for fractal noise [S1][S2, unofficial][S3, S4]. Wavelet noise addresses this by explicitly constructing a band-limited noise field using quadratic B-splines [S1, S3, S4]. The method extends wavelet-based noise generation to provide a smooth, band-limited alternative to gradient-based noise functions [S1, S3, S4]. It is part of the family of explicit noise functions, alongside anisotropic noise [S1, S3, S4].
## Sources
[S1] A Survey of Procedural Noise Functions, https://doi.org/10.1111/j.1467-8659.2010.01827.x, fetched 2026-10-27
[S2] Terrain synthesis using noise, https://trepo.tuni.fi/handle/10024/101043, fetched 2026-10-26 (unofficial)
[S3] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S4] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
