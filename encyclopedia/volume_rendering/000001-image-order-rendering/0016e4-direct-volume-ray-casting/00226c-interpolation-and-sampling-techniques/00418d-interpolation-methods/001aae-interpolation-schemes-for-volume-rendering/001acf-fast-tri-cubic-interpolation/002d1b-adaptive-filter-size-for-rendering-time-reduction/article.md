# Adaptive Filter Size for Rendering Time Reduction

Also called: Adaptive interpolation, Adaptive interpolation filter, Energy-based adaptive filtering, Energy-based filter selection

## Overview
Adaptive Filter Size for Rendering Time Reduction is a technique to reduce rendering time by adaptively selecting smaller interpolation filters for low-energy frequency components, maintaining image quality within a user-defined tolerance [S1, unofficial][S3, unofficial]. It sets itself apart from standard interpolation schemes by dynamically switching between a high-quality large filter and a low-cost small filter based on the spectral energy of the input data, rather than using a fixed kernel for all samples [S2, unofficial]. This method is specifically designed for frequency-domain volume rendering, building on [Fast tri-cubic interpolation](848-fast-tri-cubic-interpolation.md) by optimizing the convolution cost during slice extraction [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset transformed into its 3D frequency spectrum (complex floating-point values) [S1, unofficial].
*   **Input:** A user-defined tolerance $\epsilon$ for the maximum acceptable mean square error [S2, unofficial].
*   **Input:** Two interpolation filters: a large filter $H_1$ (e.g., $3 \times 3 \times 3$) and a small filter $H_2$ (e.g., $1 \times 1 \times 1$) [S1, unofficial].
*   **Output:** A rendered 2D slice image with reduced computational cost compared to using $H_1$ uniformly [S2, unofficial].

**Data Structures:**
*   **Spectrum:** A 3D array of complex numbers representing the Fourier transform of the volume [S1, unofficial].
*   **Mark Array:** A 3D boolean array of the same dimensions as the spectrum. `Mark[i,j,k]` is true if the voxel at `(i,j,k)` is eligible for the small filter $H_2$ [S1, unofficial].

**Parameters:**
*   **Filter Sizes:** $H_1$ is typically $3 \times 3 \times 3$ (27 input samples per output). $H_2$ is typically $1 \times 1 \times 1$ (1 input sample per output) [S1, unofficial][S2, unofficial][S3, unofficial].
*   **Tolerance ($\epsilon$):** User-defined bound for mean square error [S2, unofficial].
*   **Energy Distribution:** Empirically, 99% of the spectral energy is contained in about 10% of the frequency components [S1, unofficial][S2, unofficial][S3, unofficial].

**Formulas:**
The adaptive scheme approximates the result of filtering the entire spectrum $F$ by $H_1$ by splitting $F$ into two sets: $F_1$ (samples filtered by $H_1$) and $F_2$ (samples filtered by $H_2$), where $F_1 + F_2 = F$ [S2, unofficial]. The spatial domain result $f$ is approximated as:
$$ f \approx F_1 * H_1 + F_2 * H_2 $$
This can be rewritten to highlight the error term:
$$ f \approx F * H_1 + F_2 * (H_2 - H_1) $$
The term $F_2 * (H_2 - H_1)$ represents the error introduced by using $H_2$ for those components [S2, unofficial]. The mean square error (MSE) upper bound is derived using Rayleigh’s theorem in the frequency domain [S2, unofficial]:
$$ \text{MSE} \le \frac{1}{L} h_{\max}^2 \int |f|^2 dx $$
Where:
*   $L$ is the length of the non-zero region of the signal $f$ [S2, unofficial].
*   $h_{\max}$ is the maximum value of $|H_2 - H_1|$ in the spatial domain [S2, unofficial].
*   The integral $\int |f|^2 dx$ is the total energy of the signal [S2, unofficial].

To ensure the MSE is below the tolerance $\epsilon$, samples are marked for $H_2$ if their magnitude satisfies the error bound equation derived from the above inequality [S1, unofficial][S2, unofficial][S3, unofficial].

**Numbered Steps:**

1.  **Preprocessing (Marking):**
    *   Iterate through every voxel in the 3D spectrum [S1, unofficial].
    *   Calculate the magnitude of the frequency component [S1, unofficial].
    *   Determine if the component satisfies the error bound equation for the user-defined tolerance [S1, unofficial].
    *   **Optimization:** To avoid testing all neighbors during rendering, modify the marking criteria: mark a voxel only if it satisfies the error bound AND all neighboring voxels within a distance equal to half the support of $H_1$ also satisfy the bound [S1, unofficial][S3, unofficial].
    *   Store the result in the Mark Array. This step is performed once after the volume is transformed to the frequency domain [S1, unofficial][S2, unofficial][S3, unofficial].

2.  **Slice Extraction (Rendering):**
    *   For each sample point on the slice plane [S1, unofficial]:
    *   **Optimized Check:** Due to the conservative marking in Step 1, it is sufficient to test only the spectrum voxel closest to the slice sample position [S1, unofficial][S3, unofficial].
    *   If that specific voxel is marked, it is guaranteed that all other voxels in the support of $H_1$ are also marked [S1, unofficial]. Use $H_2$ if marked; otherwise, use $H_1$ [S1, unofficial][S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def preprocess_spectrum(spectrum, tolerance, h_max, L, total_energy):
    """
    Marks spectrum voxels eligible for the small filter H2.
    spectrum: 3D list of complex numbers.
    tolerance: User-defined MSE bound.
    h_max: Maximum value of |H2 - H1| in spatial domain.
    L: Length of non-zero region of signal.
    total_energy: Integral of |f|^2.
    """
    # The sources state that a voxel is marked if its magnitude satisfies 
    # the error bound equation (16), derived from the MSE inequality.
    # The exact constant for Eq 16 is not provided in the sources [G].
    # We implement a threshold check based on the bound.
    threshold = tolerance * L / (h_max * h_max)
    
    r, c, d = len(spectrum), len(spectrum[0]), len(spectrum[0][0])
    marked = [[[False for _ in range(d)] for _ in range(c)] for _ in range(r)]
    
    for i in range(r):
        for j in range(c):
            for k in range(d):
                mag_sq = abs(spectrum[i][j][k]) ** 2
                
                # Check if this voxel satisfies the error bound
                if mag_sq <= threshold:
                    is_safe = True
                    # Check neighbors within half support of H1
                    # Assuming H1 is 3x3x3, half support is 1.5.
                    # Neighbors within distance 1.5 from (i,j,k) are those with dx,dy,dz in {-1,0,1}
                    # such that sqrt(dx^2+dy^2+dz^2) <= 1.5.
                    # This includes (0,0,0), (1,0,0) type neighbors, but NOT (1,1,1) which is dist 1.732.
                    for di in [-1, 0, 1]:
                        for dj in [-1, 0, 1]:
                            for dk in [-1, 0, 1]:
                                if di == 0 and dj == 0 and dk == 0:
                                    continue
                                if di**2 + dj**2 + dk**2 <= 2.25:
                                    ni, nj, nk = i + di, j + dj, k + dk
                                    if 0 <= ni < r and 0 <= nj < c and 0 <= nk < d:
                                        if abs(spectrum[ni][nj][nk]) ** 2 > threshold:
                                            is_safe = False
                                            break
                            if not is_safe: break
                        if not is_safe: break
                    
                    if is_safe:
                        marked[i][j][k] = True
    return marked

def render_slice(spectrum, marked, slice_points):
    """
    Renders a slice using adaptive filtering.
    slice_points: List of (x, y, z) coordinates in spectrum space.
    """
    image = []
    r, c, d = len(spectrum), len(spectrum[0]), len(spectrum[0][0])
    
    for point in slice_points:
        # Find the closest spectrum voxel to the slice sample
        closest_voxel = (int(round(point[0])), int(round(point[1])), int(round(point[2])))
        
        # Optimized check: if the closest voxel is marked, use H2
        if marked[closest_voxel[0]][closest_voxel[1]][closest_voxel[2]]:
            # Use small filter H2 (e.g., 1x1x1, just fetch the value)
            value = spectrum[closest_voxel[0]][closest_voxel[1]][closest_voxel[2]]
        else:
            # Use large filter H1 (e.g., 3x3x3, fetch 27 neighbors)
            value = 0
            for di in [-1, 0, 1]:
                for dj in [-1, 0, 1]:
                    for dk in [-1, 0, 1]:
                        ni, nj, nk = closest_voxel[0] + di, closest_voxel[1] + dj, closest_voxel[2] + dk
                        if 0 <= ni < r and 0 <= nj < c and 0 <= nk < d:
                            value += spectrum[ni][nj][nk]
        image.append(value)
    return image
```

**Edge Cases and Pitfalls:**
*   **Precision:** The 3D spectrum is complex and requires floating-point representation. Using double precision consumes 16 times more memory than single precision, which is a significant cost [S3, unofficial].
*   **Ghosts:** Using the small filter $H_2$ uniformly causes strong ghosts in the image because the smaller filter has a less sharp cutoff in the spatial domain [S1, unofficial][S2, unofficial][S3, unofficial].
*   **Testing Cost:** Testing all voxels in the larger filter support during slice extraction is expensive. The optimization of conservative marking (checking neighbors during preprocessing) avoids this runtime cost [S1, unofficial][S3, unofficial].

## Operating characteristics
*   **Time Complexity:** The cost of interpolation is $O(n^3)$ and is determined by the filter size [S2, unofficial][S3, unofficial]. A $3 \times 3 \times 3$ filter requires 27 input samples per output sample, while a $1 \times 1 \times 1$ filter requires 1 [S1, unofficial][S2, unofficial][S3, unofficial].
*   **Performance:** Adaptive filtering reduces average cost by using smaller filters for low-energy components. Since 99% of energy is in 10% of components, the majority of samples can potentially use the cheaper filter [S1, unofficial][S2, unofficial][S3, unofficial].
*   **Strengths:** Significant rendering time reduction while maintaining high image quality within a user-defined tolerance [S1, unofficial][S2, unofficial][S3, unofficial].
*   **Limitations:** Requires a preprocessing step (FFT and marking) which adds overhead. The memory cost for the complex spectrum is high [S3, unofficial].
*   **When to use:** When rendering time is critical and the volume data has a sparse frequency spectrum (most energy in low frequencies) [S1, unofficial].
*   **When not to use:** When memory is extremely constrained (due to complex spectrum storage) or when the volume data has high-frequency content distributed uniformly, reducing the benefit of adaptive filtering [S3, unofficial].

## Background
*   **Origin:** The method was introduced by Takashi Totsuka and Marc Levoy in their work on Frequency Domain Volume Rendering [S1, unofficial][S2, unofficial][S3, unofficial].
*   **History:** It is part of the broader frequency domain volume rendering pipeline, which transforms volume data to the frequency domain to perform shading and filtering operations more efficiently [S2, unofficial].
*   **Usage:** Used in frequency domain volume rendering systems to optimize the slice extraction step, which is often the bottleneck [S1, unofficial][S3, unofficial].

## Sources

[S1] Frequency Domain Volume Rendering - Stanford University, https://graphics.stanford.edu/papers/fvr/totsuka-fvr-sig93.pdf, fetched 2026-10-27 (unofficial)
[S2] FREQUENCY DOMAIN VOLUME RENDERING Takashi Totsuka Marc Levoy ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/93/570/CSL-TR-93-570.pdf, fetched 2026-10-27 (unofficial)
[S3] Frequency Domain Volume Rendering, https://graphics.stanford.edu/papers/fvr/fvr.pdf, fetched 2026-10-27 (unofficial)
[S4] Frequency Domain Volume Rendering, https://graphics.stanford.edu/papers/fvr/fvr.pdf, fetched 2026-10-27 (unofficial)
