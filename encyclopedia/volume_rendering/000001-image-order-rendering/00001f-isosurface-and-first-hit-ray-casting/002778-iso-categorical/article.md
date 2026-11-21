# iso_categorical
## Overview
`iso_categorical` is a volume rendering method for categorical label datasets that replaces fixed-step sampling with exact voxel traversal to eliminate sawtooth artifacts at grazing angles [S1, unofficial]. It sets itself apart from its siblings by ensuring every intersected voxel is visited, making the result independent of the sampling step size and preventing rays from missing thin structures or voxel corners between samples [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A categorical volume dataset (integer labels), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), and clipping planes [S1, unofficial].
- Output: The color of the first non-transparent label encountered, and the depth of the entry point into that label's voxel [S1, unofficial].

**Data Structures:**
- The volume is stored as a 3D grid of integer labels [G].
- Clipping planes are defined in world space as pairs of points and outward normals [G].
- The volume object provides `origin` (3D vector), `spacing` (3D vector), `max_depth` (float), `get_label(voxel)` (int), `get_label_color(label)` (color), and `background_color` (color) [G].

**Algorithm Steps:**
1. **Ray Definition:** Define the view ray $r(t) = \mathbf{o} + t\mathbf{d}$ for the current pixel [G].
2. **Initial March:** Advance along the ray using a fixed step size (default `relative_step_size` = 0.8 voxel) until the first sample point is not clipped by the viewing frustum or clipping planes [S1, unofficial].
3. **Exact Voxel Traversal:** From the first valid sample point, perform an exact voxel traversal using the Amanatides & Woo algorithm [S1, unofficial]. This walks through every voxel the ray passes through in order, ensuring no voxel is skipped regardless of the step size [S1, unofficial].
4. **Ray Shortening:** Shorten the ray segment by a tiny margin at both ends to prevent rounding errors from reading neighboring voxels when a clipping plane starts exactly on a voxel boundary [S1, unofficial].
5. **Label Detection:** During traversal, check each voxel for a non-transparent label. Stop at the first voxel containing a non-transparent label [S1, unofficial].
6. **Clipping Evaluation:** Evaluate clipping planes at the exact entry and exit points of each voxel to ensure cut surfaces remain in their correct positions [S1, unofficial].
7. **Depth Calculation:** Set the pixel's depth to the point where the ray enters the hit voxel, expressed in data coordinates. This avoids the previous error where depth was shifted by half a voxel along each axis, causing incorrect occlusion between layers with different scales [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_iso_categorical(ray_origin, ray_direction, volume, clipping_planes, step_size=0.8):
    # 1. Initial march to find first unclipped sample
    t = 0.0
    while t < volume.max_depth:
        pos = ray_origin + t * ray_direction
        if any(dot(pos - plane_point, plane_normal) < 0 for plane_point, plane_normal in clipping_planes):
            t += step_size
            continue
        break
    
    # 2. Exact voxel traversal (Amanatides & Woo)
    epsilon = 1e-6
    t_end = volume.max_depth - epsilon
    
    # Initialize Amanatides & Woo state at current t
    voxel = floor((ray_origin + t * ray_direction - volume.origin) / volume.spacing)
    t_max = (volume.origin + (voxel + 0.5) * volume.spacing - (ray_origin + t * ray_direction)) / ray_direction
    t_delta = volume.spacing / ray_direction
    
    current_t = t
    while current_t < t_end:
        # Check current voxel for non-transparent label
        label = volume.get_label(voxel)
        if label != 0:
            entry_point = ray_origin + current_t * ray_direction
            if all(dot(entry_point - plane_point, plane_normal) >= 0 for plane_point, plane_normal in clipping_planes):
                depth = (entry_point - volume.origin) / volume.spacing
                return volume.get_label_color(label), depth
        
        # Advance to next voxel
        axis = argmin(t_max)
        current_t = t_max[axis]
        t_max[axis] += t_delta[axis]
        voxel[axis] += 1
    
    return volume.background_color, None
```

**Parameters:**
- `relative_step_size`: Default 0.8 voxel. Used only for the initial march to find the first unclipped sample; the final traversal result is independent of this value [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Sawtooth Edges:** The old method caused sawtooth artifacts at grazing angles because rays clipping voxel corners between samples missed them. This new method eliminates this by visiting every voxel [S1, unofficial].
- **Depth Offset Error:** The old method shifted depth by half a voxel, causing incorrect occlusion for layers with different scales. The new method uses the exact entry point in data coordinates [S1, unofficial].
- **Rounding Errors:** Clipping planes exactly on voxel boundaries can cause reads from neighboring voxels. Shortening the ray by a tiny margin prevents this [S1, unofficial].

## Subtopics

### [translucent_categorical](6031-translucent-categorical.md)
A variant of `iso_categorical` used when `rendering='translucent'`. It shares the exact voxel traversal logic and clipping evaluation but differs in how the hit voxel is shaded (translucent shading instead of opaque). Performance impact is similar to `iso_categorical`, with frame time increases ranging from 1.1x to 1.8x depending on scene density and view angle.

## Operating characteristics
- **Time Complexity:** The traversal visits every voxel along the ray. This costs more per visit than the old sample loop, especially at oblique angles where a ray crosses more voxels than it took samples [S1, unofficial].
- **Memory Complexity:** The method requires no additional memory structures beyond the volume data and clipping plane definitions [S1, unofficial].
- **Measured Performance:** On an M1 Max (canvas ~2000x1800 px, iso_categorical, default step 0.8):
  - Empty 256³ volume: 3.6 ms (main) → 4.3 ms (new) in axis-aligned view; 3.7 ms → 6.3 ms in oblique view [S1, unofficial].
  - 300 nuclei, 256³: 3.1 ms → 3.4 ms (axis-aligned); 3.7 ms → 4.6 ms (oblique) [S1, unofficial].
  - 1500 nuclei, 512³: 6.7 ms → 7.2 ms (axis-aligned); 8.1 ms → 9.8 ms (oblique) [S1, unofficial].
- **Strengths:** Eliminates sawtooth artifacts; depth is accurate across different scales; result is independent of `relative_step_size` [S1, unofficial].
- **Limitations:** Higher computational cost per ray than fixed-step sampling, particularly at oblique angles [S1, unofficial].
- **When to Use:** When rendering categorical label data (e.g., segmented anatomical features) where visual accuracy at grazing angles and correct depth ordering are critical [S1, unofficial].
- **When Not to Use:** When performance is paramount and slight visual artifacts are acceptable, or when rendering non-categorical scalar data (use standard volume raycasting) [S1, unofficial].

## Background
`iso_categorical` was developed as part of the napari project to fix rendering artifacts in 3D labels layers [S1, unofficial]. It was applied to the vispy library (PR #2807) to provide a robust solution for categorical volume rendering [S1, unofficial]. The method addresses specific issues in the previous implementation where fixed-step sampling led to visual inaccuracies [S1, unofficial].

## Sources
[S1] Fix sawtooth edges and depth of iso_categorical labels rendering, https://github.com/napari/napari/pull/9674, fetched 2026-10-25 (unofficial)
