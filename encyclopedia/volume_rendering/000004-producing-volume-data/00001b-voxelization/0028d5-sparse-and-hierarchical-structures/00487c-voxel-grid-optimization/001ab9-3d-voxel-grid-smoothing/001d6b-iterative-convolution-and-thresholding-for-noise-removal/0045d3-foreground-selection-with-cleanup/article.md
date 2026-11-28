# Foreground Selection with Cleanup

Also called: Data Preparation for PCbTF, Intensity Thresholding with Morphological Cleanup, Outlier Polylines Erasure, Region Growing Background Removal

## Overview
Foreground Selection with Cleanup isolates relevant volumetric data from surrounding empty space and removes small noise artifacts [S2, unofficial]. It serves as a preprocessing step for volume rendering pipelines, particularly those utilizing parallel coordinates for transfer function design [S1]. It applies intensity thresholding and optional morphological cleanup to select foreground voxels [S2, unofficial].

## How it works
**Inputs:**
- `volume`: A 3D array of voxel intensities [S2, unofficial].
- `threshold`: A scalar intensity value. Voxels with intensity $>$ threshold are considered foreground [S2, unofficial].
- `cleanup`: A string specifying the cleanup strategy: `'none'`, `'connected'`, `'opening'`, or `'legacy'`. Default: `'none'` [S2, unofficial].
- `min_region_voxels`: An integer specifying the minimum number of voxels a connected component must have to be retained. Default: 512 [S2, unofficial].

**Outputs:**
- `mask`: A 3D boolean array of the same shape as `volume`, where `True` indicates foreground voxels [S2, unofficial].

**Data Structures:**
- `mask`: Boolean array representing the selected foreground [S2, unofficial].
- `labels`: Integer array from connected component labeling [S2, unofficial].
- `sizes`: Integer array where index $i$ holds the voxel count of the component labeled $i$ [S2, unofficial].

**Steps:**

1. **Region Growing Background Removal (Optional Optimization):**
   If seed points within the object are known, use region growing to remove background air voxels, decreasing computational time and operation complexity [S1]. If no seeds are provided, proceed to step 2 [G: Region growing is not detailed in the sources; implement a standard seed-based flood fill or BFS/DFS on the volume.].

2. **Initial Thresholding:**
   Create an initial binary mask by comparing voxel intensities against the threshold:
   $$ mask = (volume > threshold) $$ [S2, unofficial].

3. **Outlier Noise Removal (Conceptual):**
   Apply the concept of an outlier to remove randomly distributed noise, which corresponds to erasing outlier polylines in parallel coordinate space [S1].

