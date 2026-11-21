# Automatic Crop Box Generation via Otsu Thresholding

Also called: Automatic trim, Object box from coarse level

## Overview
This method automatically computes a tight bounding box for a 3D volume by thresholding a downsampled (coarse) level using Otsu's method, identifying connected components, and projecting the result to the target resolution. It solves the problem of manually placing crop boxes by providing a reproducible, data-driven alternative that significantly reduces voxel count and export size. It sets itself apart from sibling methods like [K-Means Spatial-Intensity Partitioning](11588-k-means-spatial-intensity-partitioning.md) by focusing on global intensity separation rather than spatial clustering, and from [Greedy Axis-Aligned Spatial Partitioning](11589-greedy-axis-aligned-spatial-partitioning.md) by relying on intensity-based segmentation rather than geometric splitting.

## How it works
**Inputs:**
- Volume data at a selected coarse level (e.g., level 2).
- A spatial transform (e.g., ITK transform) defining the rotation/translation from the coarse level's frame to the output frame.
- Target resolution (level 0) spacing.
- Minimum piece size filter (in voxels or physical units).
- Safety margin parameters: one rotated coarse voxel + 10 voxels at target spacing.

**Outputs:**
- A 3D axis-aligned bounding box (min/max coordinates) in the target output frame.
- Metadata recording the window/threshold used and the transform applied.

**Data Structures:**
- `CoarseVolume`: Array of intensity values at the coarse level resolution.
- `BinaryMask`: Boolean array of same shape as `CoarseVolume`, where `True` if `value > threshold`.
- `ConnectedComponents`: Label map where each voxel is assigned an integer ID corresponding to its connected component.
- `BoundingBox`: Tuple `(min_x, min_y, min_z, max_x, max_y, max_z)` in the output frame coordinates.

**Steps:**
1. **Select Coarse Level:** Choose a downsampled level (e.g., level 2) to reduce computational cost. [S1, unofficial]
2. **Compute Threshold:** Apply Otsu's method to the intensity histogram of the coarse level data to find the optimal threshold $T$. [S1, unofficial]
3. **Threshold:** Create a binary mask $M$ where $M[i] = 1$ if $CoarseVolume[i] > T$, else $0$. [S1, unofficial]
4. **Connected Components:** Identify connected components (pieces) in $M$ using 6-connectivity (or 26-connectivity, depending on implementation; 6 is standard for axis-aligned). [S1, unofficial]
5. **Filter Small Pieces:** Discard components with voxel count below a minimum threshold (e.g., to exclude noise or dust). [S1, unofficial]
6. **Rotate Voxels:** For each voxel $v$ in the kept components, apply the inverse of the coarse-to-output transform to get its position $p_{out}$ in the target frame. [S1, unofficial]
7. **Compute Bounds:** Find the minimum and maximum coordinates across all rotated voxels:
   $$ min_x = \min(p_{out}.x), \quad max_x = \max(p_{out}.x) $$
   (Similarly for $y$ and $z$). [S1, unofficial]
