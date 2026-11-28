# NeRF Accumulation Map Thresholding

Also called: Accumulation Map Thresholding

## Overview
NeRF Accumulation Map Thresholding is a post-processing method for extracting point clouds from Neural Radiance Field (NeRF) volume rendering, specifically designed to reduce noise and outliers in medical imaging contexts. It differs from its parent, Density Threshold Filtering, by operating on the accumulated transmittance/intensity along rays rather than the raw density field, which provides a more robust signal for detecting volumetric structures. It serves as a baseline for comparison against adaptive and 2D detection methods in sparse-view volumetry tasks.

## How it works
**Inputs and Outputs:**
- Input: A rendered NeRF accumulation map (volume) and the corresponding camera geometry (field-of-view, ray origins/directions).
- Output: A set of 3D points representing the detected volume.

**Data Structures:**
- The accumulation map is a 3D grid (e.g., $64^3$) where each voxel stores the accumulated value along the ray passing through it.
- Camera parameters include the field-of-view (FOV) and ray casting geometry used during rendering.

**Steps:**
1. **Thresholding:** Compute the 95th percentile value of the accumulation map. Let $V_{acc}$ be the set of all voxel values in the accumulation map. The threshold $T$ is defined as:
   $$ T = \text{percentile}(V_{acc}, 95) $$
   Any voxel with an accumulation value greater than $T$ is marked as part of the detected region. [S1, unofficial]

2. **Back-projection:** For each detected voxel, determine its 3D position. This is done by back-projecting the detected regions using the mean ray depth associated with the accumulation and the known field-of-view geometry. [S1, unofficial]

**Parameters:**
- Threshold percentile: 95th percentile. [S1, unofficial]
- Grid resolution: $64^3$ (in the cited medical imaging application). [S1, unofficial]

**Edge Cases and Pitfalls:**
- The method may capture large contiguous regions (e.g., the full lung field) rather than specific small structures (e.g., nodules) if the threshold does not sufficiently discriminate local intensity peaks. [S1, unofficial]
- Precision issues may arise if the accumulation map values are not normalized or if the rendering process introduces saturation artifacts. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_accumulation_map_thresholding(accumulation_map, camera_geometry):
    """
    Extract points from NeRF accumulation map using 95th percentile thresholding.
    
    Args:
        accumulation_map: 3D numpy array of shape (H, W, D) or similar grid layout.
        camera_geometry: Object containing field_of_view and ray casting functions.
        
    Returns:
        points: List of 3D coordinates (x, y, z) for detected voxels.
    """
    # Step 1: Compute threshold at 95th percentile
    # Flatten the map to get all voxel values
    all_values = accumulation_map.flatten()
    threshold = np.percentile(all_values, 95)
    
    # Identify detected voxels
    detected_mask = accumulation_map > threshold
    
    # Step 2: Back-project detected regions
    points = []
    
    # Get indices of detected voxels
    voxel_indices = np.argwhere(detected_mask)
    
    for idx in voxel_indices:
        # Convert voxel index to 3D coordinate
        # This requires knowledge of the grid spacing and origin
        # Assuming uniform grid with spacing 's' and origin 'o'
        # Note: Exact back-projection formula depends on specific renderer implementation
        # [G] Standard linear interpolation from voxel center to world space
        voxel_center = idx + 0.5
        world_coord = camera_geometry.voxel_to_world(voxel_center)
        
        # Use mean ray depth if available in camera_geometry for refinement
        # [G] If mean ray depth is stored per ray, average the depths for rays passing through this voxel
        # For simplicity, we assume voxel_to_world provides the center position
        points.append(world_coord)
        
    return points
```

## Subtopics

### Otsu Adaptive Thresholding
Replaces the fixed 95th percentile threshold with Otsu's adaptive criterion applied to the MedNeRF accumulation map. [S1, unofficial] However, this variant suffers from rendering saturation in MedNeRF, which causes the accumulation map values to collapse to the range $[-1, 1]$. This collapse leads Otsu's algorithm to select near-zero thresholds, resulting in a detected volume of only $5.4 \text{ mm}^3$ (99.9% error, 12/72 valid detections), which is worse than the fixed threshold method in every respect. [S1, unofficial]

## Operating characteristics
- **Accuracy:** Achieves a 32.1% volumetric error with 45/72 valid detections in the tested medical imaging scenario. [S1, unofficial]
- **Limitations:** Frequently captures the entire lung field rather than isolating specific nodules, indicating poor specificity for small structures. [S1, unofficial]
- **Comparison:** Outperforms Otsu Adaptive Thresholding and 2D detection methods (CheXNet, LUNA16-style UNet), which yield errors above 86% and exceed 1000% on smaller patients. [S1, unofficial]
- **Scalability:** Performance is sensitive to the choice of percentile and the quality of the NeRF rendering (e.g., saturation issues). [S1, unofficial]

## Background
This method was evaluated in the context of sparse-view lung nodule volumetry using Anatomy-Regularized TensoRF (AReT). It serves as a baseline for comparing different thresholding strategies (fixed vs. adaptive) and 3D vs. 2D detection approaches. The study highlights the challenges of isolating sub-centimeter nodules without explicit 3D geometric reasoning. [S1, unofficial]

## Sources
[S1] Sparse-View Lung Nodule Volumetry from Digitally Reconstructed Radiographs via AReT: Anatomy-Regularized TensoRF, https://doi.org/10.48550/arxiv.2606.02639, fetched 2026-10-26 (unofficial)
