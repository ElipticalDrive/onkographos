# Depth-Prior Ray Truncation
Also called: depthPriorLearned
## Overview
Depth-Prior Ray Truncation adjusts the near and far bounds of volumetric ray sampling using a learned depth prior (disparity map) to restrict integration to relevant depth ranges, thereby reducing floating artifacts and unnecessary computation [S1, unofficial]. It differs from sibling methods like [Empty-space skipping](1002-empty-space-skipping.md) which rely on geometric emptiness, and [Early Stop](1790-early-stop.md) which relies on accumulated opacity; instead, it uses external depth information to prune the search space before integration begins.
## How it works
**Inputs and Outputs:**
- Input: Ray origin, ray direction, raw AABB intersection bounds ($t_{min}, t_{max}$), a boolean flag `depthPriorLearned`, and a per-pixel disparity value from a ground truth or learned depth map (`groundTruth.disparityDevice`).
- Output: Modified ray bounds $t_{near}$ and $t_{far}$ used for subsequent volume sampling.

**Data Structures:**
- `rayInfo`: Array or struct containing raw intersection distances. `rayInfo[0]` is the near bound, `rayInfo[1]` is the far bound.
- `groundTruth.disparityDevice`: 1D array or texture mapping pixel indices to disparity values.

**Steps:**
1. Evaluate the boolean flag `depthPriorLearned`.
2. If `depthPriorLearned` is true:
   - Retrieve the pixel index corresponding to the current ray's screen position.
   - Fetch the disparity value $d$ from `groundTruth.disparityDevice` at that pixel index.
   - Calculate the new near bound: $t_{near} = rayInfo[0] + d \cdot 0.5$.
   - Calculate the new far bound: $t_{far} = rayInfo[1] - 10.0$.
3. If `depthPriorLearned` is false:
   - Set $t_{near} = rayInfo[0]$.
   - Set $t_{far} = rayInfo[1]$.
4. Use $t_{near}$ and $t_{far}$ as the integration limits for the ray marching loop.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_depth_prior_truncation(ray_info, depth_prior_learned, ground_truth_disparity, pixel_index):
    """
    ray_info: tuple (t_min, t_max) representing raw AABB intersection bounds
    depth_prior_learned: bool
    ground_truth_disparity: array or function accessing disparity by pixel index
    pixel_index: int, index into the disparity map corresponding to the ray's screen position
    """
    t_min, t_max = ray_info
    
    if depth_prior_learned:
        # Fetch disparity for the current pixel
        disparity = ground_truth_disparity[pixel_index]
        
        # Apply scaling factor to disparity for near bound
        # Note: 0.5 is a fixed scaling factor from the source
        t_near = t_min + disparity * 0.5
        
        # Apply fixed offset to far bound
        # Note: 10.0 is a fixed offset subtracted from the far bound
        t_far = t_max - 10.0
    else:
        # Fallback to raw intersection bounds
        t_near = t_min
        t_far = t_max
        
    return t_near, t_far
```

**Parameters:**
- `depthPriorLearned`: Boolean flag enabling the use of the depth prior. Default behavior (false) uses raw bounds.
- `0.5`: Scaling factor applied to the disparity value to compute the near bound offset.
- `10.0`: Fixed offset subtracted from the raw far bound to truncate the ray end.

**Edge Cases and Pitfalls:**
- **Disparity Accuracy:** If the depth prior is inaccurate (e.g., due to occlusion or learning errors), the truncated bounds may exclude valid volume data or include empty space, leading to visual artifacts or wasted computation.
- **Negative Bounds:** If $t_{near} > t_{far}$, the ray has no valid intersection range. The implementation should handle this by either skipping the ray or falling back to raw bounds.
- **Boundary Conditions:** The fixed offset `10.0` assumes a specific scene scale. In scenes with different scales, this value may need adjustment or normalization.

## Subtopics

### [Auxiliary Depth Prediction Head](7207-auxiliary-depth-prediction-head.md)
A depth prediction head appended after each scale encoding volume to predict depth maps and confidence scores, guiding sampling points and narrowing depth ranges for subsequent levels.

## Operating characteristics
- **Time Complexity:** Reduces the number of samples per ray by limiting the integration range, thus decreasing computational cost per pixel. The overhead of fetching the disparity map is negligible compared to volume evaluation.
- **Memory Complexity:** Requires storage for the depth prior (disparity map), which is typically per-pixel and low-precision (e.g., float16).
- **Strengths:** Effective at removing floating artifacts caused by sampling in empty space beyond the actual geometry.
- **Limitations:** Relies on the accuracy of the depth prior. If the prior is missing or incorrect, visual quality may degrade. The fixed offset `10.0` may not be universally applicable.
- **When to use:** When a reliable depth prior is available and floating artifacts are a concern.
- **When not to use:** When depth information is unavailable or unreliable, or when the scene scale differs significantly from the assumed offset.

## Background
This method is associated with the Clear-Plenoxels project, which aims to improve the visual quality of Plenoctrees by addressing artifacts through depth-aware sampling [S1, unofficial]. It represents a specific implementation strategy for leveraging learned depth information in volume rendering acceleration.

## Sources
[S1] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
