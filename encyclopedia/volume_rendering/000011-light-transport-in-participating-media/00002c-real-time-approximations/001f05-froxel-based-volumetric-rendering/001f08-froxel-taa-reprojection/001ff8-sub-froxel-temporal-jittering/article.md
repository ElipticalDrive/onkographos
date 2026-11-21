# Sub-Froxel Temporal Jittering

Also called: 1D Froxel Jittering, Sub-Froxel Jittering, Temporal Supersampling Jitter

## Overview
Sub-Froxel Temporal Jittering is a technique for real-time volumetric rendering that offsets the sampling position within a froxel along the camera's depth axis to enable temporal supersampling and anti-aliasing [S1, unofficial]. It distinguishes itself from full 3D jittering by restricting movement to the z-axis, thereby reducing spatial variance in the x and y directions to improve temporal stability when accumulating history [S1, unofficial].

## How it works
**Inputs:**
- Reprojection vector describing camera movement from previous to current frame [S1, unofficial].
- A pre-calculated low-discrepancy sequence of jitter offsets [S1, unofficial].
- Froxel center position in camera space [S1, unofficial].

**Outputs:**
- Jittered sampling position for the current frame's froxel grid [S1, unofficial].

**Data Structures:**
- **Jitter Sequence:** A 1D array of floating-point offsets for depth-only jittering, or a 3D array/list for full 3D jittering [S1, unofficial]. Commonly uses Halton(2,3) of length 8 or 16, or Halton(2,3,5) of length 16 [S1, unofficial].
- **Reprojection Vector:** A vector describing camera translation and rotation between frames [S1, unofficial]. It is constant for all froxels in a frame, as froxels represent static world-space positions [S1, unofficial].

**Steps:**
1. Center the sampling point at the middle of the current froxel in camera space [S1, unofficial].
2. Acquire a per-frame sub-froxel jitter offset from a pre-defined low-discrepancy sequence [S1, unofficial]. Each froxel uses the same per-frame offset [S1, unofficial].
3. Check reprojection validity: If the reprojection vector points outside the history buffer, skip jittering and use the centered sampling position directly, as jittering is only useful when blending with history [S1, unofficial].
4. Apply the jitter offset to the sampling position within the froxel [S1, unofficial]. For depth-only jittering, apply the offset only to the z-axis (camera forward direction), remaining centered in the x and y directions [S1, unofficial].
5. Evaluate participating media density and in-scattered radiance at the jittered sampling position in the compute shader [S1, unofficial].
6. Blend the current frame's sample with the reprojected history sample using temporal accumulation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-calculated Halton(2,3) sequence of length 16 for 1D depth jitter
HALTON_1D = [0.5, 0.25, 0.75, 0.125, 0.625, 0.375, 0.875, 0.0625, 
             0.5625, 0.3125, 0.8125, 0.0625, 0.5625, 0.3125, 0.8125, 0.0625]

def sub_froxel_jittering(froxel_center_camera_space, froxel_depth, frame_index, reprojection_valid):
    """
    froxel_center_camera_space: vec3 (center of froxel in camera space)
    froxel_depth: float (size of froxel along z-axis)
    frame_index: int (current frame number)
    reprojection_valid: bool (true if reprojection vector points inside history buffer)
    """
    if not reprojection_valid:
        # If reprojection fails, no sub-froxel temporal jittering is applied [S1, unofficial]
        return froxel_center_camera_space

    # Acquire per-frame sub-froxel jitter offset from pre-defined set [S1, unofficial]
    jitter_index = frame_index % 16
    jitter_value = HALTON_1D[jitter_index]

    # Center the sampling point in the middle of the froxel, then offset only along depth [S1, unofficial]
    # [G] Standard sub-froxel jitter maps [0, 1] to [-0.5, 0.5] relative to froxel size.
    offset_z = (jitter_value - 0.5) * froxel_depth

    # Apply jitter offset only to the z-axis, remaining centered in x and y [S1, unofficial]
    jittered_position = froxel_center_camera_space
    jittered_position.z += offset_z

    return jittered_position
```

**Parameters:**
- Sequence Type: Halton(2,3) for 1D, Halton(2,3,5) for 3D [S1, unofficial].
- Sequence Length: 8 or 16 [S1, unofficial].
- Jitter Axis: Z-axis (depth) only for 1D jittering [S1, unofficial].

**Edge Cases:**
- Reprojection Failure: If the reprojection vector points outside the history buffer, jittering is skipped and the current frame sample is returned directly [S1, unofficial].
- View-Dependent Phase Functions: Highly view-dependent phase functions can cause artifacts in reprojected history data, though this is noted as out of scope for the primary method description [S1, unofficial].

## Operating characteristics
- Temporal Stability: High. Restricting jitter to the z-axis minimizes spatial variance in x and y, which is critical for stable temporal accumulation [S1, unofficial].
- Performance: Low overhead. The jitter offset is pre-calculated and passed as a shader parameter, requiring no additional per-froxel computation beyond the offset addition [S1, unofficial].
- Memory: Minimal. Requires storing a small pre-calculated sequence (e.g., 16 floats) [S1, unofficial].
- Strengths: Reduces flickering and aliasing in static participating media without introducing significant spatial noise [S1, unofficial].
- Limitations: Only effective when reprojection is valid. Does not address view-dependent phase function artifacts [S1, unofficial].

## Background
This technique is used in the Frostbite engine [S1, unofficial]. It was adapted for froxel-based rendering to provide temporal anti-aliasing for in-scattered radiance, building on the concept of temporal supersampling [S1, unofficial]. The method was analyzed in the context of real-time volumetric rendering, specifically for static participating media [S1, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