4. **Morphological Cleanup:**
   Apply the selected cleanup strategy to the `mask` [S2, unofficial].
   - **Case `'none'`:** Do nothing. `mask` remains as defined in Step 2 [S2, unofficial].
   - **Case `'legacy'`:** Calculate `margin = min(3, min(volume.shape) // 10)` [S2, unofficial]. If `margin > 0`, set `mask` to `False` for edge slices of size `margin` on all three axes [S2, unofficial]. If `margin > 0` and `min(volume.shape) > 10`, apply 3x3x3 binary erosion [S2, unofficial].
   - **Case `'opening'`:** Apply 3x3x3 binary opening [S2, unofficial].
   - **Case `'connected'`:** Label `mask` with six-neighbor connectivity [S2, unofficial]. Compute region sizes. Create keep mask for sizes $\ge$ `min_region_voxels` [S2, unofficial]. Set `keep[0] = False` (label 0 is background) [S2, unofficial]. Apply keep mask: `mask = keep[labels]` [S2, unofficial].

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
    # Step 1: Region Growing Background Removal (Optional)
    # [G: Sources do not provide the region growing algorithm. 
    # Implement a standard seed-based flood fill or BFS/DFS here if seeds are provided.]
    
    # Step 2: Initial Thresholding
    mask = volume > threshold
    
    # Step 4: Morphological Cleanup
    if cleanup == 'none':
        pass
    elif cleanup == 'legacy':
        min_dim = np.min(volume.shape)
        margin = min(3, min_dim // 10)
        
        if margin > 0:
            mask[:margin, :, :] = False
            mask[-margin:, :, :] = False
            mask[:, :margin, :] = False
            mask[:, -margin:, :] = False
            mask[:, :, :margin] = False
            mask[:, :, -margin:] = False
            
            if min_dim > 10:
                structure = np.ones((3, 3, 3), dtype=bool)
                mask = ndimage.binary_erosion(mask, structure=structure)
                
    elif cleanup == 'opening':
        structure = np.ones((3, 3, 3), dtype=bool)
        mask = ndimage.binary_opening(mask, structure=structure)
        
    elif cleanup == 'connected':
        # 6-neighbor connectivity structure (center + 6 axial neighbors)
        structure_6 = np.zeros((3, 3, 3), dtype=bool)
        structure_6[1, 1, 1] = True
        structure_6[0, 1, 1] = True
        structure_6[2, 1, 1] = True
        structure_6[1, 0, 1] = True
        structure_6[1, 2, 1] = True
        structure_6[1, 1, 0] = True
        structure_6[1, 1, 2] = True
        
        labels, _ = ndimage.label(mask, structure=structure_6)
        sizes = np.bincount(labels.ravel())
        keep = sizes >= min_region_voxels
        keep[0] = False
        mask = keep[labels]
        
    else:
        raise ValueError(f"Unknown cleanup method: {cleanup}")
        
    return mask
```

**Edge Cases and Pitfalls:**
- **Precision:** The threshold comparison `volume > threshold` is strict; ensure compatible data types between `volume` and `threshold` [S2, unofficial].
- **Boundaries:** The `'legacy'` method explicitly removes border voxels, which may cut off valid foreground data if the object touches the volume boundary [S2, unofficial].
- **Degenerate Input:** If all voxels are below threshold, `mask` will be all `False`; `ndimage.label` returns all zeros, `sizes` has length 1, and `keep[labels]` safely returns an all-`False` mask [G].

## Subtopics

### Region Growing Background Removal
Removes background air voxels using seed points [S1]. Sets itself apart by reducing the search space for subsequent steps [S1]. Choose when the object's approximate location is known and computational speed is critical [S1]. [Link: [Region Growing](region-growing.md)]

### Binary Morphological Operations
Applies erosion and opening to smooth boundaries and remove small noise [S2, unofficial]. Sets itself apart by structurally modifying the mask shape [S2, unofficial]. Choose for general noise removal [S2, unofficial]. [Link: [Morphological Operations](morphological-operations.md)]

### Connected Component Analysis
Labels and filters components by size [S2, unofficial]. Sets itself apart by isolating spatially contiguous structures [S2, unofficial]. Choose when small disconnected noise must be removed while preserving larger structures [S2, unofficial]. [Link: [Connected Components](connected-components.md)]

## Operating characteristics
- **Time and Memory Complexity:** Not given in the sources.
- **Strengths:** Simple and fast; effective at removing large background volumes and small noise artifacts; `'legacy'` mode provides historical compatibility for specific datasets [S1][S2, unofficial].
- **Limitations:** `'legacy'` may remove valid foreground data at boundaries; thresholding is sensitive to the `threshold` choice; does not handle partial volume effects or complex intensity distributions well [S2, unofficial].
- **When to Use:** Background is uniform and significantly different in intensity from the foreground; computational resources are limited; preparing data for parallel coordinates-based transfer function design [S1][S2, unofficial].
- **When Not to Use:** Foreground and background have overlapping intensity ranges; object is small and near the volume boundary (avoid `'legacy'`); high precision for boundary representation is required [S2, unofficial].

## Background
The method originates from preparing volumetric data for multi-dimensional reduction and transfer function design using parallel coordinates [S1]. The region growing approach decreases computational time by removing air voxels before extracting statistical attributes [S1]. The concept of outlier polylines removes randomly distributed noise in parallel coordinate space [S1]. Implementation details for `'legacy'`, `'opening'`, and `'connected'` cleanup modes are documented in the VolumeSegmenter repository [S2, unofficial]. The `'legacy'` mode audits border removal and erosion, not a reconstruction of historical outputs [S2, unofficial].

## Sources
[S1] Multi-dimensional Reduction and Transfer Function Design using Parallel Coordinates, https://doi.org/10.2312/vg/vg10/069-076, fetched 2026-10-26
[S2] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
