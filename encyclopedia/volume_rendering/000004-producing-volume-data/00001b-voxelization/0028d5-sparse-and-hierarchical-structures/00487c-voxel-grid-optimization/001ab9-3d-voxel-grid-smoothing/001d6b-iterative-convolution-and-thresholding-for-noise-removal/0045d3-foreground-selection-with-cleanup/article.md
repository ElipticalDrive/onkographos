# Foreground Selection with Cleanup

Also called: Data Preparation for PCbTF, Intensity Thresholding with Morphological Cleanup, Outlier Polylines Erasure, Region Growing Background Removal

## Overview
Foreground Selection with Cleanup isolates the relevant volumetric data (foreground) from the surrounding empty space (background) and removes small, scattered noise artifacts. It serves as a preprocessing step for volume rendering pipelines, particularly those utilizing parallel coordinates for transfer function design [S1]. Unlike its parent method, which handles scattered noise via iterative convolution, this method focuses on removing the large background volume to reduce computational complexity and applying morphological operations to clean up the remaining foreground structure [S1][S2, unofficial].

## How it works
**Inputs:**
- `volume`: A 3D numpy array of voxel intensities (e.g., uint8, float32).
- `threshold`: A scalar intensity value. Voxels with intensity $\le$ threshold are considered background.
- `cleanup`: A string specifying the cleanup strategy: `'none'`, `'connected'`, `'opening'`, or `'legacy'`. Default: `'none'`.
- `min_region_voxels`: An integer specifying the minimum number of voxels a connected component must have to be retained. Default: 512.

**Outputs:**
- `mask`: A 3D boolean array of the same shape as `volume`, where `True` indicates foreground voxels and `False` indicates background/noise.

**Data Structures:**
- `mask`: Boolean array.
- `labels`: Integer array from connected component labeling.
- `sizes`: Integer array where index $i$ holds the voxel count of the component labeled $i$.

**Steps:**

1. **Region Growing Background Removal (Optional Optimization):**
   If a set of seed points within the object is known, use region growing to identify and remove background voxels (air) surrounding the object. This reduces the search space for subsequent steps [S1]. If no seeds are provided, proceed to step 2. [G: If seeds are not provided, this step is skipped, and the full volume is processed in step 2.]

2. **Initial Thresholding:**
   Create an initial binary mask by comparing voxel intensities against the threshold:
   $$ mask = (volume > threshold) $$

3. **Outlier Noise Removal (Conceptual):**
   Apply the concept of outlier detection to remove randomly distributed noise. In the context of parallel coordinates-based transfer function design, this corresponds to erasing outlier polylines [S1]. In the implementation provided by [S2, unofficial], this is implicitly handled by the subsequent morphological cleanup steps which remove small disconnected components.

4. **Morphological Cleanup:**
   Apply the selected cleanup strategy to the `mask`.

   - **Case `'none'`:**
     Do nothing. `mask` remains as defined in Step 2.

   - **Case `'legacy'`:**
     1. Calculate margin:
        $$ margin = \min(3, \lfloor \min(\text{volume.shape}) / 10 \rfloor) $$
     2. If $margin > 0$, set `mask` to `False` for edge slices of size `margin` on all three axes (both positive and negative ends).
     3. If $margin > 0$ and $\min(\text{volume.shape}) > 10$, apply 3x3x3 binary erosion:
        $$ mask = \text{binary\_erosion}(mask, \text{structure}=\text{ones}((3,3,3))) $$

   - **Case `'opening'`:**
     Apply 3x3x3 binary opening:
     $$ mask = \text{binary\_opening}(mask, \text{structure}=\text{ones}((3,3,3))) $$

   - **Case `'connected'`:**
     1. Label connected components in `mask` using six-neighbor connectivity:
        $$ labels, regions\_before = \text{label}(mask) $$
     2. Compute region sizes:
        $$ sizes = \text{bincount}(labels.\text{ravel}()) $$
     3. Create a keep mask for components with size $\ge$ `min_region_voxels`:
        $$ keep = (sizes \ge min\_region\_voxels) $$
     4. Set `keep[0] = False` (label 0 is background).
     5. Apply keep mask:
        $$ mask = keep[labels] $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy import ndimage

