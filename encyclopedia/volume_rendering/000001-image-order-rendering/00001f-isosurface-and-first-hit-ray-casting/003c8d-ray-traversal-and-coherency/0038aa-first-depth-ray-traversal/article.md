# First Depth Ray Traversal

Also called: First Intensity Ray Traversal, Tuy & Tuy traversal

## Overview
First Depth Ray Traversal is a ray traversal method that extracts iso-surfaces by recording the first scalar intensity encountered along a viewing ray [S1, unofficial][S2, unofficial]. It is distinguished from sibling methods like Maximum Intensity Projection (MIP) or Accumulate compositing by its immediate termination upon the first hit, making it specifically suited for isolating high-density or specific threshold structures without contribution from deeper volumes [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 3D scalar volume $V$, a viewing ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, and a transfer function or threshold $\tau$.
*   Output: The color and depth of the first voxel where the scalar value meets the selection criteria.

**Data Structures:**
*   The volume is typically stored as a 3D array of scalar values.
*   Ray traversal uses a step size $\Delta t$ corresponding to the voxel grid spacing.

**Steps:**
1.  Initialize ray origin $\mathbf{o}$ and direction $\mathbf{d}$ for the pixel.
2.  Iterate along the ray from $t=0$ to the volume exit.
3.  At each step, sample the scalar value $s = V(\mathbf{r}(t))$.
4.  Check if $s$ satisfies the condition (e.g., $s \ge \tau$ or first non-background value).
5.  If the condition is met:
    *   Record the current position $\mathbf{p} = \mathbf{r}(t)$ and value $s$.
    *   Terminate the ray traversal immediately.
6.  If the loop completes without a hit, assign background color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def first_depth_traversal(ray_origin, ray_direction, volume, threshold):
    t = 0.0
    step_size = 1.0 / volume.resolution  # Simplified step
    max_t = volume.max_distance
    
    while t < max_t:
        # Sample scalar value at current ray position
        # Note: In practice, interpolation (e.g., trilinear) is used
        scalar_value = volume.sample(ray_origin + ray_direction * t)
        
        # Check for first hit meeting criteria
        if scalar_value >= threshold:
            # Return the first encountered intensity/position
            return scalar_value, t
        
        t += step_size
        
    # No hit found
    return background_color, -1.0
```

**Parameters:**
*   Threshold $\tau$: Defines the iso-surface or minimum intensity. Typical values depend on the data normalization (e.g., 0.5 for normalized [0,1] data).
*   Step size: Often equal to voxel spacing for exact voxel traversal, or smaller for sub-voxel precision.

**Edge Cases:**
*   If no voxel exceeds the threshold, the ray returns background.
*   Precision errors at boundaries may cause missed hits if step size is not aligned with voxel centers.

## Operating characteristics
*   **Time Complexity:** $O(k)$ where $k$ is the number of voxels traversed until the first hit. In sparse volumes, this is significantly faster than full compositing. In dense volumes, it may still require traversing the entire ray length if the hit is deep or absent.
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray, as it does not accumulate opacity/color buffers.
*   **Strengths:** Fast extraction of specific structures; no need for transfer function tuning for opacity accumulation; ideal for high-contrast features.
*   **Limitations:** Loses depth information for structures behind the first hit; sensitive to noise (first noisy voxel may be selected); does not show internal structure.

## Background
First Depth Ray Traversal was introduced by Tuy and Tuy in 1984 [S1, unofficial][S2, unofficial]. It is one of the earliest methods for ray-casting based volume rendering, specifically designed for iso-surface extraction via ray traversal. It is often contrasted with later methods like Maximum Intensity Projection (MIP) and Accumulate compositing (Levoy '88) which consider the entire ray path [S1, unofficial][S2, unofficial].

## Sources
[S1] PPT - Direct Volume Rendering (DVR): Ray-casting PowerPoint Presentation - ID:465503, https://slideserve.com/jariath/direct-volume-rendering-dvr-ray-casting, fetched 2026-10-24 (unofficial)
[S2] Volume Rendering Volume Rendering. Lecture 21. Acknowledgements. These slides are collected from many sources. A particularly valuable source is the IEEE Visualization conference tutorials. Sources fr, https://www.slideserve.com/hayes-levine/volume-rendering-powerpoint-ppt-presentation, fetched 2026-10-25 (unofficial)
