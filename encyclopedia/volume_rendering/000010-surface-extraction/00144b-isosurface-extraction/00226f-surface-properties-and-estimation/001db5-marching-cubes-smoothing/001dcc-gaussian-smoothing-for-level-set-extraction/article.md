# Gaussian Smoothing for Level-Set Extraction
## Overview
Gaussian Smoothing for Level-Set Extraction is a fast smoothing method applied to binary volumes by converting them to float, subtracting 0.5, and applying a Gaussian filter [S1, unofficial]. It differs from its sibling [Constrained Smoothing for Level-Set Extraction](1784-constrained-smoothing-for-level-set-extraction.md) by using a direct linear convolution rather than an iterative optimization approach, making it faster but less precise in preserving thin structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A binary array (values 0 and 1) representing a volume [S1, unofficial].
- Output: A float array with smoothed values [S1, unofficial].

**Data Structures:**
- `binary_array`: Numpy array of integers or booleans [S1, unofficial].
- `vol`: Numpy array of `float64` [S1, unofficial].

**Steps:**
1. **Centering:** Convert the binary array to `float64` and subtract 0.5 [S1, unofficial].
   $$ vol = \text{float64}(binary\_array) - 0.5 $$
2. **Filtering:** Apply a Gaussian filter to the centered volume using a specified standard deviation $\sigma$ [S1, unofficial].
   $$ smoothed = \text{ndimage.gaussian\_filter}(vol, \sigma) $$

**Parameters:**
- `sigma`: The standard deviation of the Gaussian kernel. Controls the amount of smoothing. Default value is 3 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Detail Loss:** The method may destroy fine details and thin structures because the Gaussian kernel averages across boundaries indiscriminately [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy import ndimage

def gaussian_smoothing_level_set(binary_array: np.ndarray, sigma: float = 3.0) -> np.ndarray:
    """
    Apply Gaussian smoothing to a binary volume for level-set extraction.
    
    Args:
        binary_array: Binary volume (0s and 1s).
        sigma: Standard deviation for the Gaussian filter.
        
    Returns:
        Smoothed float volume centered around 0.
    """
    # Step 1: Convert to float64 and center by subtracting 0.5
    vol = np.float64(binary_array) - 0.5
    
    # Step 2: Apply Gaussian filter
    smoothed_vol = ndimage.gaussian_filter(vol, sigma=sigma)
    
    return smoothed_vol
```

## Subtopics

### [Box Smoothing for Level-Set Extraction](4420-box-smoothing-for-level-set-extraction.md)
Box Smoothing for Level-Set Extraction applies a box blur filter to a binary density field to convert sharp 0/1 steps into smooth transitions, allowing Marching Cubes to generate rounded geometry instead of sharp stair-stepped edges. It differs from its parent, Gaussian Smoothing for Level-Set Extraction, by using a simpler averaging kernel that is computationally cheaper but may preserve less of the original field's high-frequency detail.

### [Amira 3D Edge-Preserving Smoothing Filter](13392-amira-3d-edge-preserving-smoothing-filter.md)
The Amira 3D Edge-Preserving Smoothing Filter is a non-linear smoothing method applied to 3D grayscale image stacks to reduce noise while maintaining sharp boundaries between distinct structures. It addresses the limitation of linear Gaussian smoothing, which blurs edges by averaging across intensity discontinuities. Unlike the parent [Gaussian Smoothing for Level-Set Extraction](1605-gaussian-smoothing-for-level-set-extraction.md) which uses a fixed kernel, this method dynamically adjusts the convolution kernel based on local image content to prevent edge blurring.

### Quick Smooth
Quick Smooth is a 3D Gaussian-like smoothing filter applied to the input volume to soften the image before surface extraction [S2, unofficial]. It uses a fixed 5-tap kernel with specific weights rather than a true Gaussian distribution [S2, unofficial].
- **Parameters:** `kwid` (kernel width) is set to 2, `k0` (center weight) is 0.45, `k1` (adjacent weight) is 0.225, and `k2` (outer weight) is 0.05 [S2, unofficial].
- **Code:** The smoothing operation for a voxel `v` is computed as:
  $$ tmp[v] = (img[v-2] \times k2) + (img[v-1] \times k1) + (img[v] \times k0) + (img[v+1] \times k1) + (img[v+2] \times k2) $$
- **Pitfall:** The function returns `EXIT_FAILURE` if any dimension (`nx`, `ny`, `nz`) is less than 5, as the kernel width requires a minimum volume size to avoid boundary errors [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Performance:** Recommended for large arrays where the level-set does not contain thin structures [S1, unofficial].
- **Strengths:** Faster than constrained smoothing [S1, unofficial].
- **Limitations:** Less precise; tends to destroy fine details and thin structures due to the linear averaging nature of the Gaussian kernel [S1, unofficial].
- **When to use it:** Use when processing large volumes where computational speed is prioritized over preserving thin geometric features [S1, unofficial].

## Background
The method is implemented in the PyMCubes library [S1, unofficial]. The specific function `smooth_gaussian` is part of this library [S1, unofficial]. It is commonly used as a preprocessing step for Marching Cubes surface extraction to reduce aliasing artifacts in binary volume data [S1, unofficial].

## Sources
[S1] pmneila/PyMCubes, https://github.com/pmneila/PyMCubes, fetched 2026-10-26 (unofficial)
[S2] TRTUZ8/TRTUZ8.github.io, https://github.com/TRTUZ8/TRTUZ8.github.io, fetched 2026-10-28 (unofficial)
[S3] DF3 Viewer Oosawa, https://qcganime.web.fc2.com/OOSAWA/MANUAL/FilterE.html, fetched 2026-10-29
