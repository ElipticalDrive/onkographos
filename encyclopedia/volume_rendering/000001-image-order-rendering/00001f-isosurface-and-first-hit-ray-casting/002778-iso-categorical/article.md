# iso_categorical

Also called: categorical isosurface rendering

## Overview
`iso_categorical` is a volume rendering method designed for categorical label datasets (e.g., segmented anatomical structures) that replaces fixed-step sampling with exact voxel traversal to eliminate sawtooth artifacts at grazing angles [S1, unofficial]. Unlike standard isosurface methods that rely on scalar field thresholds, this method tests discrete integer labels, stopping at the first non-transparent voxel encountered along the ray [S1, unofficial]. It sets itself apart from its siblings by ensuring that every voxel the ray physically intersects is visited, making the result independent of the sampling step size and preventing rays from skipping thin voxel corners [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D categorical label volume (integer grid), camera parameters (origin, direction), clipping planes, and a transparency mask for labels.
- Output: A pixel color (from the hit label) and a depth value (in data coordinates).

**Data Structures:**
- Volume: A 3D array of integers representing labels. Dimensions are $N_x \times N_y \times N_z$.
- Ray: Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Clipping Planes: A set of planes defining the visible volume boundaries.

**Algorithm Steps:**
1.  **Ray Construction:** Construct the view ray for the current pixel from the camera center through the pixel center [S1, unofficial].
2.  **Initial Marching (Coarse):** March along the ray using a fixed step size (default `relative_step_size` = 0.8 voxels) to find the first sample point that is inside the volume bounds and not clipped by the near/far clipping planes [S1, unofficial]. If no such point is found, the pixel is background.
3.  **Exact Voxel Traversal (Fine):** From the first valid sample point, perform an exact voxel traversal using the Amanatides & Woo algorithm [S1, unofficial]. This algorithm walks through every voxel the ray passes through in order, ensuring no voxel is skipped, even if the ray only clips a corner or edge [S1, unofficial].
4.  **Clipping Evaluation:** For each voxel visited, evaluate the entry and exit points of the ray relative to any active clipping planes. If the ray segment within the voxel is entirely outside the clipping volume, skip it [S1, unofficial].
5.  **Label Test:** For each valid voxel segment, read the label value. If the label is non-transparent (and not background/empty), this is the hit voxel.
6.  **Ray Shortening (Boundary Guard):** To prevent rounding errors from reading neighboring voxels when a clipping plane starts exactly on a voxel boundary, shorten the ray segment by a tiny margin $\epsilon$ at both the entry and exit points of the voxel [S1, unofficial].
7.  **Depth Calculation:** Set the pixel's depth to the point where the ray *enters* the hit voxel, expressed in data coordinates (voxel indices). This avoids the half-voxel offset error present in older implementations where depth was calculated as `iloc * u_shape` [S1, unofficial].
8.  **Shading:** Compute the color based on the hit label. The specific shading model (e.g., flat, gradient) depends on the rendering mode (see `translucent_categorical`) [S1, unofficial].
9.  **Termination:** Stop traversal for this pixel. Do not process voxels behind the hit surface [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iso_categorical_render(ray_origin, ray_direction, volume, clipping_planes, epsilon=1e-6):
    """
    Renders a single pixel using iso_categorical method.
    Returns (color, depth).
    """
    # Step 2: Initial coarse march to find start
    step_size = 0.8  # relative_step_size in voxels
    current_pos = ray_origin
    max_steps = 1000 # Safety limit
    
    # Find first unclipped sample
    for _ in range(max_steps):
        if not is_inside_volume(current_pos):
            return background_color, 0.0
        if is_clipped(current_pos, clipping_planes):
            current_pos += ray_direction * step_size
            continue
        break
    else:
        return background_color, 0.0

    # Step 3: Exact voxel traversal
    # Initialize Amanatides & Woo traversal state
    voxel_traversal = AmanatidesWooTraversal(current_pos, ray_direction, volume.shape)
    
    while voxel_traversal.has_next():
        voxel_coords = voxel_traversal.next_voxel()
        
        # Get entry and exit points of the ray within this voxel
        t_entry, t_exit = voxel_traversal.get_t_values()
        
        # Step 4 & 6: Clipping and Ray Shortening
        # Shorten ray segment to avoid boundary rounding errors
        t_entry += epsilon
        t_exit -= epsilon
        
        if t_entry >= t_exit:
            continue
            
        # Check if the shortened segment is clipped
        if is_clipped_segment(t_entry, t_exit, ray_origin, ray_direction, clipping_planes):
            continue
            
        # Step 5: Label Test
        label = volume[voxel_coords]
        if label is not transparent_label:
            # Hit found
            # Step 7: Depth is entry point in data coordinates
            hit_pos = ray_origin + ray_direction * t_entry
            depth = voxel_to_data_coords(hit_pos)
            
            # Step 8: Shading (simplified)
            color = get_label_color(label)
            return color, depth
            
    return background_color, 0.0
```

**Parameters:**
- `relative_step_size`: Default 0.8 voxels. Used only for the initial coarse march to find the first valid sample. The final result is independent of this value [S1, unofficial].
- `epsilon`: Tiny margin for ray shortening. Value not specified in sources [G].

**Edge Cases and Pitfalls:**
- **Sawtooth Edges:** In the old fixed-step method, rays clipping voxel corners between samples were missed, causing shifting sawtooth artifacts at grazing angles. This is eliminated by visiting every voxel [S1, unofficial].
- **Depth Offset Error:** The old method shifted depth by half a voxel along each axis, causing incorrect occlusion between layers with different scales. The new method uses the voxel entry point in data coordinates, fixing this [S1, unofficial].
- **Clipping Plane Boundaries:** Without ray shortening, rounding errors could cause the shader to read neighboring voxels when a clipping plane aligns exactly with a voxel boundary [S1, unofficial].

## Subtopics

### translucent_categorical
A variant of `iso_categorical` used when `rendering='translucent'`. It shares the exact voxel traversal logic and depth calculation of `iso_categorical` but differs in the shading model applied to the hit voxel [S1, unofficial]. It is chosen when semi-transparent rendering of categorical labels is required. Performance impact is similar to `iso_categorical`, with frame time increases ranging from 1.1x to 1.8x depending on scene density and view angle [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: The traversal visits every voxel along the ray. This is more expensive per visit than the old fixed-step sampling, especially at oblique angles where a ray crosses more voxels than it took samples [S1, unofficial]. The cost is independent of `relative_step_size` [S1, unofficial].
- Memory: No additional memory structures are required beyond the volume data and traversal state [S1, unofficial].

**Measured Performance:**
- Hardware: Apple M1 Max, canvas ~2000 × 1800 px [S1, unofficial].
- Scene: 256³ volume, empty or with 300/1500 nuclei [S1, unofficial].
- Results:
    - Empty 256³, axis-aligned: 3.6 ms → 4.3 ms (1.2×) [S1, unofficial].
    - Empty 256³, oblique: 3.7 ms → 6.3 ms (1.7×) [S1, unofficial].
    - 300 nuclei, 256³, axis-aligned: 3.1 ms → 3.4 ms (1.1×) [S1, unofficial].
    - 300 nuclei, 256³, oblique: 3.7 ms → 4.6 ms (1.3×) [S1, unofficial].
    - 1500 nuclei, 512³, axis-aligned: 6.7 ms → 7.2 ms (1.1×) [S1, unofficial].
    - 1500 nuclei, 512³, oblique: 8.1 ms → 9.8 ms (1.2×) [S1, unofficial].
- Dense labels (rays stop immediately) render in under 1 ms [S1, unofficial].
- Compared to the old method with a finer step (0.4), the new method is as fast or faster (0.66× to 1.0×) [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Eliminates sawtooth artifacts; depth is accurate across different scales; result is independent of sampling step size [S1, unofficial].
- Limitations: Higher computational cost per pixel due to voxel traversal, especially at oblique angles; performance degrades with sparse data where rays traverse many empty voxels [S1, unofficial].

**When to Use:**
- Use for categorical label rendering where visual accuracy at grazing angles is critical.
- Do not use for scalar field isosurfaces (use `iso_scalar` variants) or when performance is paramount and artifacts are acceptable.

## Background
`iso_categorical` was developed as part of the napari project to fix rendering artifacts in 3D label layers. It was implemented in a pull request to the napari repository (PR #9674) and subsequently ported to the vispy library (PR #2807) [S1, unofficial]. The method addresses limitations in the previous fixed-step sampling approach used in `iso_categorical` and `translucent_categorical` within `src/napari/_vispy/visuals/volume.py` [S1, unofficial].

## Sources
[S1] Fix sawtooth edges and depth of iso_categorical labels rendering, https://github.com/napari/napari/pull/9674, fetched 2026-10-25 (unofficial)