def foreground_selection_with_cleanup(volume, threshold, cleanup='none', min_region_voxels=512):
    """
    Select foreground voxels and apply cleanup.
    
    Args:
        volume (np.ndarray): 3D array of voxel intensities.
        threshold (float): Intensity threshold. Voxels > threshold are foreground.
        cleanup (str): One of 'none', 'connected', 'opening', 'legacy'.
        min_region_voxels (int): Minimum size for a region to be kept (used in 'connected').
        
    Returns:
        np.ndarray: Boolean mask of foreground voxels.
    """
    # Step 2: Initial Thresholding
    mask = volume > threshold
    
    # Step 4: Morphological Cleanup
    if cleanup == 'none':
        pass
    elif cleanup == 'legacy':
        # Calculate margin
        min_dim = np.min(volume.shape)
        margin = min(3, min_dim // 10)
        
        if margin > 0:
            # Remove edge slices
            # Create a copy to avoid modifying mask in place during slicing if needed,
            # though boolean indexing is safe.
            mask[:margin, :, :] = False
            mask[-margin:, :, :] = False
            mask[:, :margin, :] = False
            mask[:, -margin:, :] = False
            mask[:, :, :margin] = False
            mask[:, :, -margin:] = False
            
            # Apply erosion if min_dim > 10
            if min_dim > 10:
                structure = np.ones((3, 3, 3), dtype=bool)
                mask = ndimage.binary_erosion(mask, structure=structure)
                
    elif cleanup == 'opening':
        structure = np.ones((3, 3, 3), dtype=bool)
        mask = ndimage.binary_opening(mask, structure=structure)
        
    elif cleanup == 'connected':
        # Label connected components with 6-connectivity
        labels, num_features = ndimage.label(mask)
        
        # Compute sizes of each label
        # labels.ravel() flattens the array. bincount counts occurrences of each label.
        sizes = np.bincount(labels.ravel())
        
        # Determine which labels to keep
        # Note: label 0 is background, so we must ensure it is excluded.
        keep = sizes >= min_region_voxels
        keep[0] = False
        
        # Apply the keep mask to the labels
        mask = keep[labels]
        
    else:
        raise ValueError(f"Unknown cleanup method: {cleanup}")
        
    return mask
```

**Edge Cases and Pitfalls:**
- **Precision:** The threshold comparison `volume > threshold` is strict. Ensure the data type of `volume` and `threshold` are compatible. [G: For integer volumes, ensure threshold is cast to the same type or float to avoid truncation issues if threshold is float.]
- **Boundaries:** The `'legacy'` method explicitly removes border voxels. This may cut off valid foreground data if the object touches the volume boundary. [S2, unofficial]
- **Degenerate Input:** If `volume` is empty or all voxels are below threshold, `mask` will be all `False`. The `connected` cleanup will handle this gracefully (no labels or only background label). [G: Ensure `labels` is not empty before calling `bincount` if `mask` is all False, though `bincount` on an empty array returns an empty array, and `keep[labels]` would fail if `labels` is empty but `keep` is not. However, `ndimage.label` on an all-False mask returns `labels` of all zeros and `num_features=0`. `sizes` will have length 1 (for label 0). `keep` will be `[False]`. `keep[labels]` will be an array of `False` with the same shape as `labels`. This is safe.]

## Subtopics

### Region Growing Background Removal
A technique to remove background voxels using seed points. It is chosen when the object's approximate location is known and computational speed is critical. It reduces the complexity of subsequent operations [S1]. [Link: [Region Growing](link-to-region-growing.md)]

### Binary Morphological Operations
Operations like erosion and opening used to smooth boundaries and remove small noise. Opening is erosion followed by dilation. It is chosen for general noise removal without significant boundary loss [S2, unofficial]. [Link: [Morphological Operations](link-to-morphological-operations.md)]

### Connected Component Analysis
Labeling and filtering components by size. It is chosen when small, disconnected noise artifacts need to be removed while preserving larger structures [S2, unofficial]. [Link: [Connected Components](link-to-connected-components.md)]

## Operating characteristics
- **Time Complexity:**
  - Thresholding: $O(N)$, where $N$ is the number of voxels.
  - Region Growing: Depends on the number of seeds and the size of the background. Typically faster than global operations if the background is large [S1].
  - Morphological Operations: $O(N \cdot K)$, where $K$ is the kernel size (e.g., 27 for 3x3x3).
  - Connected Component Labeling: $O(N)$ with efficient algorithms (e.g., two-pass or union-find) [S2, unofficial].
- **Memory Complexity:** $O(N)$ for the mask and labels arrays.
- **Strengths:**
  - Simple and fast.
  - Effective at removing large background volumes and small noise artifacts.
  - The `'legacy'` method provides a historical compatibility mode for specific datasets [S2, unofficial].
- **Limitations:**
  - The `'legacy'` method may remove valid foreground data at the boundaries.
  - Thresholding is sensitive to the choice of `threshold`.
  - Does not handle partial volume effects or complex intensity distributions well without additional preprocessing.
- **When to Use:**
  - When the background is uniform and significantly different in intensity from the foreground.
  - When computational resources are limited and a quick cleanup is needed.
  - When preparing data for parallel coordinates-based transfer function design [S1].
- **When Not to Use:**
  - When the foreground and background have overlapping intensity ranges.
  - When the object of interest is small and close to the volume boundary (avoid `'legacy'`).
  - When high precision is required for boundary representation (morphological operations smooth boundaries).

## Background
The method originates from the need to prepare volumetric data for multi-dimensional reduction and transfer function design using parallel coordinates [S1]. The region growing approach was introduced to decrease computational time and operation complexity by removing air voxels before extracting statistical attributes [S1]. The concept of outlier polylines was applied to remove randomly distributed noise in the parallel coordinate space [S1]. The implementation details for `'legacy'`, `'opening'`, and `'connected'` cleanup modes are documented in the `VolumeSegmenter` repository [S2, unofficial]. The `'legacy'` mode is specifically noted as an audit for border removal and erosion, not a reconstruction of historical outputs [S2, unofficial].

## Sources
[S1] Multi-dimensional Reduction and Transfer Function Design using Parallel Coordinates, https://doi.org/10.2312/vg/vg10/069-076, fetched 2026-10-26
[S2] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
