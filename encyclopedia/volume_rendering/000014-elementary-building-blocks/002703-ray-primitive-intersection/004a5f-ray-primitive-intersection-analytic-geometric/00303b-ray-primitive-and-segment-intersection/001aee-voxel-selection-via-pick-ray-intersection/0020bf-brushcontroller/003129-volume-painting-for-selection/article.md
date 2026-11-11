# Volume Painting for Selection

Also called: Volumetric Brush Selection

## Overview
Volume Painting for Selection is a 3D volumetric painting approach for selection definition where a user clicks on the image plane, casting a ray into the volume to find the first non-transparent voxel, then drawing a 3D Gaussian brush into the selection volume [S1]. It differs from [Voxel Grid Edit via Raycasting](3190-voxel-grid-edit-via-raycasting.md) by focusing on defining a fuzzy selection envelope via additive/subtractive painting rather than direct voxel modification, and from its parent [BrushController](2362-brushcontroller.md) by providing the specific interaction logic for brush placement and composition [S1].

## How it works
**Inputs and Outputs:**
- Input: User click position on the image plane, current camera parameters, volume data (for transparency check), brush parameters (radius, shape, composition mode).
- Output: Updated selection volume (envelope).

**Data Structures:**
- Selection Volume: A 3D grid (same dimensions as data volume) storing float values representing selection strength.
- Brush: Defined by a center position $(c_x, c_y, c_z)$ and radius $r$. The brush shape is a 3D Gaussian [S1].

**Steps:**
1. **Ray Casting:** Upon user click at screen coordinates $(u, v)$, construct a ray $R(t) = O + tD$ from the camera origin $O$ through the pixel [S1].
2. **Intersection:** Cast the ray into the data volume. Find the first non-transparent voxel intersection point $P_{hit}$ [S1].
3. **Brush Placement:** Set the brush center $C$ to $P_{hit}$ [S1].
4. **Painting:** For each voxel $V$ within the bounding box of the brush:
   - If $V$ is non-transparent in the data volume:
     - Calculate distance $d$ from $V$ to $C$.
     - Compute Gaussian weight $w = e^{-\frac{d^2}{2\sigma^2}}$ where $\sigma$ is related to brush radius $r$ [S1].
     - Update selection volume: $S_{new}(V) = S_{old}(V) \circ w$, where $\circ$ is addition (painting) or subtraction (erasing) [S1].

**Parameters:**
- Composition: Addition (painting) or subtraction (erasing) [S1].
- Brush Shape: 3D Gaussian [S1].

**Edge Cases:**
- No non-transparent voxel found: Brush placement fails or defaults to nearest visible surface [S1].
- Precision: Gaussian falloff ensures smooth selection boundaries [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def paint_selection(click_screen_pos, camera, volume_data, selection_volume, brush_radius, composition_mode):
    # 1. Cast ray
    ray_origin, ray_direction = screen_to_ray(click_screen_pos, camera)
    
    # 2. Find first non-transparent voxel
    hit_point = None
    for t in ray_trace(ray_origin, ray_direction, volume_data):
        voxel = get_voxel_at(ray_origin + t * ray_direction, volume_data)
        if voxel is not None and not is_transparent(voxel):
            hit_point = ray_origin + t * ray_direction
            break
            
    if hit_point is None:
        return selection_volume # No hit
        
    # 3. Set brush center
    brush_center = hit_point
    
    # 4. Paint
    bbox_min, bbox_max = get_bounding_box(brush_center, brush_radius)
    
    for voxel_pos in iterate_voxels_in_bbox(bbox_min, bbox_max, selection_volume):
        if is_non_transparent(voxel_pos, volume_data):
            dist = distance(voxel_pos, brush_center)
            # Gaussian weight: sigma is typically radius / sqrt(2) for 1-sigma cutoff
            sigma = brush_radius / 1.414 
            weight = exp(-(dist ** 2) / (2 * sigma ** 2))
            
            current_val = selection_volume[voxel_pos]
            if composition_mode == 'add':
                selection_volume[voxel_pos] = current_val + weight
            elif composition_mode == 'sub':
                selection_volume[voxel_pos] = max(0, current_val - weight)
                
    return selection_volume
```

## Subtopics

### Brush Tip Positioning and Navigation
A virtual painting plane is defined at the current brush tip depth. This plane always stays parallel to the screen plane. New brush strokes are placed on this plane [S2, unofficial]. To enter painting mode, the user holds the Alt key. While holding Alt, moving the mouse moves the brush tip in screen space (2D) or screen space projected to a depth (3D) [S2, unofficial]. To exit painting mode and return to camera navigation, the user releases the Alt key [S2, unofficial]. In 3D mode, the scroll mouse wheel adjusts the depth of the brush tip while holding Alt [S2, unofficial]. The 3D position of the brush tip is determined by projecting the mouse cursor's screen position onto the painting plane [S2, unofficial]. Default Depth: When a new dataset is loaded, the brush tip depth is set to make it visible (floating in front of the object), calculated using the dimensions of the volumetric object’s axis-aligned Bounding Box [S2, unofficial].

## Operating characteristics
Time complexity: $O(N)$ per stroke where $N$ is the number of voxels in the brush bounding box [S1]. Memory complexity: $O(V)$ for the selection volume where $V$ is the volume size [S1]. Strengths: Intuitive, capable of achieving good results in a short time [S1]. Limitations: Requires manual painting; lacks semi-automatic mechanisms like region growing or watershed segmentation [S1].

## Background
Origin: VolumeShop system [S1]. History: Developed for direct volume illustration [S1]. Usage: Used in interactive volume illustration systems for fuzzy selection [S1].

## Sources
[S1] VolumeShop: An Interactive System for Direct Volume Illustration, https://doi.org/10.1109/vis.2005.135, fetched 2026-10-26
[S2] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