8. **Add Safety Margin:**
   - Add one rotated coarse voxel's physical size to each side of the bounds. [S1, unofficial]
   - Add a 10-voxel pad at the target (level 0) spacing on all six sides. [S1, unofficial]
   $$ final\_min = min - 10 \cdot \Delta_{target} - \Delta_{coarse\_rotated} $$
   $$ final\_max = max + 10 \cdot \Delta_{target} + \Delta_{coarse\_rotated} $$
   where $\Delta$ represents the voxel size in the respective frame.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_crop_box(coarse_volume, transform, target_spacing, min_piece_size):
    # Step 1: Otsu Thresholding
    threshold = otsu_threshold(coarse_volume)
    
    # Step 2: Binary Mask
    binary_mask = coarse_volume > threshold
    
    # Step 3: Connected Components
    labeled_mask, num_labels = connected_components(binary_mask)
    
    # Step 4: Filter Small Pieces
    valid_labels = set()
    for label in range(1, num_labels + 1):
        count = np.sum(labeled_mask == label)
        if count >= min_piece_size:
            valid_labels.add(label)
            
    # Step 5: Rotate Voxels and Compute Bounds
    min_coords = np.array([np.inf, np.inf, np.inf])
    max_coords = np.array([-np.inf, -np.inf, -np.inf])
    
    # Get voxel coordinates of valid pieces
    valid_voxels = np.argwhere(np.isin(labeled_mask, list(valid_labels)))
    
    for voxel in valid_voxels:
        # Convert voxel index to physical coordinates in coarse frame
        coarse_pos = voxel * coarse_spacing
        
        # Apply transform to get position in output frame
        # transform is assumed to be a 4x4 matrix or similar
        output_pos = transform.apply(coarse_pos)
        
        # Update bounds
        min_coords = np.minimum(min_coords, output_pos)
        max_coords = np.maximum(max_coords, output_pos)
        
    # Step 6: Add Margins
    # One rotated coarse voxel size (approximate as coarse_spacing transformed)
    # For simplicity, assume isotropic or use max dimension
    coarse_voxel_size = np.max(coarse_spacing) 
    # Or more precisely, transform the coarse voxel size vector
    
    pad_size = 10 * target_spacing
    
    final_min = min_coords - pad_size - coarse_voxel_size
    final_max = max_coords + pad_size + coarse_voxel_size
    
    return final_min, final_max
```

**Parameters:**
- `coarse_level`: Typically level 2. [S1, unofficial]
- `min_piece_size`: Not given in the sources. [G] (Implied by "filter out small pieces")
- `pad_voxels`: 10. [S1, unofficial]
- `safety_margin_voxels`: 1 (coarse level). [S1, unofficial]

**Edge Cases:**
- **Dust/Noise:** Small pieces (dust specks) can cause the box to expand significantly if not filtered. [S1, unofficial]
- **Low Thresholds:** Very low thresholds may include external holders or artifacts. [S1, unofficial]
- **Loose Pieces:** Detached objects (e.g., a loose tooth) may fall outside the main box. The method should report these rather than silently dropping them. [S1, unofficial]

## Subtopics

### [Chunked Otsu Thresholding](17130-chunked-otsu-thresholding.md)
A memory-efficient implementation of Otsu's method for large volumes, processing data in chunks to avoid high memory peaks. It sets itself apart by avoiding full-volume copies, making it suitable for very large datasets. Choose it when memory is constrained.

## Operating characteristics
- **Time Complexity:** Dominated by Otsu's histogram calculation and connected components on the coarse level. Significantly faster than full-resolution processing. [S1, unofficial]
- **Memory Complexity:** Proportional to the coarse level size. Chunked variants reduce peak memory. [S1, unofficial]
- **Performance:** On a dry skull at level 2 (24.5 µm, ~90 MB), the process took 12 seconds. [S1, unofficial]
- **Voxel Reduction:** The automatic box had 43% fewer voxels than a hand-placed box (11.4 G vs 20.0 G). [S1, unofficial]
- **Strengths:** Reproducible, reduces data size, excludes external artifacts if threshold is correct. [S1, unofficial]
- **Limitations:** Sensitive to threshold choice; may include holders if threshold is too low; requires piece filtering to avoid noise. [S1, unofficial]
- **When to Use:** When a tight, data-driven crop is needed without manual intervention. [S1, unofficial]
- **When Not to Use:** When the object has low contrast or is mixed with similar-intensity artifacts. [S1, unofficial]

## Background
This method is part of a preprocessing pipeline for volume data, specifically for exporting large volumes efficiently. It was developed as an alternative to hand-placed boxes to automate the cropping process. [S1, unofficial]

## Sources
[S1] Reorient + crop at full resolution: block-wise resampling, interpolation choice, output bit depth, https://github.com/muratmaga/omezarr-converter/issues/1, fetched 2026-10-27 (unofficial)
