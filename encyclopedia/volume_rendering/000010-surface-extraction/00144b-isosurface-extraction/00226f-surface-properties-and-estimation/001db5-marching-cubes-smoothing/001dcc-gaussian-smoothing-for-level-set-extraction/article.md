# Gaussian Smoothing for Level-Set Extraction

## Overview
Gaussian Smoothing for Level-Set Extraction is a preprocessing method that converts a binary volume into a smooth signed distance-like field to facilitate robust surface extraction via Marching Cubes. It solves the problem of jagged, stair-stepped geometry produced by sharp binary boundaries by applying a Gaussian filter to a centered binary representation. It differs from its sibling [Constrained Smoothing for Level-Set Extraction](1784-constrained-smoothing-for-level-set-extraction.md) by using a direct linear convolution rather than an iterative optimization approach, making it faster but less precise in preserving thin structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A binary array (values 0 and 1) representing a volume.
- Output: A float array with smoothed values, where the zero-crossing approximates the surface.

**Data Structures:**
- `binary_array`: Numpy array of integers or booleans.
- `vol`: Numpy array of `float64`.

**Steps:**
1. **Centering:** Convert the binary array to `float64` and subtract 0.5. This maps the binary values $\{0, 1\}$ to $\{-0.5, 0.5\}$, centering the field around zero.
   $$ vol = \text{float64}(binary\_array) - 0.5 $$
2. **Filtering:** Apply a Gaussian filter to the centered volume using a specified standard deviation $\sigma$.
   $$ smoothed = \text{gaussian\_filter}(vol, \sigma) $$

**Parameters:**
- `sigma`: The standard deviation of the Gaussian kernel. Controls the amount of smoothing. Default value is 3 [S1, unofficial].

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
    # This maps 0 -> -0.5 and 1 -> 0.5
    vol = np.float64(binary_array) - 0.5
    
    # Step 2: Apply Gaussian filter
    # scipy.ndimage.gaussian_filter applies a separable Gaussian convolution
    smoothed_vol = ndimage.gaussian_filter(vol, sigma=sigma)
    
    return smoothed_vol
```

**Edge Cases and Pitfalls:**
- **Precision:** Using `float64` is recommended to maintain precision during the subtraction and filtering steps [S1, unofficial].
- **Detail Loss:** The method may destroy fine details and thin structures because the Gaussian kernel averages across boundaries indiscriminately [S1, unofficial].

## Subtopics

### [Box Smoothing for Level-Set Extraction](4420-box-smoothing-for-level-set-extraction.md)
Box Smoothing for Level-Set Extraction applies a box blur filter to a binary density field to convert sharp 0/1 steps into smooth transitions, allowing Marching Cubes to generate rounded geometry instead of sharp stair-stepped edges. It differs from its parent, Gaussian Smoothing for Level-Set Extraction, by using a simpler averaging kernel that is computationally cheaper but may preserve less of the original field's high-frequency detail.

### [Amira 3D Edge-Preserving Smoothing Filter](13392-amira-3d-edge-preserving-smoothing-filter.md)
The Amira 3D Edge-Preserving Smoothing Filter is a non-linear smoothing method applied to 3D grayscale image stacks to reduce noise while maintaining sharp boundaries between distinct structures. It addresses the limitation of linear Gaussian smoothing, which blurs edges by averaging across intensity discontinuities. Unlike the parent [Gaussian Smoothing for Level-Set Extraction](1605-gaussian-smoothing-for-level-set-extraction.md) which uses a fixed kernel, this method dynamically adjusts the convolution kernel based on local image content to prevent edge blurring.

### Quick Smooth
Quick Smooth is a 3D Gaussian-like smoothing filter applied to the input volume to soften the image before surface extraction [S2, unofficial]. It uses a fixed 5-tap kernel with specific weights rather than a true Gaussian distribution.
- **Parameters:**
  - `kwid`: Kernel width, set to 2 [S2, unofficial].
  - `k0`: Center kernel weight, 0.45 [S2, unofficial].
  - `k1`: Adjacent kernel weight, 0.225 [S2, unofficial].
  - `k2`: Outer kernel weight, 0.05 [S2, unofficial].
- **Code:** The smoothing operation for a voxel `v` is computed as:
  $$ tmp[v] = (img[v-2] \times k2) + (img[v-1] \times k1) + (img[v] \times k0) + (img[v+1] \times k1) + (img[v+2] \times k2) $$
- **Pitfall:** The function returns `EXIT_FAILURE` if any dimension (`nx`, `ny`, `nz`) is less than 5, as the kernel width requires a minimum volume size to avoid boundary errors [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels, $O(N)$, due to the separable nature of the Gaussian filter. It is faster than constrained smoothing [S1, unofficial].
- **Memory Complexity:** Requires storage for the input binary array and the output float array. The Gaussian filter may require temporary buffers for intermediate rows/columns/slices depending on the implementation.
- **Performance:** Recommended for large arrays where the level-set does not contain thin structures [S1, unofficial].
- **Strengths:** Fast, simple to implement, and effective for general smoothing of large binary volumes.
- **Limitations:** Less precise than constrained smoothing; tends to erode or destroy thin structures and fine details due to the linear averaging nature of the Gaussian kernel [S1, unofficial].

## Background
The method is implemented in the PyMCubes library, which provides tools for extracting isosurfaces from scalar fields. The specific function `smooth_gaussian` is part of this library's preprocessing utilities.

## Sources
[S1] pmneila/PyMCubes, https://github.com/pmneila/PyMCubes, fetched 2026-10-26 (unofficial)
[S2] TRTUZ8/TRTUZ8.github.io, https://github.com/TRTUZ8/TRTUZ8.github.io, fetched 2026-10-28 (unofficial)
