# First Depth Ray Traversal

Also called: First Depth, First Intensity Ray Traversal, Tuy & Tuy traversal

## Overview
First Depth Ray Traversal is a ray traversal method that extracts iso-surfaces by recording the first scalar intensity encountered along a viewing ray [S1, unofficial][S2, unofficial]. It is distinguished from sibling methods like Maximum Intensity Projection (MIP) or Accumulate compositing by its immediate termination upon the first hit, making it specifically suited for isolating high-density or specific threshold structures without contribution from deeper volumes [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 3D scalar volume $V$, a viewing ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, a threshold $\tau$, and a step size $\Delta t$ [G].
*   Output: The color and depth of the first voxel where the scalar value meets the selection criteria [G].

**Data Structures:**
*   The volume is typically stored as a 3D array of scalar values [G].
*   Ray traversal uses a step size $\Delta t$ corresponding to the voxel grid spacing [G].

**Steps:**
1.  Initialize ray origin $\mathbf{o}$ and direction $\mathbf{d}$ for the pixel [G].
2.  Iterate along the ray from $t=0$ to the volume exit [G].
3.  At each step, sample the scalar value $s = V(\mathbf{r}(t))$ [G].
4.  Check if $s$ satisfies the condition (e.g., $s \ge \tau$ or first non-background value) [G].
5.  If the condition is met: record the current position $\mathbf{p} = \mathbf{r}(t)$ and value $s$, then terminate the ray traversal immediately [G].
6.  If the loop completes without a hit, assign background color [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def first_depth_traversal(ray_origin, ray_direction, volume, threshold, voxel_spacing):
    t = 0.0
    max_t = volume.max_distance
    background_color = [0.0, 0.0, 0.0, 0.0]
    
    while t <= max_t:
        pos = ray_origin + ray_direction * t
        if not volume.bounds_check(pos):
            t += voxel_spacing
            continue
        scalar_value = volume.sample(pos)
        if scalar_value >= threshold:
            color = volume.transfer_function(scalar_value)
            return color + [1.0]
        t += voxel_spacing
    return background_color
```

**Parameters:**
*   Threshold $\tau$: Defines the iso-surface or minimum intensity [G]. Typical values depend on the data normalization (e.g., 0.5 for normalized [0,1] data) [G].
*   Step size $\Delta t$: Often equal to voxel spacing for exact voxel traversal, or smaller for sub-voxel precision [G].

**Edge Cases:**
*   If no voxel exceeds the threshold, the ray returns background [G].
*   Precision errors at boundaries may cause missed hits if step size is not aligned with voxel centers [G].

## Operating characteristics
*   **Time Complexity:** $O(k)$ where $k$ is the number of voxels traversed until the first hit [G]. In sparse volumes, this is significantly faster than full compositing [G]. In dense volumes, it may still require traversing the entire ray length if the hit is deep or absent [G].
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray, as it does not accumulate opacity/color buffers [G].
*   **Strengths:** Fast extraction of specific structures; no need for transfer function tuning for opacity accumulation; ideal for high-contrast features [G].
*   **Limitations:** Loses depth information for structures behind the first hit; sensitive to noise (first noisy voxel may be selected); does not show internal structure [G].

## Background
First Depth Ray Traversal was introduced by Tuy and Tuy in 1984 [S1, unofficial][S2, unofficial]. It is one of the earliest methods for ray-casting based volume rendering, specifically designed for iso-surface extraction via ray traversal [S1, unofficial][S2, unofficial].

## Sources
[S1] PPT - Direct Volume Rendering (DVR): Ray-casting PowerPoint Presentation - ID:465503, https://slideserve.com/jariath/direct-volume-rendering-dvr-ray-casting, fetched 2026-10-24 (unofficial)
[S2] Volume Rendering Volume Rendering. Lecture 21. Acknowledgements. These slides are collected from many sources. A particularly valuable source is the IEEE Visualization conference tutorials. Sources fr, https://www.slideserve.com/hayes-levine/volume-rendering-powerpoint-ppt-presentation, fetched 2026-10-25 (unofficial)
