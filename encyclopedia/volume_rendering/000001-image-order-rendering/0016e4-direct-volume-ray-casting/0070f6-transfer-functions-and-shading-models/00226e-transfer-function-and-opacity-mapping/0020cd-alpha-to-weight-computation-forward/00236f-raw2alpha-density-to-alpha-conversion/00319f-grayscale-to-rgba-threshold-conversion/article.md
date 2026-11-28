# Grayscale to RGBA Threshold Conversion

Also called: 1D to 4D threshold mapping, Alpha Thresholding

## Overview
This method converts a 1D grayscale volume buffer into a 4D RGBA buffer by replicating the scalar value to the RGB channels and applying a hard alpha threshold [S1, unofficial]. It differs from its siblings by using a binary opacity decision rather than a continuous transfer function or luminance-based mapping [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 1D array of `char` values representing grayscale intensity.
*   Output: A 1D array of `char` values with a stride of 4 per sample, laid out as `[R, G, B, A]` in row-major order [S1, unofficial].

**Data Structures:**
*   `src`: Input array of size `size`.
*   `dest`: Output array of size `size * 4`.
*   `cfg.alpha_threshold`: The threshold value for opacity [S1, unofficial].

**Steps:**
1.  Allocate the destination buffer `dest` with size `size * 4` bytes [S1, unofficial].
2.  If `cfg.alpha_threshold > 0`, identify empty voxels where the input value is less than or equal to the threshold and mark them as transparent (alpha = 0) [S1, unofficial].
3.  Iterate over each input sample `i` from `0` to `size - 1`:
    *   Set the Red channel: `dest[i * 4 + 0] = src[i]` [S1, unofficial].
    *   Set the Green channel: `dest[i * 4 + 1] = src[i]` [S1, unofficial].
    *   Set the Blue channel: `dest[i * 4 + 2] = src[i]` [S1, unofficial].
    *   Set the Alpha channel based on the threshold:
        *   If `src[i] < 30`, set `dest[i * 4 + 3] = 0` [S1, unofficial].
        *   Otherwise, set `dest[i * 4 + 3] = 255` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grayscale_to_rgba_threshold(src: list[int], size: int, alpha_threshold: int = 30) -> list[int]:
    """
    Converts a 1D grayscale buffer to a 4D RGBA buffer using a hard alpha threshold.
    
    Args:
        src: List of integers (0-255) representing grayscale values.
        size: Number of samples in src.
        alpha_threshold: Threshold below which alpha is 0. Default 30.
        
    Returns:
        List of integers (0-255) representing RGBA values.
    """
    dest = [0] * (size * 4)
    
    for i in range(size):
        val = src[i]
        
        # Replicate grayscale value to RGB channels
        dest[i * 4 + 0] = val
        dest[i * 4 + 1] = val
        dest[i * 4 + 2] = val
        
        # Apply hard alpha threshold
        # Note: The source code uses a hardcoded value of 30 for the threshold check
        # regardless of the cfg.alpha_threshold parameter in the specific implementation cited.
        if val < 30:
            dest[i * 4 + 3] = 0
        else:
            dest[i * 4 + 3] = 255
            
    return dest
```

**Parameters:**
*   `alpha_threshold`: The value below which voxels become fully transparent. The cited implementation uses a hardcoded value of `30` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The method uses integer comparison. Values exactly equal to the threshold (if defined as `<=`) or just above/below may behave differently depending on implementation details. The cited source uses `< 30` for transparency [S1, unofficial].
*   **Boundaries:** The method assumes the input array is fully populated.
*   **Degenerate Input:** If the input contains values outside the 0-255 range, behavior is undefined unless clamped.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels. Each voxel is processed once with constant-time operations.
*   **Memory Complexity:** $O(N)$, requiring $4 \times$ the memory of the input grayscale buffer.
*   **Strengths:** Extremely fast; simple to implement; suitable for binary segmentation or simple masking.
*   **Limitations:** No gradual opacity transitions; RGB channels are identical to grayscale, limiting color-based visualization.
*   **When to use:** When a simple binary mask is needed or when RGB color is not required for the volume data.
*   **When not to use:** When smooth opacity transitions or color-coded data are necessary.

## Background
This method is a basic technique for preparing grayscale volume data for rendering systems that require RGBA buffers. It is often used as a preprocessing step in slice-based volume rendering pipelines [S1, unofficial].

## Sources
[S1] abhishektrip/SliceBasedVolumeRendering, https://github.com/abhishektrip/SliceBasedVolumeRendering, fetched 2026-10-27 (unofficial)
[S2] HollaFoil/Voxmat, https://github.com/HollaFoil/Voxmat, fetched 2026-10-28 (unofficial)
