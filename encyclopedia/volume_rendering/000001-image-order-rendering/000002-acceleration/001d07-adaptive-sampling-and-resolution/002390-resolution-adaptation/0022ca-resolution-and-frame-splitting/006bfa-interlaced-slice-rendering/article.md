# Interlaced Slice Rendering

Also called: Odd-Even Slice Rendering

## Overview
Interlaced Slice Rendering is an image-order volume rendering optimization that halves the per-frame computational workload by rendering only half of the horizontal slices per frame, alternating between odd and even slices. It solves the problem of maintaining high frame rates in real-time volume rendering by doubling the effective frame rate perception while maintaining full spatial resolution over two frames, distinguishing itself from simple half-resolution rendering by preserving pixel detail when the camera is static or moving slowly.

## How it works
**Inputs and Outputs:**
- Input: A volume dataset, camera parameters, and a target frame time budget.
- Output: A rendered frame buffer.

**Data Structures:**
- Frame Buffer: A 2D array of pixels. The article does not specify bit depth, but notes 4 voxel colors with a brightness component [S1, unofficial].
- Render State: A boolean or integer flag indicating the current interlacing phase (odd/even) and a quality reduction factor (`reduceQuality`) [S1, unofficial].

**Steps:**
1. Determine the rendering mode based on camera velocity and frame time history.
2. If standard interlacing is active:
   - Calculate the set of slices to render. If the current frame index $N$ is even, render odd-numbered slices ($1, 3, 5, \dots$). If $N$ is odd, render even-numbered slices ($2, 4, 6, \dots$) [S1, unofficial].
   - Render the selected slices into the frame buffer.
   - Leave the unselected slices untouched from the previous frame [S1, unofficial].
3. If fast camera movement is detected (or adaptive quality triggers half-resolution mode):
   - Calculate only 120 slices (half the total, e.g., 240 slices) [S1, unofficial].
   - For each calculated slice, copy the pixel data twice to fill the full resolution height, effectively duplicating rows [S1, unofficial].
   - Apply dithering separately to the duplicated pixels to avoid interlacing artifacts and improve visual quality over simple blitting [S1, unofficial].
4. Display the frame buffer.

**Parameters:**
- Total Slices: Example implementation uses 240 slices [S1, unofficial].
- Target Frame Time: Hardcoded to 27 ms in the source, but can be dynamic [S1, unofficial].
- Reduce Quality Factor: An integer value (e.g., 20) determining the width of edge regions for adaptive quality reduction [S1, unofficial].

**Edge Cases and Pitfalls:**
- Fast Camera Movement: Causes visible banding artifacts because only half the pixels update per frame in standard interlacing mode. The method mitigates this by switching to half-resolution duplication with dithering when movement is rapid [S1, unofficial].
- Precision: The source does not specify floating-point precision, but notes shading is pre-computed per tile, not per pixel, in the specific implementation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_interlaced_slice(volume, camera, frame_buffer, frame_index, reduce_quality):
    # Determine if camera is moving fast [S1, unofficial]
    is_fast_movement = check_camera_velocity(camera)
    
    # Determine slice indices to render
    if not is_fast_movement:
        # Standard interlacing: render half the slices
        if frame_index % 2 == 0:
            slice_indices = [i for i in range(1, total_slices + 1) if i % 2 != 0]
        else:
            slice_indices = [i for i in range(1, total_slices + 1) if i % 2 == 0]
        
        # Render selected slices, leave others untouched
        for slice_idx in slice_indices:
            render_slice(volume, camera, slice_idx, frame_buffer)
            
    else:
        # Fast movement: half-resolution with duplication
        # Calculate only half the slices
        half_slices = total_slices // 2
        for i in range(half_slices):
            slice_idx = i + 1
            # Render the slice
            slice_data = render_slice_data(volume, camera, slice_idx)
            # Copy slice data twice to fill full resolution
            frame_buffer[2*i : 2*i+2, :] = slice_data
            
    return frame_buffer
```

## Subtopics

### Adaptive Quality with Edge Interlacing
A dynamic quality adjustment system that increases the interlacing factor (e.g., from 2-way to 3-way) at the screen edges when rendering time exceeds a target, reducing workload where visual attention is lowest [S1, unofficial]. It measures frame render time using `playdate->system->getElapsedTime()` and maintains a moving average over multiple frames [S1, unofficial]. Every four frames, it compares the moving average to the desired time (27 ms); if the average is higher, it increases `reduceQuality`, and if lower, it decreases it [S1, unofficial]. The reduced quality is applied by performing 3-way interlacing (or tripling slices in fast mode) only in the left-most and right-most slices, proportional to `reduceQuality` (e.g., 20 slices on each side) [S1, unofficial]. This mitigates visible banding artifacts by restricting them to screen edges where user attention is lower [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Reduces per-frame rendering work by approximately 50% in standard interlaced mode, as only half the slices are computed [S1, unofficial].
- **Memory Complexity:** Requires storage for the full frame buffer, but no additional significant memory for the interlacing logic itself [S1, unofficial].
- **Measured Performance:** In the source implementation, the method allows for a perceived 30 fps frame rate while only computing 120 slices per frame (out of 240) [S1, unofficial]. The target frame time is 27 ms [S1, unofficial].
- **Scaling:** The method scales with the number of slices; halving the slices per frame halves the computational load regardless of total slice count [S1, unofficial].
- **Strengths:** Maintains full spatial resolution over two frames; improves perceived frame rate; adaptive quality allows dynamic adjustment to hardware constraints [S1, unofficial].
- **Limitations:** Fast camera movement causes banding artifacts in standard mode; requires dithering in half-resolution mode which is slightly slower than simple blitting [S1, unofficial].
- **When to Use:** Real-time volume rendering where perceived smoothness is prioritized over instantaneous pixel accuracy, and camera movement is generally slow to moderate [S1, unofficial].
- **When Not to Use:** When fast camera movement is frequent and banding artifacts are unacceptable, or when full instantaneous resolution is required [S1, unofficial].

## Background
The method was described by Thomas in a devlog post on the Playdate Developer Forum in 2024, as part of a side project building a 3D engine for the Playdate console [S1, unofficial]. The implementation uses a map of 128x128 tiles, where each tile is a voxel block with a base of 32x32 pixels and a maximum height of 28, using four voxel colors with dynamic brightness for shadows [S1, unofficial]. Shading is pre-computed once per tile during startup, not in real-time [S1, unofficial].

## Sources
[S1] Building yet another 3D engine -- a devlog, of sorts - Playdate Developer Forum, https://devforum.play.date/t/building-yet-another-3d-engine-a-devlog-of-sorts/19197, fetched 2026-10-29 (unofficial)
