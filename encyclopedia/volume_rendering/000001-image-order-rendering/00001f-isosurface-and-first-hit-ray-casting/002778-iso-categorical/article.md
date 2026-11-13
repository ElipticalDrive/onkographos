# iso_categorical


## Overview
`iso_categorical` is a volume rendering method designed for categorical label datasets (e.g., segmented anatomical structures) that replaces fixed-step sampling with exact voxel traversal to eliminate sawtooth artifacts at grazing angles [S1, unofficial]. It sets itself apart from its siblings by ensuring that every voxel intersected by a ray is visited, making the result independent of the sampling step size and preventing rays from missing thin structures or voxel corners between samples [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A categorical volume dataset (integer labels), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), and clipping planes.
- Output: The color of the first non-transparent label encountered, and the depth of the entry point into that label's voxel.

**Data Structures:**
- The volume is stored as a 3D grid of integer labels.
- Clipping planes are defined in world space.

**Algorithm Steps:**
1. **Ray Definition:** Define the view ray $r(t) = \mathbf{o} + t\mathbf{d}$ for the current pixel.
2. **Initial Traversal:** March along the ray using a fixed step size (default `relative_step_size` = 0.8 voxel) until the first sample point that is not clipped by the viewing frustum or clipping planes is found [S1, unofficial].
3. **Exact Voxel Traversal:** From the first valid sample point, perform an exact voxel traversal using the Amanatides & Woo algorithm [S1, unofficial]. This algorithm walks through every voxel the ray passes through in order, ensuring no voxel is skipped regardless of the step size [S1, unofficial].
4. **Shortening Ray:** Shorten the ray segment by a tiny margin at both ends to prevent rounding errors from reading neighboring voxels when a clipping plane starts exactly on a voxel boundary [S1, unofficial].
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
        if is_clipped(pos, clipping_planes):
            t += step_size
            continue
        break
    
    # 2. Exact voxel traversal (Amanatides & Woo)
    # Shorten ray ends to avoid boundary rounding errors
    t_start = t + epsilon
    t_end = volume.max_depth - epsilon
    
    # Initialize traversal state for Amanatides & Woo
    voxel = get_voxel_index(pos)
    t_max_x, t_max_y, t_max_z = compute_t_max(voxel, ray_direction)
    t_delta_x, t_delta_y, t_delta_z = compute_t_delta(voxel, ray_direction)
    
    while t < t_end:
        # Check current voxel for non-transparent label
        label = volume.get_label(voxel)
        if label is not transparent:
            # 3. Evaluate clipping planes at entry point
            entry_point = ray_origin + t * ray_direction
            if is_inside_clipping_planes(entry_point, clipping_planes):
                # 4. Set depth to entry point in data coordinates
                depth = voxel_to_data_coords(voxel)
                return get_label_color(label), depth
        
        # Advance to next voxel
        t, voxel = advance_amanatides_woo(
            t, voxel, 
            t_max_x, t_max_y, t_max_z, 
            t_delta_x, t_delta_y, t_delta_z
        )
    
    return background_color, None
```

**Parameters:**
- `relative_step_size`: Default 0.8 voxel. Used only for the initial march to find the first unclipped sample; the final result is independent of this value [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Sawtooth Edges:** The old method caused sawtooth artifacts at grazing angles because rays clipping voxel corners between samples missed them. This new method eliminates this by visiting every voxel [S1, unofficial].
- **Depth Offset Error:** The old method shifted depth by half a voxel, causing incorrect occlusion for layers with different scales. The new method uses the exact entry point in data coordinates [S1, unofficial].
- **Rounding Errors:** Clipping planes exactly on voxel boundaries can cause reads from neighboring voxels. Shortening the ray by a tiny margin prevents this [S1, unofficial].

## Subtopics

### translucent_categorical
A variant of `iso_categorical` used when `rendering='translucent'`. It shares the exact voxel traversal logic and clipping evaluation but differs in how the hit voxel is shaded (translucent shading instead of opaque) [S1, unofficial]. Performance impact is similar to `iso_categorical`, with frame time increases ranging from 1.1x to 1.8x depending on scene density and view angle [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The traversal visits every voxel along the ray. This costs more per visit than the old sample loop, especially at oblique angles where a ray crosses more voxels than it took samples [S1, unofficial].
- **Memory Complexity:** No additional memory structures are required beyond the volume data and clipping plane definitions [S1, unofficial].
- **Measured Performance:** On an M1 Max (canvas ~2000x1800 px, iso_categorical, default step 0.8):
  - Empty 256³ volume: 3.6 ms (main) → 4.3 ms (new) in axis-aligned view; 3.7 ms → 6.3 ms in oblique view [S1, unofficial].
  - 300 nuclei, 256³: 3.1 ms → 3.4 ms (axis-aligned); 3.7 ms → 4.6 ms (oblique) [S1, unofficial].
  - 1500 nuclei, 512³: 6.7 ms → 7.2 ms (axis-aligned); 8.1 ms → 9.8 ms (oblique) [S1, unofficial].
  - Dense labels (rays stop immediately): <1 ms [S1, unofficial].
- **Strengths:** Eliminates sawtooth artifacts; depth is accurate across different scales; result is independent of `relative_step_size` [S1, unofficial].
- **Limitations:** Higher computational cost per ray than fixed-step sampling, particularly at oblique angles [S1, unofficial].
- **When to Use:** When rendering categorical label data (e.g., segmented anatomical features) where visual accuracy at grazing angles and correct depth ordering are critical [S1, unofficial].
- **When Not to Use:** When performance is paramount and slight visual artifacts are acceptable, or when rendering non-categorical scalar data (use standard volume raycasting) [S1, unofficial].

## Background
`iso_categorical` was developed as part of the napari project to fix rendering artifacts in 3D labels layers. It was introduced in a pull request to the napari repository (PR #9674) and subsequently applied to the vispy library (PR #2807) to provide a robust solution for categorical volume rendering [S1, unofficial]. The method addresses specific issues in the previous implementation where fixed-step sampling led to visual inaccuracies [S1, unofficial].

## Sources
[S1] Fix sawtooth edges and depth of iso_categorical labels rendering, https://github.com/napari/napari/pull/9674, fetched 2026-10-25 (unofficial)
