# Volume Painting for Selection

Also called: Volumetric Brush Selection

## Overview
Volume Painting for Selection is a 3D volumetric painting approach for selection definition where a user clicks on the image plane, casting a ray into the volume to find the first non-transparent voxel, then drawing a 3D Gaussian brush into the selection volume [S1]. It differs from [Voxel Grid Edit via Raycasting](3190-voxel-grid-edit-via-raycasting.md) by focusing on defining a fuzzy selection envelope via additive/subtractive painting rather than direct voxel modification, and from its parent [BrushController](2362-brushcontroller.md) by providing the specific interaction logic for brush placement and composition [S1].

## How it works
**Inputs and Outputs:**
- Input: User click position on the image plane, current camera parameters, volume data (for transparency check), brush parameters (radius, shape, composition mode) [S1].
- Output: Updated selection volume (envelope) [S1].

**Data Structures:**
- Selection Volume: A 3D grid matching the data volume dimensions, storing float values representing selection strength [G].
- Brush: Defined by a center position $(c_x, c_y, c_z)$ and radius $r$. The brush shape is a 3D Gaussian [S1].

**Steps:**
1. **Ray Casting:** Upon user click at screen coordinates $(u, v)$, construct a ray $R(t) = O + tD$ from the camera origin $O$ through the pixel [S1].
2. **Intersection:** Cast the ray into the data volume. Find the first non-transparent voxel intersection point $P_{hit}$ [S1].
3. **Brush Placement:** Set the brush center $C$ to $P_{hit}$ [S1].
4. **Painting:** For each voxel $V$ within the bounding box of the brush:
   - If $V$ is non-transparent in the data volume:
     - Calculate distance $d$ from $V$ to $C$.
     - Compute a standard 3D Gaussian weight based on $d$ and brush radius [G].
     - Update selection volume: $S_{new}(V) = S_{old}(V) \circ w$, where $\circ$ is addition (painting) or subtraction (erasing) [S1].

**Parameters:**
- Composition: Addition (painting) or subtraction (erasing) [S1].
- Brush Shape: 3D Gaussian [S1].

**Edge Cases and Pitfalls:**
- No non-transparent voxel found: The ray misses the volume or passes entirely through transparent regions; painting is skipped [G].
- Precision: Floating-point accumulation during ray marching may cause jitter; use fixed step sizes aligned with voxel grid or DDA traversal [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def is_transparent(voxel_val):
    return voxel_val == 0.0  # [G] standard alpha=0 check

def paint_selection(click_screen_pos, camera, volume_data, selection_volume, brush_radius, composition_mode):
    # 1. Construct ray from camera through screen pixel
    ray_origin = camera.origin
    ray_direction = camera.unproject(click_screen_pos)
    ray_len = math.sqrt(sum(c**2 for c in ray_direction))
    ray_direction = tuple(d / ray_len for d in ray_direction)
    
    # 2. Trace ray to find first non-transparent voxel
    hit_point = None
    step_size = 1.0
    t = 0.0
    while t < volume_data.max_depth:
        voxel_pos = tuple(o + t * d for o, d in zip(ray_origin, ray_direction))
        if not volume_data.contains(voxel_pos):
            break
        voxel_val = volume_data.get(voxel_pos)
        if not is_transparent(voxel_val):
            hit_point = voxel_pos
            break
        t += step_size
        
    if hit_point is None:
        return selection_volume
        
    # 3. Set brush center to hit point
    brush_center = hit_point
    
    # 4. Paint within bounding box
    bbox_min = tuple(c - brush_radius for c in brush_center)
    bbox_max = tuple(c + brush_radius for c in brush_center)
    
    for x in range(math.floor(bbox_min[0]), math.ceil(bbox_max[0]) + 1):
        for y in range(math.floor(bbox_min[1]), math.ceil(bbox_max[1]) + 1):
            for z in range(math.floor(bbox_min[2]), math.ceil(bbox_max[2]) + 1):
                voxel_pos = (x, y, z)
                if not volume_data.contains(voxel_pos):
                    continue
                voxel_val = volume_data.get(voxel_pos)
                if is_transparent(voxel_val):
                    continue
                    
                dist = math.sqrt(sum((v - b)**2 for v, b in zip(voxel_pos, brush_center)))
                sigma = brush_radius / math.sqrt(2.0)
                weight = math.exp(-(dist ** 2) / (2 * sigma ** 2))
                
                current_val = selection_volume.get(voxel_pos, 0.0)
                if composition_mode == 'add':
                    selection_volume.set(voxel_pos, current_val + weight)
                elif composition_mode == 'sub':
                    selection_volume.set(voxel_pos, max(0.0, current_val - weight))
                    
    return selection_volume
```

## Subtopics

### Brush Tip Positioning and Navigation
A virtual painting plane is defined at the current brush tip depth [S2, unofficial]. This plane always stays parallel to the screen plane [S2, unofficial]. New brush strokes are placed on this plane [S2, unofficial]. To enter painting mode, the user holds the Alt key [S2, unofficial]. While holding Alt, moving the mouse moves the brush tip in screen space (2D) or screen space projected to a depth (3D) [S2, unofficial]. To exit painting mode and return to camera navigation, the user releases the Alt key [S2, unofficial]. In 3D mode, the scroll mouse wheel adjusts the depth of the brush tip while holding Alt [S2, unofficial]. The 3D position of the brush tip is determined by projecting the mouse cursor's screen position onto the painting plane [S2, unofficial]. Default Depth: When a new dataset is loaded, the brush tip depth is set to make it visible (floating in front of the object), calculated using the dimensions of the volumetric object’s axis-aligned Bounding Box [S2, unofficial].

## Operating characteristics
Strengths: Intuitive, capable of achieving good results in a short time [S1]. Limitations: Requires manual painting; lacks semi-automatic mechanisms like region growing or watershed segmentation [S1]. Time and memory complexity: Not given in the sources.

## Background
Origin: VolumeShop system [S1]. History: Developed for direct volume illustration [S1]. Usage: Used in interactive volume illustration systems for fuzzy selection [S1].

## Sources
[S1] VolumeShop: An Interactive System for Direct Volume Illustration, https://doi.org/10.1109/vis.2005.135, fetched 2026-10-26
[S2] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
